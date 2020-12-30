# SelectorSpread 调度插件 #

SelectorSpread 对于属于 Services、ReplicaSets 和 StatefulSets的Pod，偏好跨多个节点部署，也就是说这个调度插件会让当前 Pod 所属的 Services、ReplicaSets 和 StatefulSets 对象所包含的 Pod 尽可能均匀分布在每个节点上。

当对 Pod 进行均匀分布的时候，会从两个维度对每个节点的得分进行计算：
- ***节点维度***
- ***每个节点所属的 Zone 的维度***

当匹配到的 Pod 所在的节点都不属于任何一个 Zone 的时候，则不会考虑 Zone 维度。在本文中我们会针对这两种情况进行详细的分析。给每个节点的打分算法有一点点复杂，在分析完代码之后会列举几个具体的例子以便更直观地体会这个算法的作用。

SelectorSpread 调度插件实现了 `PreScore`、`Score` 以及 `NormalizeScore` 扩展点。

## PreScore ##

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

在 PreScore 扩展点，会根据当前 Pod 的 Label 找出包含此 Pod 的 "上级" 资源的 Label，并将结果存入到调度缓存中。注意返回的是 Label 的合集。

核心实现在 `DefaultSelector()` 中。

``` go
func DefaultSelector(pod *v1.Pod, sl corelisters.ServiceLister, cl corelisters.ReplicationControllerLister, rsl appslisters.ReplicaSetLister, ssl appslisters.StatefulSetLister) labels.Selector {
	labelSet := make(labels.Set)

	if services, err := GetPodServices(sl, pod); err == nil {
		for _, service := range services {
			labelSet = labels.Merge(labelSet, service.Spec.Selector)
		}
	}

	if rcs, err := cl.GetPodControllers(pod); err == nil {
		for _, rc := range rcs {
			labelSet = labels.Merge(labelSet, rc.Spec.Selector)
		}
	}

	selector := labels.NewSelector()
	if len(labelSet) != 0 {
		selector = labelSet.AsSelector()
	}

	if rss, err := rsl.GetPodReplicaSets(pod); err == nil {
		for _, rs := range rss {
			if other, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector); err == nil {
				if r, ok := other.Requirements(); ok {
					selector = selector.Add(r...)
				}
			}
		}
	}

	if sss, err := ssl.GetPodStatefulSets(pod); err == nil {
		for _, ss := range sss {
			if other, err := metav1.LabelSelectorAsSelector(ss.Spec.Selector); err == nil {
				if r, ok := other.Requirements(); ok {
					selector = selector.Add(r...)
				}
			}
		}
	}

	return selector
}
```

可以看出会将 Pod 与其它所有上级资源进行匹配，以 service 为例，会找到所有的匹配的 service，然后将它们的 Label 进行合并，最后返回。

``` go
	if services, err := GetPodServices(sl, pod); err == nil {
		for _, service := range services {
			labelSet = labels.Merge(labelSet, service.Spec.Selector)
		}
	}
```

## Score ##

Score 扩展点会计算每个节点的得分并将其返回。注意 Score 不像 PreScore 那样每个 Pod 执行一次，而是针对每个 Pod 执行 N 次，N 是节点的数量。每次返回值是当前节点的得分。

``` go
func (pl *SelectorSpread) Score(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (int64, *framework.Status) {
	if skipSelectorSpread(pod) {
		return 0, nil
	}

	c, err := state.Read(preScoreStateKey)
    ...

	s, ok := c.(*preScoreState)
    ...

	nodeInfo, err := pl.sharedLister.NodeInfos().Get(nodeName)
    ...

	count := countMatchingPods(pod.Namespace, s.selector, nodeInfo)
	return int64(count), nil
}
```

首先，会判断当前节点是否需要跳过 SelectorSpread 的评分，判断规则如下：

``` go
func skipSelectorSpread(pod *v1.Pod) bool {
	return len(pod.Spec.TopologySpreadConstraints) != 0
}
```

也就意味着如果当前节点设置了[拓扑分布约束](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-topology-spread-constraints/)，则会将当前节点的得分设为 0。注意这与在当前节点禁用 SelectorSpread 插件不一样，如果禁用的话当前节点的得分为满分 100 分。

如果没有设置拓扑分布约束，则使用 `countMatchingPods()` 来计算当前节点的得分。

``` go
func countMatchingPods(namespace string, selector labels.Selector, nodeInfo *framework.NodeInfo) int {
	if len(nodeInfo.Pods) == 0 || selector.Empty() {
		return 0
	}
	count := 0
	for _, p := range nodeInfo.Pods {
		// Ignore pods being deleted for spreading purposes
		// Similar to how it is done for SelectorSpreadPriority
		if namespace == p.Pod.Namespace && p.Pod.DeletionTimestamp == nil {
			if selector.Matches(labels.Set(p.Pod.Labels)) {
				count++
			}
		}
	}
	return count
}
```

这个函数会遍历当前节点上与当前 Pod 处于同一个 Namespace 中的其它所有 Pod，如果这些 Pod 没有处于删除状态，并且与 PreScore 阶段得到的 Label 相匹配，则将当前节点的得分+1。

也就是说 Score 扩展点返回的值是每个节点上与当前 Pod 处于同一个 Namespace 且 Label 与 PreScore 扩展点返回的 Label 相匹配的所有 Pod 的数量。

## NormalizeScore ##

在 Score 扩展点计算出来的结果并不是每个节点最终的得分，还需要通过在 NormalizeScore 进行修正。这里的修正其实是计算过程中非常重要的一个步骤，并非可有可无。

SelectorSpread 的 NormalizeScore 扩展点的逻辑才是计算节点得分的最重要的部分。

分两种情况进行讨论：
1. 匹配到的 Pod 所在的节点不属于任何一个 Zone 的情况
2. 匹配到的 Pod 所在的节点有的属于某个 Zone 的情况

### 不考虑 Zone 的情况 ###

``` go

func (pl *SelectorSpread) NormalizeScore(ctx context.Context, state *framework.CycleState, pod *v1.Pod, scores framework.NodeScoreList) *framework.Status {
	if skipSelectorSpread(pod) {
		return nil
	}
```

``` go
func skipSelectorSpread(pod *v1.Pod) bool {
	return len(pod.Spec.TopologySpreadConstraints) != 0
}
```

首先，如果当前 Pod 已经设置了拓扑分布约束，那么则当前节点得分为0。

``` go
	maxCountByNodeName := int64(0)

	for i := range scores {
		if scores[i].Score > maxCountByNodeName {
			maxCountByNodeName = scores[i].Score
		}
        ...
	}
```

找出在 Score 扩展点得分最大的节点，然后把这个节点的得分存入 maxCountByNodeName。

```
    ...
	maxCountByNodeNameFloat64 := float64(maxCountByNodeName)
	MaxNodeScoreFloat64 := float64(framework.MaxNodeScore)

	for i := range scores {
		// initializing to the default/max node score of maxPriority
		fScore := MaxNodeScoreFloat64
		if maxCountByNodeName > 0 {
			fScore = MaxNodeScoreFloat64 * (float64(maxCountByNodeName-scores[i].Score) / maxCountByNodeNameFloat64)
		}
        ...
		scores[i].Score = int64(fScore)
	}
	return nil
}
```

最后计算当前节点的最终得分为 `100*(最大节点的得分-当前节点的得分)/最大节点的得分`。

### 考虑 Zone 的情况 ###

如果考虑 Zone 的话，相当于多了一个考察的维度，与原先单纯考虑 Node 维度的结果要加起来为总的得分。这两个维度的在总得分中所占的比重不同，Zone 的为 2/3，Node 的为 1/3。

下面在 PreScore 的代码分析过程中穿插的 Zone 相关的代码。

``` go
	for i := range scores {
        ...
		nodeInfo, err := pl.sharedLister.NodeInfos().Get(scores[i].Name)
		if err != nil {
			return framework.NewStatus(framework.Error, fmt.Sprintf("getting node %q from Snapshot: %v", scores[i].Name, err))
		}
		zoneID := utilnode.GetZoneKey(nodeInfo.Node())
		if zoneID == "" {
			continue
		}
		countsByZone[zoneID] += scores[i].Score
	}
```

首先计算出总共有多少个 Zone，并且计算出每个 Zone 的得分，当前 Zone 所包含的节点的得分加起来的和即为当前 Zone 的得分。

``` go
	for zoneID := range countsByZone {
		if countsByZone[zoneID] > maxCountByZone {
			maxCountByZone = countsByZone[zoneID]
		}
	}
```

接着计算得分最高的 Zone，将得分存入 maxCountByZone。

``` go
	for i := range scores {
        ...
		if haveZones {
			nodeInfo, err := pl.sharedLister.NodeInfos().Get(scores[i].Name)
			if err != nil {
				return framework.NewStatus(framework.Error, fmt.Sprintf("getting node %q from Snapshot: %v", scores[i].Name, err))
			}

			zoneID := utilnode.GetZoneKey(nodeInfo.Node())
			if zoneID != "" {
				zoneScore := MaxNodeScoreFloat64
				if maxCountByZone > 0 {
					zoneScore = MaxNodeScoreFloat64 * (float64(maxCountByZone-countsByZone[zoneID]) / maxCountByZoneFloat64)
				}
				fScore = (fScore * (1.0 - zoneWeighting)) + (zoneWeighting * zoneScore)
			}
		}
		scores[i].Score = int64(fScore)
	}
	return nil
}
```

这里分为两个小步骤：
1. 计算当前节点在其所在的Zong维度的得分：`100*(最大Zone的得分-当前Zone的得分)/最大Zone的得分`。
2. 按照权重算当前节点最终的得分：`节点维度得分*(1-Zone维度的权重)+Zone维度的得分*Zong维度的权重`。

其中 Zone 维度的权重为 2/3，也就意味着 节点维度的权重为`(1-2/3)=1/3`。

## 例子 ##

### 不考虑 Zone 的情况 ###

环境信息：

有 2个 Label：

``` go
labels1 := map[string]string{
	"foo": "bar",
	"baz": "blah",
}
labels2 := map[string]string{
	"bar": "foo",
	"baz": "blah",
}
```

有 2 个节点 n1 和 n2。

#### 例子1 ####

当前 Pod 的 Label 为 label1。
节点1上有2个 Pod，Label 分别为 label1 和 label2；节点2上有2个Pod，Label 都是 label1。
有一个 Service，Label 为 label1

这两个节点的得分计算过程如下：
PreScore 扩展点：先匹配所有的 Label，匹配的结果是 Service 的 Label: label1。
Score 扩展点：分别计算每个节点的得分：节点1上第一个 Pod 的 Label 为 label1，与 PreSore 返回的 Label 匹配，当前节点得分为1；节点2上有2个 Label 为 label1 的 Pod，得分为2。
NormalizeScore 扩展点：找出得分最大的节点，其得分为2。节点1和节点2的最终得分分别为：100*(2-1)/2=50、100*(2-2)/2=0。

#### 例子2 ####

当前 Pod 的 Label 为 label1。
节点1上有2个 Pod，Label 分别为 label1 和 label2；节点2上有1个Pod，Label 是 label1。
有一个 Service，Label 为 "baz": "blah"
有一个 ReplicationController，Label 为 "foo": "bar"。

这两个节点的得分计算过程如下：
PreScore 扩展点：先匹配所有的 Label，匹配的结果是 "baz": "blah" 和 "foo": "bar"。
Score 扩展点：分别计算每个节点的得分：节点1上第一个 Pod 的 Label 为 label1，与 PreSore 返回的 Label 匹配，第2个 Pod 的 Label 为 label2，只与 "baz": "blah" 匹配，但与 "foo": "bar" 不匹配，因此不能计算入内，因此节点1得分为1；节点2上有1个 Label 为 label1 的 Pod，匹配，得分为1。
NormalizeScore 扩展点：找出得分最大的节点，其得分为1。节点1和节点2的最终得分分别为：100*(1-1)/2=0、100*(1-1)/2=0。

这里需要特别注意的地方是 Label 与当前节点上的 Pod 进行匹配的时候，结果是取 "交集"。

### 考虑 Zone 的情况 ###

有 2个 Label：

``` go
labels1 := map[string]string{
	"foo": "bar",
	"baz": "blah",
}
labels2 := map[string]string{
	"bar": "foo",
	"baz": "blah",
}
```

有 6 个节点和 3 个 Zone，对应关系为
|节点|Zone|
|---|---|
|1|1|
|2|2|
|3|2|
|4|3|
|5|3|
|6|3|

#### 例子3 ####

当前 Pod 的 Label 为 label1
前 5 个节点上分别有一个 Pod，对应的 Label 分别为 label2、label1、label1、label2 和 label1。

计算过程：

|节点|1(Zone 1)|2(Zone 2)|3(Zone 2)|4(Zone 3)|5(Zone 3)|6(Zone 3)|说明|
|---|---|---|---|---|---|---|---|
|节点计数|0|1|1|0|1|0|最大节点的得分为1|
|节点维度最初得分|100|0|0|100|0|100||
|节点维度最终得分|33|0|0|33|0|33|为节点维度最初得分*1/3|
|节点的Zone的计数|0|2|2|1|1|1||
|节点的Zone最初得分|100|0|0|50|50|50|100*(最大Zone得分-当前Zone得分)/最大Zone得分|
|节点的Zone最终得分|66|0|0|33|33|33|节点的Zone最初得分*2/3|
|节点最终得分|100|0|0|66|33|66|节点维度最终得分+节点的Zone最终得分|

#### 例子4 ####

当前 Pod 的 Label 为 label1
前 4 个节点上分别有一个 Pod，对应的 Label 分别为label1、label1、label2 和 label1。

计算过程：

|节点|1(Zone 1)|2(Zone 2)|3(Zone 2)|4(Zone 3)|5(Zone 3)|6(Zone 3)|说明|
|---|---|---|---|---|---|---|---|
|节点计数|1|1|0|1|0|0|最大节点的得分为1|
|节点维度最初得分|0|0|100|0|100|100||
|节点维度最终得分|0|0|33|0|33|33|为节点维度最初得分*1/3|
|节点的Zone的计数|1|1|1|1|1|1||
|节点的Zone最初得分|0|0|0|0|0|0|100*(最大Zone得分-当前Zone得分)/最大Zone得分|
|节点的Zone最终得分|0|0|0|0|0|0|节点的Zone最初得分*2/3|
|节点最终得分|0|0|33|0|33|33|节点维度最终得分+节点的Zone最终得分|
