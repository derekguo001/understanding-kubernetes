# NodeUnschedulable 调度插件 #

NodeUnschedulable 调度插件实现了 `Filter` 扩展点，如果一个节点被标记为不可调度的，同时当前 Pod 又不能容忍此污点，那么此调度插件就不会把这个节点加入到候选节点中。

``` go
func (pl *NodeUnschedulable) Filter(ctx context.Context, _ *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
    ...
	podToleratesUnschedulable := v1helper.TolerationsTolerateTaint(pod.Spec.Tolerations, &v1.Taint{
		Key:    v1.TaintNodeUnschedulable,
		Effect: v1.TaintEffectNoSchedule,
	})
	// TODO (k82cn): deprecates `node.Spec.Unschedulable` in 1.13.
	if nodeInfo.Node().Spec.Unschedulable && !podToleratesUnschedulable {
		return framework.NewStatus(framework.UnschedulableAndUnresolvable, ErrReasonUnschedulable)
	}
	return nil
}
```

``` go
const (
	TaintNodeUnschedulable = "node.kubernetes.io/unschedulable"
)

const (
	TaintEffectNoSchedule TaintEffect = "NoSchedule"
)
```

首先，会调用 `TolerationsTolerateTaint()` 来判断当前 Pod 是否可以容忍 `不可调度` 的污点。这个函数的详细解析可参见 [TaintToleration](./taint-toleration.md) 一节。

然后，如果节点已经被标记为不可调度，并且 Pod 不能容忍 `不可调度` 的污点，那么就会返回一个错误信息，然后当前节点便不会被加到候选节点中。

这里的 `节点不可调度` 是判断依据是 `Node.Spec.Unschedulable` 字段，如果手动给一个正常的节点添加此字段，那么同时会自动为此节点添加一个对应的 taint，如下所示。

``` yaml
  spec:
    taints:
    - effect: NoSchedule
      key: node.kubernetes.io/unschedulable
      timeAdded: "2020-12-02T13:00:56Z"
    unschedulable: true
```
