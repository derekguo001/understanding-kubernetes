# Score 整体流程 #

在 `sched.Algorithm.Schedule()` 中执行完 PreScore 扩展点之后，会进入具体的节点打分流程。

``` go
    ...

	startPriorityEvalTime := time.Now()
	// When only one node after predicate, just use it.
	if len(filteredNodes) == 1 {
		metrics.DeprecatedSchedulingAlgorithmPriorityEvaluationSecondsDuration.Observe(metrics.SinceInSeconds(startPriorityEvalTime))
		return ScheduleResult{
			SuggestedHost:  filteredNodes[0].Name,
			EvaluatedNodes: 1 + len(filteredNodesStatuses),
			FeasibleNodes:  1,
		}, nil
	}

	priorityList, err := g.prioritizeNodes(ctx, prof, state, pod, filteredNodes)
	if err != nil {
		return result, err
	}
    ...

	host, err := g.selectHost(priorityList)
	trace.Step("Prioritizing done")

	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(filteredNodes) + len(filteredNodesStatuses),
		FeasibleNodes:  len(filteredNodes),
	}, err
}
```

首先判断候选节点的数量，如果只有一个，就没有必要进行 Score 扩展点，会直接返回。

否则会通过调用 `prioritizeNodes()` 对节点进行打分，打分结束后使用 `selectHost()` 来选一个最优的节点，最后返回所选的节点和一些其它相关信息。先来看下 `selectHost()` 函数：

``` go
func (g *genericScheduler) selectHost(nodeScoreList framework.NodeScoreList) (string, error) {
	if len(nodeScoreList) == 0 {
		return "", fmt.Errorf("empty priorityList")
	}
	maxScore := nodeScoreList[0].Score
	selected := nodeScoreList[0].Name
	cntOfMaxScore := 1
	for _, ns := range nodeScoreList[1:] {
		if ns.Score > maxScore {
			maxScore = ns.Score
			selected = ns.Name
			cntOfMaxScore = 1
		} else if ns.Score == maxScore {
			cntOfMaxScore++
			if rand.Intn(cntOfMaxScore) == 0 {
				// Replace the candidate with probability of 1/cntOfMaxScore
				selected = ns.Name
			}
		}
	}
	return selected, nil
}
```

它会选择得分最高的节点；如果得分最高的节点有多个，则随机选择一个。

## prioritizeNodes() ##

现在回头看 `prioritizeNodes()`，这个函数的主要作用是同时调用新旧两种扩展方式实现的打分功能。新的方式就是调度框架(Framework)，旧的方式是通过调度器扩展来实现，我们主要关注前者，会在下一节详细分析其中的 Score 和 NormalizeScore 扩展点的执行，而旧的方式相关调用可以自行参阅相关代码。

``` go
func (g *genericScheduler) prioritizeNodes(
	ctx context.Context,
	prof *profile.Profile,
	state *framework.CycleState,
	pod *v1.Pod,
	nodes []*v1.Node,
) (framework.NodeScoreList, error) {
    ...
	if len(g.extenders) == 0 && !prof.HasScorePlugins() {
		result := make(framework.NodeScoreList, 0, len(nodes))
		for i := range nodes {
			result = append(result, framework.NodeScore{
				Name:  nodes[i].Name,
				Score: 1,
			})
		}
		return result, nil
	}
```

如果没有调度器扩展且调度框架中没有 Score 扩展点的插件，则认为所有节点的得分都是 1，然后将结果返回。

``` go
	// Run the Score plugins.
	scoresMap, scoreStatus := prof.RunScorePlugins(ctx, state, pod, nodes)
	if !scoreStatus.IsSuccess() {
		return framework.NodeScoreList{}, scoreStatus.AsError()
	}

	// Summarize all scores.
	result := make(framework.NodeScoreList, 0, len(nodes))

	for i := range nodes {
		result = append(result, framework.NodeScore{Name: nodes[i].Name, Score: 0})
		for j := range scoresMap {
			result[i].Score += scoresMap[j][i].Score
		}
	}

	if len(g.extenders) != 0 && nodes != nil {
        ...
	}

    ...
	return result, nil
}
```

接着调用 `RunScorePlugins()` 执行具体的打分流程。将返回的结果进行格式转换，然后存入 `result` 变量中。然后判断是否有调度扩展插件，如果有的话，则执行这些插件对应的 `Prioritize()` 函数(这个不在这里进行分析)。最后将 `result` 结果返回。

下面分析数据的格式转换过程。

``` go
	RunScorePlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) (PluginToNodeScores, *Status)
```

``` go
// PluginToNodeScores declares a map from plugin name to its NodeScoreList.
type PluginToNodeScores map[string]NodeScoreList

// NodeScoreList declares a list of nodes and their scores.
type NodeScoreList []NodeScore

// NODESCORE is a struct with node name and score.
type NodeScore struct {
	Name  string
	Score int64
}
```

`RunScorePlugins()` 返回的值是一个 Map 对象。key 为插件名称，value 为由 `NodeScore` 组成的数组，而 `NodeScore` 则包含了节点名称和当前节点的得分。也就是说，value 包含了当前插件在每个节点上执行的得分和节点名称。

``` go
	result := make(framework.NodeScoreList, 0, len(nodes))
```

而 `result` 则是一个由 `NodeScore` 组成的数组，每个元素包含了节点的名称和当前节点的总得分。

``` go
	for i := range nodes {
		result = append(result, framework.NodeScore{Name: nodes[i].Name, Score: 0})
		for j := range scoresMap {
			result[i].Score += scoresMap[j][i].Score
		}
	}
```

在 `result` 里没有了插件名称的信息。这个转换过程相当于进行了汇总，即针对某一个节点，将这个节点上所有插件的得分相加，结果作为当前节点的最终得分。

下一节对 `RunScorePlugins()` 进行分析，它会计算每个插件在每个节点上的得分计算过程，主要包括 Score 和 NormalizeScore 扩展点的执行。
