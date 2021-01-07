# Score 和 NormalizeScore 扩展点的执行 #

在上一节分析了打分过程的整体框架，本节详细分析任意一个插件针对每个节点是如何进行打分的，代码在 `RunScorePlugins()` 中。

```
func (f *framework) RunScorePlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) (ps PluginToNodeScores, status *Status) {
    ...
	pluginToNodeScores := make(PluginToNodeScores, len(f.scorePlugins))
	for _, pl := range f.scorePlugins {
		pluginToNodeScores[pl.Name()] = make(NodeScoreList, len(nodes))
	}
    ...
```

首先定义返回值变量 `pluginToNodeScores`，它是一个 Map 对象。key 为插件名称，value 为由 `NodeScore` 组成的数组，而 `NodeScore` 则包含了节点名称和当前节点的得分。也就是说，value 包含了当前插件在每个节点上执行的得分和节点名称。这段代码对 `pluginToNodeScores` 进行初始化，用插件名称填充其 key 值。

接下来执行具体的打分过程

``` go
	// Run Score method for each node in parallel.
	workqueue.ParallelizeUntil(ctx, 16, len(nodes), func(index int) {
		for _, pl := range f.scorePlugins {
			nodeName := nodes[index].Name
			s, status := f.runScorePlugin(ctx, pl, state, pod, nodeName)
			if !status.IsSuccess() {
				errCh.SendErrorWithCancel(fmt.Errorf(status.Message()), cancel)
				return
			}
			pluginToNodeScores[pl.Name()][index] = NodeScore{
				Name:  nodeName,
				Score: int64(s),
			}
		}
	})
    ...

	// Run NormalizeScore method for each ScorePlugin in parallel.
	workqueue.ParallelizeUntil(ctx, 16, len(f.scorePlugins), func(index int) {
		pl := f.scorePlugins[index]
		nodeScoreList := pluginToNodeScores[pl.Name()]
		if pl.ScoreExtensions() == nil {
			return
		}
		status := f.runScoreExtension(ctx, pl, state, pod, nodeScoreList)
		if !status.IsSuccess() {
			err := fmt.Errorf("normalize score plugin %q failed with error %v", pl.Name(), status.Message())
			errCh.SendErrorWithCancel(err, cancel)
			return
		}
	})
    ...

	// Apply score defaultWeights for each ScorePlugin in parallel.
	workqueue.ParallelizeUntil(ctx, 16, len(f.scorePlugins), func(index int) {
		pl := f.scorePlugins[index]
		// Score plugins' weight has been checked when they are initialized.
		weight := f.pluginNameToWeightMap[pl.Name()]
		nodeScoreList := pluginToNodeScores[pl.Name()]

		for i, nodeScore := range nodeScoreList {
			// return error if score plugin returns invalid score.
			if nodeScore.Score > int64(MaxNodeScore) || nodeScore.Score < int64(MinNodeScore) {
				err := fmt.Errorf("score plugin %q returns an invalid score %v, it should in the range of [%v, %v] after normalizing", pl.Name(), nodeScore.Score, MinNodeScore, MaxNodeScore)
				errCh.SendErrorWithCancel(err, cancel)
				return
			}
			nodeScoreList[i].Score = nodeScore.Score * int64(weight)
		}
	})
    ...

	return pluginToNodeScores, nil
}
```

整个打分过程分成三个步骤：

1. 针对每个节点执行，调用所有的打分插件的 Score 扩展点进行初步打分
2. 针对每个打分插件，调用 NormalizeScore 扩展点对上一步的结果进行修正
3. 结合每个插件自身的权重，计算每个节点最终的得分

下面对这三个步骤分别进行分析。

## Score 扩展点的执行 ##

``` go
	// Run Score method for each node in parallel.
	workqueue.ParallelizeUntil(ctx, 16, len(nodes), func(index int) {
		for _, pl := range f.scorePlugins {
			nodeName := nodes[index].Name
			s, status := f.runScorePlugin(ctx, pl, state, pod, nodeName)
			if !status.IsSuccess() {
				errCh.SendErrorWithCancel(fmt.Errorf(status.Message()), cancel)
				return
			}
			pluginToNodeScores[pl.Name()][index] = NodeScore{
				Name:  nodeName,
				Score: int64(s),
			}
		}
	})
```

会针对每一个节点，执行所有的打分插件，通过调用 `runScorePlugin()` 来给节点进行打分。

``` go
func (f *framework) runScorePlugin(ctx context.Context, pl ScorePlugin, state *CycleState, pod *v1.Pod, nodeName string) (int64, *Status) {
	if !state.ShouldRecordPluginMetrics() {
		return pl.Score(ctx, state, pod, nodeName)
	}
	startTime := time.Now()
	s, status := pl.Score(ctx, state, pod, nodeName)
	f.metricsRecorder.observePluginDurationAsync(score, pl.Name(), status, metrics.SinceInSeconds(startTime))
	return s, status
}
```

`runScorePlugin()` 会调用每个打分插件的 Score 扩展点。

## NormalizeScore 扩展点的执行 ##

``` go
	// Run NormalizeScore method for each ScorePlugin in parallel.
	workqueue.ParallelizeUntil(ctx, 16, len(f.scorePlugins), func(index int) {
		pl := f.scorePlugins[index]
		nodeScoreList := pluginToNodeScores[pl.Name()]
		if pl.ScoreExtensions() == nil {
			return
		}
		status := f.runScoreExtension(ctx, pl, state, pod, nodeScoreList)
		if !status.IsSuccess() {
			err := fmt.Errorf("normalize score plugin %q failed with error %v", pl.Name(), status.Message())
			errCh.SendErrorWithCancel(err, cancel)
			return
		}
	})
```

会并行遍历所有的打分插件，通过调用 `runScoreExtension()` 对上一步的得分进行修正。

``` go
func (f *framework) runScoreExtension(ctx context.Context, pl ScorePlugin, state *CycleState, pod *v1.Pod, nodeScoreList NodeScoreList) *Status {
	if !state.ShouldRecordPluginMetrics() {
		return pl.ScoreExtensions().NormalizeScore(ctx, state, pod, nodeScoreList)
	}
	startTime := time.Now()
	status := pl.ScoreExtensions().NormalizeScore(ctx, state, pod, nodeScoreList)
	f.metricsRecorder.observePluginDurationAsync(scoreExtensionNormalize, pl.Name(), status, metrics.SinceInSeconds(startTime))
	return status
}
```

`runScoreExtension()` 会调用此插件的 NormalizeScore 扩展点。

需要注意的是 `NormalizeScore()` 的参数是节点的列表，也就意味着通过遍历每个节点来分别调用 `NormalizeScore()`。

另外，在实际的调度插件的实现中，NormalizeScore 扩展点的功能可以不仅仅局限于对上一步的得分进行 `修正`，而是作为最终得分的一个主要步骤甚至核心步骤进行实现。例如在 [SelectorSpread](../../scheduler-plugins/selector-spread.md) 插件中 NormalizeScore 扩展点就是核心计算过程。

## 计算最终得分 ##

在上面的两个步骤中，针对每个节点，每个打分插件都计算出了对应的得分，现在还需要考虑一个因素，即所有实现了打分功能的插件，这些插件本身的权重是不一样的。因此需要是根据每个插件自身的权重，对结果进行修正。

每个插件的权重存储于 Framework.pluginNameToWeightMap 字段中，详见 [Kubernetes 调度框架](../../component/framework.md)，这些权重有一些默认值，可参见 [Algorithm Provider](../../component/algorithm-provider.md)。

``` go
		Score: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: noderesources.BalancedAllocationName, Weight: 1},
				{Name: imagelocality.Name, Weight: 1},
				{Name: interpodaffinity.Name, Weight: 1},
				{Name: noderesources.LeastAllocatedName, Weight: 1},
				{Name: nodeaffinity.Name, Weight: 1},
				{Name: nodepreferavoidpods.Name, Weight: 10000},
				{Name: defaultpodtopologyspread.Name, Weight: 1},
				{Name: tainttoleration.Name, Weight: 1},
			},
		},
```

用户也可以通过配置文件对这些默认值进行覆盖修改。

``` go
	workqueue.ParallelizeUntil(ctx, 16, len(f.scorePlugins), func(index int) {
		pl := f.scorePlugins[index]
		// Score plugins' weight has been checked when they are initialized.
		weight := f.pluginNameToWeightMap[pl.Name()]
		nodeScoreList := pluginToNodeScores[pl.Name()]

		for i, nodeScore := range nodeScoreList {
			// return error if score plugin returns invalid score.
			if nodeScore.Score > int64(MaxNodeScore) || nodeScore.Score < int64(MinNodeScore) {
				err := fmt.Errorf("score plugin %q returns an invalid score %v, it should in the range of [%v, %v] after normalizing", pl.Name(), nodeScore.Score, MinNodeScore, MaxNodeScore)
				errCh.SendErrorWithCancel(err, cancel)
				return
			}
			nodeScoreList[i].Score = nodeScore.Score * int64(weight)
		}
	})
```

在执行时会并行遍历所有的打分插件，先获取此插件的权重，然后对所有的节点进行遍历，如果得分小于 0 或者大于 100，则报错返回，否则，会将当前插件在当前节点的得分乘以当前节点的权重，然后将结果重新保存回原来的位置。相当于对得分进行了修正。
