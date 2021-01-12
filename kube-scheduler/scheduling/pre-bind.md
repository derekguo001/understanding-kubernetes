# PreBind 扩展点的执行 #

在 `sched.Algorithm.Schedule()` 中，成功执行完 WaitOnPermit 后会执行 PreBind 扩展点。

PreBind 扩展点的功能主要是做一些 Pod 与节点绑定之前的准备工作，或者让其它相关资源的绑定。

例如：如果一个 Pod 使用了 PVC，则需要将 PVC 和对应的 PV 进行绑定，这些操作都在 PreBind 扩展点执行。

``` go
		// Run "prebind" plugins.
		preBindStatus := fwk.RunPreBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if !preBindStatus.IsSuccess() {
            ...
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
			}
            ...
			return
		}
```

如果 PreBind 执行失败，则需要执行 Unreverse 扩展点将预先消费的资源释放掉(例如 PVC 和 PV)，并将 Pod 从调度队列中删除，然后返回错误，在这种情况下整个调度过程会终止，后续的 Bind 和 PostBind 扩展点将不会执行。

``` go
func (f *frameworkImpl) RunPreBindPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (status *framework.Status) {
    ...
	for _, pl := range f.preBindPlugins {
		status = f.runPreBindPlugin(ctx, pl, state, pod, nodeName)
		if !status.IsSuccess() {
			err := status.AsError()
			klog.ErrorS(err, "Failed running PreBind plugin", "plugin", pl.Name(), "pod", klog.KObj(pod))
			return framework.AsStatus(fmt.Errorf("running PreBind plugin %q: %w", pl.Name(), err))
		}
	}
	return nil
}

func (f *frameworkImpl) runPreBindPlugin(ctx context.Context, pl framework.PreBindPlugin, state *framework.CycleState, pod *v1.Pod, nodeName string) *framework.Status {
	if !state.ShouldRecordPluginMetrics() {
		return pl.PreBind(ctx, state, pod, nodeName)
	}
	startTime := time.Now()
	status := pl.PreBind(ctx, state, pod, nodeName)
	f.metricsRecorder.observePluginDurationAsync(preBind, pl.Name(), status, metrics.SinceInSeconds(startTime))
	return status
}
```

会遍历所有实现了 PreBind 扩展点的插件，调用其中的 `PreBind()` 函数。

目前，只有一个内置插件实现了 PreBind 扩展点，即 VolumeBinding 插件。

``` go
func (pl *VolumeBinding) PreBind(ctx context.Context, cs *framework.CycleState, pod *v1.Pod, nodeName string) *framework.Status {
	s, err := getStateData(cs)
	if err != nil {
		return framework.AsStatus(err)
	}
	if s.allBound {
		// no need to bind volumes
		return nil
	}
	// we don't need to hold the lock as only one node will be pre-bound for the given pod
	podVolumes, ok := s.podVolumesByNode[nodeName]
	if !ok {
		return framework.AsStatus(fmt.Errorf("no pod volumes found for node %q", nodeName))
	}
	klog.V(5).Infof("Trying to bind volumes for pod \"%v/%v\"", pod.Namespace, pod.Name)
	err = pl.Binder.BindPodVolumes(pod, podVolumes)
	if err != nil {
		klog.V(1).Infof("Failed to bind volumes for pod \"%v/%v\": %v", pod.Namespace, pod.Name, err)
		return framework.AsStatus(err)
	}
	klog.V(5).Infof("Success binding volumes for pod \"%v/%v\"", pod.Namespace, pod.Name)
	return nil
}
```

它的 PreBind 扩展点实现功能就是调用 `pl.Binder.BindPodVolumes(pod, podVolumes)` 将 Pod 使用的 PVC 和 PV 之间实现绑定。
