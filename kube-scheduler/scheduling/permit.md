# Permit 扩展点的执行 #

在调度框架中，真正执行绑定操作之前，有一个 Permit 扩展点，它允许批准、拒绝或者延时绑定当前 Pod。它是调度周期中的最后一个扩展点。

``` go
	runPermitStatus := fwk.RunPermitPlugins(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
	if runPermitStatus.Code() != framework.Wait && !runPermitStatus.IsSuccess() {
        ...
		fwk.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
			klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
		}
		sched.recordSchedulingFailure(fwk, assumedPodInfo, runPermitStatus.AsError(), reason, "")
		return
	}
```

如果在调度插件中的 Permit 扩展点明确拒绝调度当前 Pod 或者执行失败，则会执行 Unreverse 扩展点将预先消费的资源释放掉(例如 PVC 和 PV)，并将 Pod 从调度队列中删除，然后返回错误。

``` go
func (f *frameworkImpl) RunPermitPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (status *framework.Status) {
    ...
	for _, pl := range f.permitPlugins {
		status, timeout := f.runPermitPlugin(ctx, pl, state, pod, nodeName)
		if !status.IsSuccess() {
			if status.IsUnschedulable() {
				msg := fmt.Sprintf("rejected pod %q by permit plugin %q: %v", pod.Name, pl.Name(), status.Message())
				klog.V(4).Infof(msg)
				return framework.NewStatus(status.Code(), msg)
			}
			if status.Code() == framework.Wait {
				// Not allowed to be greater than maxTimeout.
				if timeout > maxTimeout {
					timeout = maxTimeout
				}
				pluginsWaitTime[pl.Name()] = timeout
				statusCode = framework.Wait
			} else {
				err := status.AsError()
				klog.ErrorS(err, "Failed running Permit plugin", "plugin", pl.Name(), "pod", klog.KObj(pod))
				return framework.AsStatus(fmt.Errorf("running Permit plugin %q: %w", pl.Name(), err))
			}
		}
	}
	if statusCode == framework.Wait {
		waitingPod := newWaitingPod(pod, pluginsWaitTime)
		f.waitingPods.add(waitingPod)
		msg := fmt.Sprintf("one or more plugins asked to wait and no plugin rejected pod %q", pod.Name)
		klog.V(4).Infof(msg)
		return framework.NewStatus(framework.Wait, msg)
	}
	return nil
}

func (f *frameworkImpl) runPermitPlugin(ctx context.Context, pl framework.PermitPlugin, state *framework.CycleState, pod *v1.Pod, nodeName string) (*framework.Status, time.Duration) {
	if !state.ShouldRecordPluginMetrics() {
		return pl.Permit(ctx, state, pod, nodeName)
	}
	startTime := time.Now()
	status, timeout := pl.Permit(ctx, state, pod, nodeName)
	f.metricsRecorder.observePluginDurationAsync(permit, pl.Name(), status, metrics.SinceInSeconds(startTime))
	return status, timeout
}
```

通过执行 `runPermitPlugin()` 来调用每个插件的 Permit 扩展点。`Permit()` 可能返回的值包括：
- framework.Success。正常执行后续的调度扩展点。
- framework.Wait。延时绑定。
- 其它情况，这种表示 Permit 扩展点明确拒绝调度当前 Pod 或者出错，会终止调度，返回。

下面分析延时绑定的情况。

``` go
  		if status.Code() == framework.Wait {
  			// Not allowed to be greater than maxTimeout.
  			if timeout > maxTimeout {
  				timeout = maxTimeout
  			}
  			pluginsWaitTime[pl.Name()] = timeout
  			statusCode = framework.Wait
      ...
 ```

这种情况下，`Permit()` 还会返回一个延时时间，在 `RunPermitPlugins()` 中执行 `pluginsWaitTime[pl.Name()] = timeout`，构造一个 `pluginsWaitTime` 变量。

``` go
  if statusCode == framework.Wait {
  	waitingPod := newWaitingPod(pod, pluginsWaitTime)
  	f.waitingPods.add(waitingPod)
  	msg := fmt.Sprintf("one or more plugins asked to wait and no plugin rejected pod %q", pod.Name)
  	klog.V(4).Infof(msg)
  	return framework.NewStatus(framework.Wait, msg)
  }
```

接着使用 `pluginsWaitTime` 作为参数调用上一节 [延时绑定](waiting-pod.md) 中的 `newWaitingPod()` 构造一个 waitingPod 对象，然后将其加入 Framework.waitingPods 中。最后返回 `framework.NewStatus(framework.Wait, msg)` 状态。

需要注意的是，这里只是创建了延时绑定的 Pod 对象(创建的同时会开始计时)，最终处理延时绑定结果的逻辑在下一节 [WaitOnPermit 扩展点](wait-on-permit.md) 中。
