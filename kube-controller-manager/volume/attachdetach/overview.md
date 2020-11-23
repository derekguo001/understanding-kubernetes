# AttachDetach Controller #

AttachDetach Controller 实现存储的 attach 和 detach 的功能。当用户创建了一个 pod 之后，与 pod 关联的某一些 volume 需要将底层的存储 attach 到 pod 所在的节点上。相应地，pod 删除后，对应的底层存储需要从 pod 所在的节点 detach 掉。

AttachDetach Controller 内部有两个缓存对象

- ActualStateOfWorld
- DesiredStateOfWorld

这两个对象代表 Volume 的 attach/detach 状态，第一个是实际状态，第二个是期望状态。这两个对象的详细分析可参见 [ActualStateOfWorld](actualstateofworld.md) 和 [DesiredStateOfWorld](desiredstateofworld.md)。

针对 DesiredStateOfWorld，还有一个名为 DesiredStateOfWorldPopulator 的对象，它会更新 DesiredStateOfWorld 中的缓存数据。详见 [DesiredStateOfWorldPopulator](desiredstateofworldpopulator.md)。

AttachDetach Controller 中还有一个名为 Reconciler 的对象，它会周期性地比较 ActualStateOfWorld 和 DesiredStateOfWorld 这两个缓存对象，然后执行相应的操作。详细分析请参见 [Reconciler](reconciler.md)。

当一个 Volume attach 到一个节点上后，在这个节点的状态信息中会有相应的信息。下面是将一个 PVC 挂载到一个 Pod 上之后，Pod 所在节点的信息。

``` bash
$ kubectl get node k1 -o yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: Node
  metadata:
    annotations:
    ...
    name: k1
  status:
    ...
    volumesAttached:
    - devicePath: ""
      name: kubernetes.io/iscsi/178.104.163.35:3260:iqn.2020-09.com.test:lv1:0
    volumesInUse:
    - kubernetes.io/iscsi/178.104.163.35:3260:iqn.2020-09.com.test:lv1:0
```

在节点的 `Status.VolumesAttached ` 中会列出当前节点上所有已经 attach 的 Volume 的信息。ActualStateOfWorld 也是根据这个字段来判断一个 Volume 是否已经真正 attach 到了一个节点上。在节点的 `Status.volumesInUse` 中会列出当前节点上所有已经 mount 的 Volume，表明这些 Volume 正在使用中。
