# PostFilter 扩展点的执行 #

在前文中分析了如何在 `sched.Algorithm.Schedule()` 中实现 PreFilter、Filter、PreScore、Score 和 NormalizeScore 等扩展点。`sched.Algorithm.Schedule()` 执行结束后，如果执行失败，会继续执行 PostFilter 扩展点，此扩展点主要实现了抢占的功能。

``` go
	scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, fwk, state, pod)
	if err != nil {
		nominatedNode := ""
		if fitError, ok := err.(*core.FitError); ok {
			if !fwk.HasPostFilterPlugins() {
				klog.V(3).Infof("No PostFilter plugins are registered, so no preemption will be performed.")
			} else {
				// Run PostFilter plugins to try to make the pod schedulable in a future scheduling cycle.
				result, status := fwk.RunPostFilterPlugins(ctx, state, pod, fitError.FilteredNodesStatuses)
                ...
				if status.IsSuccess() && result != nil {
					nominatedNode = result.NominatedNodeName
				}
			}
            ...
		} else if err == core.ErrNoNodesAvailable {
			// No nodes available is counted as unschedulable rather than an error.
			metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
		} else {
			klog.ErrorS(err, "Error selecting node for pod", "pod", klog.KObj(pod))
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
		}
		sched.recordSchedulingFailure(fwk, podInfo, err, v1.PodReasonUnschedulable, nominatedNode)
		return
	}
```

主要通过调用 `RunPostFilterPlugins()` 来执行所有已注册插件的 PostFilter 扩展点，代码如下：

``` go
func (f *frameworkImpl) RunPostFilterPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, filteredNodeStatusMap framework.NodeToStatusMap) (_ *framework.PostFilterResult, status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(postFilter, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()

	statuses := make(framework.PluginToStatus)
	for _, pl := range f.postFilterPlugins {
		r, s := f.runPostFilterPlugin(ctx, pl, state, pod, filteredNodeStatusMap)
		if s.IsSuccess() {
			return r, s
		} else if !s.IsUnschedulable() {
			// Any status other than Success or Unschedulable is Error.
			return nil, framework.NewStatus(framework.Error, s.Message())
		}
		statuses[pl.Name()] = s
	}

	return nil, statuses.Merge()
}

func (f *frameworkImpl) runPostFilterPlugin(ctx context.Context, pl framework.PostFilterPlugin, state *framework.CycleState, pod *v1.Pod, filteredNodeStatusMap framework.NodeToStatusMap) (*framework.PostFilterResult, *framework.Status) {
	if !state.ShouldRecordPluginMetrics() {
		return pl.PostFilter(ctx, state, pod, filteredNodeStatusMap)
	}
	startTime := time.Now()
	r, s := pl.PostFilter(ctx, state, pod, filteredNodeStatusMap)
	f.metricsRecorder.observePluginDurationAsync(postFilter, pl.Name(), s, metrics.SinceInSeconds(startTime))
	return r, s
}
```

这里的调用比较简单，就不会代码进行分析。

目前只有一个插件实现了 PostFilter 扩展点，即默认抢占插件，抢占功能的具体逻辑都抽象到了抢占插件的代码中。由于抢占功能的实现和调度框架关系比较密切，因此在调度框架本身的代码中也有一些涉及到抢占功能的组件，这些都会在[抢占插件](../scheduler-plugins/default-preemption/overview.md)中详细分析。
