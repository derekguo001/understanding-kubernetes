# PreScore 扩展点的执行 #

PreScore 扩展点和 PreFilter 扩展点作用类似，一般也是用于数据的预处理，处理后将其存入调度框架的缓存中，然后供后续的 Score 扩展点的插件使用。

在 `sched.Algorithm.Schedule()` 中执行完 PreFilter 和 Filter 扩展点后会接着执行 PreScore 扩展点。

``` go
	// Run "prescore" plugins.
	prescoreStatus := prof.RunPreScorePlugins(ctx, state, pod, filteredNodes)
	if !prescoreStatus.IsSuccess() {
		return result, prescoreStatus.AsError()
	}
	trace.Step("Running prescore plugins done")
```

通过 Framework 的 `RunPreScorePlugins()` 执行。

``` go
func (f *framework) RunPreScorePlugins(
	ctx context.Context,
	state *CycleState,
	pod *v1.Pod,
	nodes []*v1.Node,
) (status *Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(preScore, status.Code().String()).Observe(metrics.SinceInSeconds(startTime))
	}()
	for _, pl := range f.preScorePlugins {
		status = f.runPreScorePlugin(ctx, pl, state, pod, nodes)
		if !status.IsSuccess() {
			msg := fmt.Sprintf("error while running %q prescore plugin for pod %q: %v", pl.Name(), pod.Name, status.Message())
			klog.Error(msg)
			return NewStatus(Error, msg)
		}
	}

	return nil
}
```

会遍历 Framework 的 `preScorePlugins []PreScorePlugin` 字段，最终会执行到每个调度插件的 `PreScore()` 函数。

``` go
func (f *framework) runPreScorePlugin(ctx context.Context, pl PreScorePlugin, state *CycleState, pod *v1.Pod, nodes []*v1.Node) *Status {
	if !state.ShouldRecordPluginMetrics() {
		return pl.PreScore(ctx, state, pod, nodes)
	}
	startTime := time.Now()
	status := pl.PreScore(ctx, state, pod, nodes)
	f.metricsRecorder.observePluginDurationAsync(preScore, pl.Name(), status, metrics.SinceInSeconds(startTime))
	return status
}
```

以 [SelectorSpread](../scheduler-plugins/selector-spread.md) 插件为例，它的 PreScore 扩展点所做的操作就是根据当前 Pod 的 Label 找出包含此 Pod 的 "上级" 资源的 Label，并将结果存入到调度缓存中，在此插件的 Score 扩展点会取回这些信息然后进一步进行处理。

``` go
func (pl *SelectorSpread) PreScore(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod, nodes []*v1.Node) *framework.Status {
	if skipSelectorSpread(pod) {
		return nil
	}
	var selector labels.Selector
	selector = helper.DefaultSelector(
		pod,
		pl.services,
		pl.replicationControllers,
		pl.replicaSets,
		pl.statefulSets,
	)
	state := &preScoreState{
		selector: selector,
	}
	cycleState.Write(preScoreStateKey, state)
	return nil
}
```

下一节会对 Score 扩展点的执行进行分析。
