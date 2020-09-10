# 关于 attach 和 detach 操作

在将底层的存储挂载到 Pod 对应的目录之前，需要将底层的 Volume 先 attach 到 Pod 所在的节点上，这个 attach 操作可以由 Kube Controller Manager 来完成，也可以由 Kubelet 来完成。在 Kubelet 中有一个参数对其进行配置。

```
--enable-controller-attach-detach

Enables the Attach/Detach controller to manage attachment/detachment of volumes scheduled to this node, and disables kubelet from executing any attach/detach operations (default true)
```

这个参数默认情况下为 true，也就意味着会使用 Kube Controller Manager(具体是其中的 AttachDetach Controller)来进行 attach/detach 操作，这种情况下 Kubelet 会为当前节点增加以下的 Annotation:

``` go
	ControllerManagedAttachAnnotation string = "volumes.kubernetes.io/controller-managed-attach-detach"
```

然后，在 Kube Controller Manager 进行处理的时候，也只会处理带有这个 Annotation 的节点上的 Volume。

如果 `--enable-controller-attach-detach` 参数设置为 false ，Kubelet(具体是其中的 Volume Manager 组件)会负责 attach/detach 操作，这个节点也不会被添加上 `ControllerManagedAttachAnnotation` 这个 Annotation，Kube Controller Manager 便可知道此节点上的所有 Volume 由 Kubelet 来执行 attach/detach 操作，Kube Controller Manager 自己会忽略这个节点上所有的 Volume。
