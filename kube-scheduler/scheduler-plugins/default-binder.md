# DefaultBinder 调度插件 #

DefaultBinder 调度插件是调度器默认使用用来将 Pod 与节点进行绑定的插件，它实现了 Bind 扩展点。可以同时开启多个 Bind 扩展点的插件。

``` go
// Bind binds pods to nodes using the k8s client.
func (b DefaultBinder) Bind(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) *framework.Status {
	klog.V(3).Infof("Attempting to bind %v/%v to %v", p.Namespace, p.Name, nodeName)
	binding := &v1.Binding{
		ObjectMeta: metav1.ObjectMeta{Namespace: p.Namespace, Name: p.Name, UID: p.UID},
		Target:     v1.ObjectReference{Kind: "Node", Name: nodeName},
	}
	err := b.handle.ClientSet().CoreV1().Pods(binding.Namespace).Bind(ctx, binding, metav1.CreateOptions{})
	if err != nil {
		return framework.AsStatus(err)
	}
	return nil
}
```

通过创建当前 Pod 的 binding 子资源来实现 Pod 与节点之间的绑定。
