# 调度流程分析 #

在 [Kubernetes Scheduler 概述](../overview.md) 中对调度过程的的各个阶段进行了简要的概述，在 [Kubernetes Scheduler 中的组件和启动过程](../../README.md#Kubernetes-Scheduler-中的组件和启动过程) 中分析了主要的组件和初始化过程，在本节我们开始针对调度的整体流程进行代码层面的分析。在后面的章节中会针对每一个扩展点进行详细分析。

![scheduler](https://raw.githubusercontent.com/kubernetes/enhancements/master/keps/sig-scheduling/624-scheduling-framework/scheduling-framework-extensions.png)

调度器运行后，会执行一个名为 `scheduleOne()` 的函数。

``` go
func (sched *Scheduler) Run(ctx context.Context) {
    ...
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)
    ...
}
```

这个函数包含了整个调度过程，其中的每一个步骤都与上图中的各个节点严格对应。

``` go
func (sched *Scheduler) scheduleOne(ctx context.Context) {
	podInfo := sched.NextPod()
    ...
	pod := podInfo.Pod
	prof, err := sched.profileForPod(pod)
    ...
```

首先，从队列中取出一个 Pod，然后根据 Pod 的 `pod.Spec.SchedulerName` 字段找到对应里的 `Profile` 对象，其中包含有当前调度器对应的调度框架对象，详见 [Profile](../component/profile.md)。后续的调度过程中主要依赖这个调度框架对象对 Pod 进行调度。

``` go
	klog.V(3).Infof("Attempting to schedule pod: %v/%v", pod.Namespace, pod.Name)

    ...
	scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, prof, state, pod)
```

然后开始调度过程，先运行 `sched.Algorithm.Schedule()`，尝试为当前 Pod 找出一个合适的 Node。这个过程包含了图中的 [PreFilter](pre-filter.md)、[Filter](filter.md)、[PreScore](pre-score.md)、[Score](score/overview.md) 和 [Normalize Score](score/score-and-normalizescore.md) 等步骤。具体过程可分别参见各自的章节。

``` go
	if err != nil {
        ...
		if fitError, ok := err.(*core.FitError); ok {
			if sched.DisablePreemption {
				klog.V(3).Infof("Pod priority feature is not enabled or preemption is disabled by scheduler configuration." +
					" No preemption is performed.")
			} else {
				preemptionStartTime := time.Now()
				sched.preempt(schedulingCycleCtx, prof, state, pod, fitError)
                ...
			}
            ...
		} else {
			klog.Errorf("error selecting node for pod: %v", err)
			metrics.PodScheduleErrors.Inc()
		}
		sched.recordSchedulingFailure(prof, podInfo.DeepCopy(), err, v1.PodReasonUnschedulable, err.Error())
		return
	}
```

如果找不到合适的 Node，则尝试进行抢占操作。抢占操作对应于调度流程中的 PostFilter 扩展点。

``` go
    ...

	// Run "reserve" plugins.
	if sts := prof.RunReservePlugins(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
		sched.recordSchedulingFailure(prof, assumedPodInfo, sts.AsError(), SchedulerError, sts.Message())
		metrics.PodScheduleErrors.Inc()
		return
	}
```

接下来执行 [Reserve](reserve.md) 扩展点，Reserver 扩展点的作用是做一些相关附属资源的预留，例如 PVC。

``` go
    ...

	// Run "permit" plugins.
	runPermitStatus := prof.RunPermitPlugins(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
	if runPermitStatus.Code() != framework.Wait && !runPermitStatus.IsSuccess() {
		var reason string
		if runPermitStatus.IsUnschedulable() {
			metrics.PodScheduleFailures.Inc()
			reason = v1.PodReasonUnschedulable
		} else {
			metrics.PodScheduleErrors.Inc()
			reason = SchedulerError
		}
		if forgetErr := sched.Cache().ForgetPod(assumedPod); forgetErr != nil {
			klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
		}
		// One of the plugins returned status different than success or wait.
		prof.RunUnreservePlugins(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		sched.recordSchedulingFailure(prof, assumedPodInfo, runPermitStatus.AsError(), reason, runPermitStatus.Message())
		return
	}
```

然后执行 [Permit](permit.md) 扩展点。Permit 扩展点允许在执行真正的调度操作之前对 Pod 绑定操作进行最后的批准、拒绝或者执行延时调度。

以上部分对应于 [Kubernetes Scheduler 概述](../overview.md) 中的 `调度周期`，接下来会在一个单独的 goroutine 中执行 `绑定周期`。

``` go
	// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
	go func() {
		bindingCycleCtx, cancel := context.WithCancel(ctx)
		defer cancel()
		metrics.SchedulerGoroutines.WithLabelValues("binding").Inc()
		defer metrics.SchedulerGoroutines.WithLabelValues("binding").Dec()

		waitOnPermitStatus := prof.WaitOnPermit(bindingCycleCtx, assumedPod)
		if !waitOnPermitStatus.IsSuccess() {
			var reason string
			if waitOnPermitStatus.IsUnschedulable() {
				metrics.PodScheduleFailures.Inc()
				reason = v1.PodReasonUnschedulable
			} else {
				metrics.PodScheduleErrors.Inc()
				reason = SchedulerError
			}
			if forgetErr := sched.Cache().ForgetPod(assumedPod); forgetErr != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
			}
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			prof.RunUnreservePlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			sched.recordSchedulingFailure(prof, assumedPodInfo, waitOnPermitStatus.AsError(), reason, waitOnPermitStatus.Message())
			return
		}
```

绑定操作中的第一步是 [WaitOnPermit 扩展点](wait-on-permit.md)，它主要结合 Permit 扩展点一起完成延时调度的功能。

``` go
		// Run "prebind" plugins.
		preBindStatus := fwk.RunPreBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if !preBindStatus.IsSuccess() {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, preBindStatus.AsError(), SchedulerError, "")
			return
		}
```

接着执行 [PreBind 扩展点](pre-bind.md)，这个扩展点的功能主要是做一些 Pod 与节点绑定之前的准备工作，或者让其它相关资源的绑定，例如 PVC 和 PV。

		// Run "prebind" plugins.
		preBindStatus := prof.RunPreBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if !preBindStatus.IsSuccess() {
			var reason string
			metrics.PodScheduleErrors.Inc()
			reason = SchedulerError
			if forgetErr := sched.Cache().ForgetPod(assumedPod); forgetErr != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
			}
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			prof.RunUnreservePlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			sched.recordSchedulingFailure(prof, assumedPodInfo, preBindStatus.AsError(), reason, preBindStatus.Message())
			return
		}
```


``` go
		err := sched.bind(bindingCycleCtx, fwk, assumedPod, scheduleResult.SuggestedHost, state)
		if err != nil {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if err := sched.SchedulerCache.ForgetPod(assumedPod); err != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", err)
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, fmt.Errorf("binding rejected: %w", err), SchedulerError, "")
		} else {
			// Calculating nodeResourceString can be heavy. Avoid it if klog verbosity is below 2.
			if klog.V(2).Enabled() {
				klog.InfoS("Successfully bound pod to node", "pod", klog.KObj(pod), "node", scheduleResult.SuggestedHost, "evaluatedNodes", scheduleResult.EvaluatedNodes, "feasibleNodes", scheduleResult.FeasibleNodes)
			}
			metrics.PodScheduled(fwk.ProfileName(), metrics.SinceInSeconds(start))
			metrics.PodSchedulingAttempts.Observe(float64(podInfo.Attempts))
			metrics.PodSchedulingDuration.WithLabelValues(getAttemptsLabel(podInfo)).Observe(metrics.SinceInSeconds(podInfo.InitialAttemptTimestamp))

			// Run "postbind" plugins.
			fwk.RunPostBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		}
	}()
}
```

然后执行 [Bind 扩展点](bind.md)，Pod 会在这个扩展点与节点之间真正产生绑定关系。

如果 Bind 扩展点执行成功，则接着会执行 [PostBind 扩展点](post-bind.md)；如果执行失败，则说明绑定错误，会直接返回。

至此，整个 `绑定周期` 结束。Pod 与某个节点完成绑定，Pod 所使用的 PVC 也与对应的 PV 完成绑定。

接下来，会详细分析每一个扩展点的代码实现。
