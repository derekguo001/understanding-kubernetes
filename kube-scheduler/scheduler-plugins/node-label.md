# NodeLabel 调度插件 #

NodeLabel 调度插件实现了 `Filter` 和 `Score` 扩展点，它会根据 ***插件本身的配置*** 对节点进行分类和打分，进而影响 Pod 的调度策略。注意它不是将 Pod 的 Label 与节点的 Lable 比较从而影响调度的。

此插件的配置参数如下：

``` go
type NodeLabelArgs struct {
	PresentLabels []string
	AbsentLabels []string
	PresentLabelsPreference []string
	AbsentLabelsPreference []string
}
```

分别表示：

- 节点必须有的 Label 的列表
- 节点必须不能有的 Label 的列表
- 节点最好有的 Label 的列表
- 节点最好没有的 Label 的列表

其中，前两个参数用于 Filter 扩展点，后两个用于 Score 扩展点。

## Filter ##

``` go
func (pl *NodeLabel) Filter(ctx context.Context, _ *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
	node := nodeInfo.Node()
	if node == nil {
		return framework.NewStatus(framework.Error, "node not found")
	}
	nodeLabels := labels.Set(node.Labels)
	check := func(labels []string, presence bool) bool {
		for _, label := range labels {
			exists := nodeLabels.Has(label)
			if (exists && !presence) || (!exists && presence) {
				return false
			}
		}
		return true
	}
	if check(pl.args.PresentLabels, true) && check(pl.args.AbsentLabels, false) {
		return nil
	}

	return framework.NewStatus(framework.UnschedulableAndUnresolvable, ErrReasonPresenceViolated)
}
```

如果当前节点的 Label 中包含了 PresentLabels 中所有的 Label 并且不包含 AbsentLabels 中任何一个 Label，才会将当前节点加入候选节点列表中。

## Score ##

``` go
const (
	// MaxNodeScore is the maximum score a Score plugin is expected to return.
	MaxNodeScore int64 = 100
    ...
)

func (pl *NodeLabel) Score(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (int64, *framework.Status) {
	nodeInfo, err := pl.handle.SnapshotSharedLister().NodeInfos().Get(nodeName)
	if err != nil {
		return 0, framework.AsStatus(fmt.Errorf("getting node %q from Snapshot: %w", nodeName, err))
	}

	node := nodeInfo.Node()
	score := int64(0)
	for _, label := range pl.args.PresentLabelsPreference {
		if labels.Set(node.Labels).Has(label) {
			score += framework.MaxNodeScore
		}
	}
	for _, label := range pl.args.AbsentLabelsPreference {
		if !labels.Set(node.Labels).Has(label) {
			score += framework.MaxNodeScore
		}
	}
	// Take average score for each label to ensure the score doesn't exceed MaxNodeScore.
	score /= int64(len(pl.args.PresentLabelsPreference) + len(pl.args.AbsentLabelsPreference))

	return score, nil
}
```

在 Score 扩展点，会根据 PresentLabelsPreference 和 AbsentLabelsPreference 这两个参数来对节点进行打分。例如：这两个参数分别是 "[a, b, c], [d]"，当前节点包含的 Label 为 "[a b]"，也就意味着当前节点具有两个"最好有的 Label"(即"a"和"b")，以及没有任何一个"最好没有的 Label"(即"d")。

因此当前的得分为 `100*((1+1)+1)/(3+1)=75`。
