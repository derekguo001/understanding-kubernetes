# prefilter #

`sched.Algorithm.Schedule()` 中首先会做一些基本的检测，然后执行所有实现了 PreFilter 扩展点的插件。

``` go
func (g *genericScheduler) Schedule(ctx context.Context, prof *profile.Profile, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	trace := utiltrace.New("Scheduling", utiltrace.Field{Key: "namespace", Value: pod.Namespace}, utiltrace.Field{Key: "name", Value: pod.Name})
	defer trace.LogIfLong(100 * time.Millisecond)

	if err := podPassesBasicChecks(pod, g.pvcLister); err != nil {
		return result, err
	}
	trace.Step("Basic checks done")

	if err := g.snapshot(); err != nil {
		return result, err
	}
	trace.Step("Snapshotting scheduler cache and node infos done")

	if g.nodeInfoSnapshot.NumNodes() == 0 {
		return result, ErrNoNodesAvailable
	}

	// Run "prefilter" plugins.
	preFilterStatus := prof.RunPreFilterPlugins(ctx, state, pod)
	if !preFilterStatus.IsSuccess() {
		return result, preFilterStatus.AsError()
	}
	trace.Step("Running prefilter plugins done")

    ...
```

`podPassesBasicChecks()` 会检测 Pod 用到的 PVC 是否存在，以及是否处于删除状态，如果没有问题则继续执行下面的流程。

接着使用 `g.snapshot()` 更新缓存。如果缓存中节点数量为 0，则直接返回。

然后通过执行 `RunPreFilterPlugins()` 来调用所有实现了 PreFilter 扩展点的插件。

``` go
func (f *framework) RunPreFilterPlugins(ctx context.Context, state *CycleState, pod *v1.Pod) (status *Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(preFilter, status.Code().String()).Observe(metrics.SinceInSeconds(startTime))
	}()
	for _, pl := range f.preFilterPlugins {
		status = f.runPreFilterPlugin(ctx, pl, state, pod)
		if !status.IsSuccess() {
			if status.IsUnschedulable() {
				msg := fmt.Sprintf("rejected by %q at prefilter: %v", pl.Name(), status.Message())
				klog.V(4).Infof(msg)
				return NewStatus(status.Code(), msg)
			}
			msg := fmt.Sprintf("error while running %q prefilter plugin for pod %q: %v", pl.Name(), pod.Name, status.Message())
			klog.Error(msg)
			return NewStatus(Error, msg)
		}
	}

	return nil
}

func (f *framework) runPreFilterPlugin(ctx context.Context, pl PreFilterPlugin, state *CycleState, pod *v1.Pod) *Status {
	if !state.ShouldRecordPluginMetrics() {
		return pl.PreFilter(ctx, state, pod)
	}
	startTime := time.Now()
	status := pl.PreFilter(ctx, state, pod)
	f.metricsRecorder.observePluginDurationAsync(preFilter, pl.Name(), status, metrics.SinceInSeconds(startTime))
	return status
}
```

这里的逻辑比较简单，就是调用每个插件的 `PreFilter()` 接口。如果某一个插件未执行成功，则认为整个流程失败，立即返回，当前 Pod 不可以被调度。

PreFilter 扩展点一般用于数据的预处理，处理后将其存入调度框架的缓存中，然后供后续的 Filter 扩展点的插件使用。例如在 [NodePorts](../scheduler-plugins/node-ports.md) 插件中，就是获取当前 Pod 所使用的端口信息，将其存入调度框架的缓存中：

``` go
func (pl *NodePorts) PreFilter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod) *framework.Status {
	s := getContainerPorts(pod)
	cycleState.Write(preFilterStateKey, preFilterState(s))
	return nil
}
```

在此插件的 Filter 扩展点会取出这些数据进行处理。

``` go
func (pl *NodePorts) Filter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
	wantPorts, err := getPreFilterState(cycleState)
	if err != nil {
		return framework.AsStatus(err)
	}

	fits := fitsPorts(wantPorts, nodeInfo)
	if !fits {
		return framework.NewStatus(framework.Unschedulable, ErrReason)
	}

	return nil
}
```

下一节分析 Filter 扩展点的调用。
