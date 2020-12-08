# NodeName 调度插件 #

NodeName 调度插件实现了 Filter 扩展点，当节点名称与当前 Pod 的 `pod.Spec.NodeName` 匹配时，才会将这些节点作为当前 Pod 的备选节点。

代码实现非常简单，就是单纯地比较节点名称与 `pod.Spec.NodeName`。

``` go
// Filter invoked at the filter extension point.
func (pl *NodeName) Filter(ctx context.Context, _ *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
	if nodeInfo.Node() == nil {
		return framework.NewStatus(framework.Error, "node not found")
	}
	if !Fits(pod, nodeInfo) {
		return framework.NewStatus(framework.UnschedulableAndUnresolvable, ErrReason)
	}
	return nil
}

// Fits actually checks if the pod fits the node.
func Fits(pod *v1.Pod, nodeInfo *framework.NodeInfo) bool {
	return len(pod.Spec.NodeName) == 0 || pod.Spec.NodeName == nodeInfo.Node().Name
}
```
