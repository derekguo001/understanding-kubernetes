# Bind 扩展点的执行 #

Bind 扩展点的功能是将 Pod 与最终选中的节点进行绑定，这是 Kubernetes Scheduler 的主要目标。

``` go
	// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
	go func() {
        ...

		// Run "prebind" plugins.
        ...

		err := sched.bind(bindingCycleCtx, fwk, assumedPod, scheduleResult.SuggestedHost, state)
		if err != nil {
            ...
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if err := sched.SchedulerCache.ForgetPod(assumedPod); err != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", err)
			}
            ...
		} else {
            ...

			// Run "postbind" plugins.
			fwk.RunPostBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		}
	}()
```

代码位于 `sched.Algorithm.Schedule()` 中，它 WaitOnPermit、PreBind 和 PostBind 都在同一个 goroutine 执行。调用 `sched.bind()` 对 Pod 与节点进行绑定，如果绑定失败，则需要执行 Unreverse 扩展点将预先消费的资源释放掉(例如 PVC 和 PV)，并将 Pod 从调度队列中删除，然后返回错误；如果执行成功，则会接着执行 PostBind 扩展点，这部分可参见下一节内容。

现在分析 `sched.bind()`。

``` go
func (sched *Scheduler) bind(ctx context.Context, fwk framework.Framework, assumed *v1.Pod, targetNode string, state *framework.CycleState) (err error) {
    ...
	bound, err := sched.extendersBinding(assumed, targetNode)
	if bound {
		return err
	}
	bindStatus := fwk.RunBindPlugins(ctx, state, assumed, targetNode)
	if bindStatus.IsSuccess() {
		return nil
	}
	if bindStatus.Code() == framework.Error {
		return bindStatus.AsError()
	}
	return fmt.Errorf("bind status: %s, %v", bindStatus.Code().String(), bindStatus.Message())
}
```

其中，`extendersBinding()` 是调用旧的调度器扩展程序进行绑定，这里不再展开。接着使用 `RunBindPlugins()` 调用 Bind 扩展点。

``` go
func (f *frameworkImpl) RunBindPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (status *framework.Status) {
    ...

	if len(f.bindPlugins) == 0 {
		return framework.NewStatus(framework.Skip, "")
	}
	for _, bp := range f.bindPlugins {
		status = f.runBindPlugin(ctx, bp, state, pod, nodeName)
		if status != nil && status.Code() == framework.Skip {
			continue
		}
		if !status.IsSuccess() {
			err := status.AsError()
			klog.ErrorS(err, "Failed running Bind plugin", "plugin", bp.Name(), "pod", klog.KObj(pod))
			return framework.AsStatus(fmt.Errorf("running Bind plugin %q: %w", bp.Name(), err))
		}
		return status
	}
	return status
}


func (f *frameworkImpl) runBindPlugin(ctx context.Context, bp framework.BindPlugin, state *framework.CycleState, pod *v1.Pod, nodeName string) *framework.Status {
	if !state.ShouldRecordPluginMetrics() {
		return bp.Bind(ctx, state, pod, nodeName)
	}
	startTime := time.Now()
	status := bp.Bind(ctx, state, pod, nodeName)
	f.metricsRecorder.observePluginDurationAsync(bind, bp.Name(), status, metrics.SinceInSeconds(startTime))
	return status
}
```

如果没有实现 Bind 扩展点的调度插件，那么直接返回。否则会使用 `runBindPlugin()` 调用每个插件的 `Bind()` 实现绑定。如果某个调度插件的 `Bind()` 不处理某种类型的 Pod，则可以返回 `Skip` 类型的状态码。

目前，只有一个内置插件实现了 Bind 扩展点，即  [DefaultBinder](../scheduler-plugins/default-binder.md)。
