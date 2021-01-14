# PostBind 扩展点的执行 #

PostBind 是绑定过程中的最后一步，也是整个调度流程中最后一步。

```
func (sched *Scheduler) scheduleOne(ctx context.Context) {
    ...

	go func() {
        ...

		// Run "prebind" plugins.
        ...

		err := sched.bind(bindingCycleCtx, fwk, assumedPod, scheduleResult.SuggestedHost, state)
		if err != nil {
            ...
		} else {
            ...

			// Run "postbind" plugins.
			fwk.RunPostBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		}
	}()
}
```

它也和 WaitOnPermit、PreBind 和 Bind 都在同一个 goroutine 执行。

``` go
func (f *frameworkImpl) RunPostBindPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(postBind, framework.Success.String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	for _, pl := range f.postBindPlugins {
		f.runPostBindPlugin(ctx, pl, state, pod, nodeName)
	}
}


func (f *frameworkImpl) runPostBindPlugin(ctx context.Context, pl framework.PostBindPlugin, state *framework.CycleState, pod *v1.Pod, nodeName string) {
	if !state.ShouldRecordPluginMetrics() {
		pl.PostBind(ctx, state, pod, nodeName)
		return
	}
	startTime := time.Now()
	pl.PostBind(ctx, state, pod, nodeName)
	f.metricsRecorder.observePluginDurationAsync(postBind, pl.Name(), nil, metrics.SinceInSeconds(startTime))
}
```

会遍历所有实现了 PostBind 扩展点的插件，调用它们的 `PostBind()` 函数。

目前，暂时没有实现 PostBind 扩展点的内置调度器插件。
