# WaitOnPermit 扩展点的执行 #

在调度框架中，执行完了 Permit 扩展点之后，就执行完了当前 Pod 的整个调度周期，接下来会进入绑定周期。绑定操作是在一个单独的 goroutine 中并行执行的。绑定周期包括 WaitOnPermit、PreBind、Bind 和 PostBind 共计四个扩展点。

其中的 WaitOnPermit 扩展点是首先得到执行的，它的作用是配合调度周期中的 [Permit 扩展点](permit.md) 一起完成 Pod 延时绑定的功能。其实严格意义上来讲，WaitOnPermit 并不是一个标准的扩展点，用户的自定义调度插件并不能在 WaitOnPermit 扩展点实现自己的功能。因此在调度过程的图里将其标示为虚线框。

``` go
	go func() {
        ...

		waitOnPermitStatus := fwk.WaitOnPermit(bindingCycleCtx, assumedPod)
		if !waitOnPermitStatus.IsSuccess() {
			var reason string
			if waitOnPermitStatus.IsUnschedulable() {
				metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
				reason = v1.PodReasonUnschedulable
			} else {
				metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
				reason = SchedulerError
			}
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, waitOnPermitStatus.AsError(), reason, "")
			return
		}
        ...
    }
```

核心是 `WaitOnPermit()` 函数。

``` go
func (f *frameworkImpl) WaitOnPermit(ctx context.Context, pod *v1.Pod) (status *framework.Status) {
	waitingPod := f.waitingPods.get(pod.UID)
	if waitingPod == nil {
		return nil
	}
```

首先，判断当前 Pod 是否在延时绑定的 Map 中，如果不在，则表示当前 Pod 在 Permit 扩展点没有被当做延时绑定的 Pod，然后直接返回执行后续的绑定操作。

```
	defer f.waitingPods.remove(pod.UID)
	klog.V(4).Infof("pod %q waiting on permit", pod.Name)

	startTime := time.Now()
	s := <-waitingPod.s
	metrics.PermitWaitDuration.WithLabelValues(s.Code().String()).Observe(metrics.SinceInSeconds(startTime))
	if !s.IsSuccess() {
		if s.IsUnschedulable() {
			msg := fmt.Sprintf("pod %q rejected while waiting on permit: %v", pod.Name, s.Message())
			klog.V(4).Infof(msg)
			return framework.NewStatus(s.Code(), msg)
		}
		err := s.AsError()
		klog.ErrorS(err, "Failed waiting on permit for pod", "pod", klog.KObj(pod))
		return framework.AsStatus(fmt.Errorf("waiting on permit for pod: %w", err))
	}
	return nil
}
```

如果 Pod 是在延时绑定的 Map 里，则会通过 channel 等待其最终执行状态，如果主动允许 Pod 进行调度，即调用 [延时绑定](waiting-pod.md) 中的 `Allow()` 函数，则这里会返回 `framework.NewStatus(framework.Success, "")`，就会从当前函数返回，然后执行后续的绑定流程。

如果没有主动允许 Pod 进行调度，或者执行了主动禁止 Pod 调度的操作(即调用 [延时绑定](waiting-pod.md) 中的 `Reject()` 函数)、或者等待超时之后，这里会返回 `framework.NewStatus(framework.Unschedulable, msg)`，那么也会在当前函数中返回，然后在上一级函数 `scheduleOne()` 里终止后续的调度流程。
