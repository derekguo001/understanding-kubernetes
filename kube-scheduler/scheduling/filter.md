# Filter 扩展点的执行 #

在 `sched.Algorithm.Schedule()` 中运行完了 PreFilter 扩展点插件之后，会接着执行 Filter 扩展点的插件。

``` go
	startPredicateEvalTime := time.Now()
	filteredNodes, filteredNodesStatuses, err := g.findNodesThatFitPod(ctx, prof, state, pod)
	if err != nil {
		return result, err
	}
	trace.Step("Computing predicates done")
```

整个流程都在 `findNodesThatFitPod()` 中实现。

``` go
func (g *genericScheduler) findNodesThatFitPod(ctx context.Context, prof *profile.Profile, state *framework.CycleState, pod *v1.Pod) ([]*v1.Node, framework.NodeToStatusMap, error) {
	filteredNodesStatuses := make(framework.NodeToStatusMap)
	filtered, err := g.findNodesThatPassFilters(ctx, prof, state, pod, filteredNodesStatuses)
	if err != nil {
		return nil, nil, err
	}

	filtered, err = g.findNodesThatPassExtenders(pod, filtered, filteredNodesStatuses)
	if err != nil {
		return nil, nil, err
	}
	return filtered, filteredNodesStatuses, nil
}
```

可以看出分成了两个步骤：

- `findNodesThatPassFilters()` 执行标准的 Filter 扩展点的插件
- `findNodesThatPassExtenders()` 执行调度器扩展程序的 `Filter()` 函数，这是一种旧的扩展插件，我们不会对这种方式进行展开分析。

接下来对 `findNodesThatPassFilters()` 进行分析。

## 大规模集群场景下的优化 ##

``` go
func (g *genericScheduler) findNodesThatPassFilters(ctx context.Context, prof *profile.Profile, state *framework.CycleState, pod *v1.Pod, statuses framework.NodeToStatusMap) ([]*v1.Node, error) {
	allNodes, err := g.nodeInfoSnapshot.NodeInfos().List()
	if err != nil {
		return nil, err
	}

	numNodesToFind := g.numFeasibleNodesToFind(int32(len(allNodes)))
```

函数开始执行后先使用 `numFeasibleNodesToFind()` 获取一个数值。当节点非常多的时候，对所有节点全部进行 Filter 扩展点的验证会造成效率非常低下。因此，这里的逻辑是达到一定数量后就立即返回。这个函数实现的就是如何确定此值。

这个值是可以通过 [Kubernetes Scheduler 配置文件](https://kubernetes.io/docs/reference/scheduling/config/) 进行配置的，默认为 0。此值会存储于 `genericScheduler.percentageOfNodesToScore`。

``` go
const (
	minFeasibleNodesToFind = 100
	minFeasibleNodesPercentageToFind = 5
)

func (g *genericScheduler) numFeasibleNodesToFind(numAllNodes int32) (numNodes int32) {
	if numAllNodes < minFeasibleNodesToFind || g.percentageOfNodesToScore >= 100 {
		return numAllNodes
	}
```

首先，当节点数小于 100 或者用户配置的这个百分比的值超过 100，则对所有的节点执行 Filter 扩展点。因为小于 100 时，没有必要进行优化；如果用户配置的百分比超过 100，表示用户明确表示对所有节点执行 Filter 扩展点。

``` go
	adaptivePercentage := g.percentageOfNodesToScore
	if adaptivePercentage <= 0 {
		basePercentageOfNodesToScore := int32(50)
		adaptivePercentage = basePercentageOfNodesToScore - numAllNodes/125
		if adaptivePercentage < minFeasibleNodesPercentageToFind {
			adaptivePercentage = minFeasibleNodesPercentageToFind
		}
	}

	numNodes = numAllNodes * adaptivePercentage / 100
	if numNodes < minFeasibleNodesToFind {
		return minFeasibleNodesToFind
	}

	return numNodes
}
```

如果节点数超过 100 且用户配置的百分比小于等于 0，则需要计算一个百分比。公式为：

```
百分比 = 50 - (总节点数)/125
```

如果计算出的百分比小于 5，则将其设为 5。按照此算法，在总节点数大于等于 6250 的情况下，百分比就固定为 5。

最后使用总节点数乘以这个百分比，然后将结果返回。

## Filter 扩展点的执行 ##

接着执行 Filter 扩展点。

``` go
	filtered := make([]*v1.Node, numNodesToFind)

	if !prof.HasFilterPlugins() {
		for i := range filtered {
			filtered[i] = allNodes[i].Node()
		}
		g.nextStartNodeIndex = (g.nextStartNodeIndex + len(filtered)) % len(allNodes)
		return filtered, nil
	}
```

如果当前调度框架没有启用任何 Filter 扩展点的插件，则从所有节点中随机找到预定数量的节点后直接返回。

``` go
	checkNode := func(i int) {
        ...
	}

    ...
	workqueue.ParallelizeUntil(ctx, 16, len(allNodes), checkNode)
	processedNodes := int(filteredLen) + len(statuses)
	g.nextStartNodeIndex = (g.nextStartNodeIndex + processedNodes) % len(allNodes)

	filtered = filtered[:filteredLen]
	if err := errCh.ReceiveError(); err != nil {
		statusCode = framework.Error
		return nil, err
	}
	return filtered, nil
}
```

接着会定义一个函数 `checkNode`，它封装了一个执行 Filter 扩展点的操作。然后并行地对所有节点执行这个函数。下面是这个函数的内容：

``` go
	checkNode := func(i int) {
		nodeInfo := allNodes[(g.nextStartNodeIndex+i)%len(allNodes)]
		fits, status, err := g.podPassesFiltersOnNode(ctx, prof, state, pod, nodeInfo)
		if err != nil {
			errCh.SendErrorWithCancel(err, cancel)
			return
		}
		if fits {
			length := atomic.AddInt32(&filteredLen, 1)
			if length > numNodesToFind {
				cancel()
				atomic.AddInt32(&filteredLen, -1)
			} else {
				filtered[length-1] = nodeInfo.Node()
			}
		} else {
			statusesLock.Lock()
			if !status.IsSuccess() {
				statuses[nodeInfo.Node().Name] = status
			}
			statusesLock.Unlock()
		}
	}
```

如果在这个函数内部执行时，已经找到了预定数量的节点，那么会终止整个并行执行过程，然后将结果返回。

对某一个节点执行 Filter 扩展点的调用封装在 `podPassesFiltersOnNode()` 中。


``` go
func (g *genericScheduler) podPassesFiltersOnNode(
	ctx context.Context,
	prof *profile.Profile,
	state *framework.CycleState,
	pod *v1.Pod,
	info *schedulernodeinfo.NodeInfo,
) (bool, *framework.Status, error) {
    ...
	for i := 0; i < 2; i++ {
		stateToUse := state
		nodeInfoToUse := info
		if i == 0 {
			var err error
			podsAdded, stateToUse, nodeInfoToUse, err = g.addNominatedPods(ctx, prof, pod, state, info)
			if err != nil {
				return false, nil, err
			}
		} else if !podsAdded || !status.IsSuccess() {
			break
		}

		statusMap := prof.RunFilterPlugins(ctx, stateToUse, pod, nodeInfoToUse)
		status = statusMap.Merge()
		if !status.IsSuccess() && !status.IsUnschedulable() {
			return false, status, status.AsError()
		}
	}

	return status.IsSuccess(), status, nil
}
```

具体通过 Framework 的 `RunFilterPlugins()` 执行：

``` go
func (f *framework) RunFilterPlugins(
	ctx context.Context,
	state *CycleState,
	pod *v1.Pod,
	nodeInfo *schedulernodeinfo.NodeInfo,
) PluginToStatus {
	var firstFailedStatus *Status
	statuses := make(PluginToStatus)
	for _, pl := range f.filterPlugins {
		pluginStatus := f.runFilterPlugin(ctx, pl, state, pod, nodeInfo)
		if len(statuses) == 0 {
			firstFailedStatus = pluginStatus
		}
		if !pluginStatus.IsSuccess() {
			if !pluginStatus.IsUnschedulable() {
				// Filter plugins are not supposed to return any status other than
				// Success or Unschedulable.
				firstFailedStatus = NewStatus(Error, fmt.Sprintf("running %q filter plugin for pod %q: %v", pl.Name(), pod.Name, pluginStatus.Message()))
				return map[string]*Status{pl.Name(): firstFailedStatus}
			}
			statuses[pl.Name()] = pluginStatus
			if !f.runAllFilters {
				// Exit early if we don't need to run all filters.
				return statuses
			}
		}
	}

	return statuses
}

func (f *framework) runFilterPlugin(ctx context.Context, pl FilterPlugin, state *CycleState, pod *v1.Pod, nodeInfo *schedulernodeinfo.NodeInfo) *Status {
	if !state.ShouldRecordPluginMetrics() {
		return pl.Filter(ctx, state, pod, nodeInfo)
	}
	startTime := time.Now()
	status := pl.Filter(ctx, state, pod, nodeInfo)
	f.metricsRecorder.observePluginDurationAsync(Filter, pl.Name(), status, metrics.SinceInSeconds(startTime))
	return status
}
```

会遍历 Framework 的 `filterPlugins []FilterPlugin` 字段，最终会执行到每个调度插件的 `Filter()` 函数。

在使用 `findNodesThatFitPod()` 执行完插件的 PreFilter 和 Filter 扩展点后，会返回到 `sched.Algorithm.Schedule()` 中。

``` go
	if len(filteredNodes) == 0 {
		return result, &FitError{
			Pod:                   pod,
			NumAllNodes:           g.nodeInfoSnapshot.NumNodes(),
			FilteredNodesStatuses: filteredNodesStatuses,
		}
	}
```

如果没有发现可调度的节点，就没有必要执行 PreScore 和 Score 等扩展点，会直接返回。

在下一节会对 PreScore 扩展点进行分析。
