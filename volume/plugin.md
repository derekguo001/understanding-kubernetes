## Volume Plugin 框架 ##

为了在 Kubernetes 中支持各种外部的存储，对各种存储的操作进行了抽象，定义出一系列 Volume Plugin 对象。代码位于 `pkg/volume/plugins.go`。

|Plugin Interface               |说明|
|---|---|
|VolumePlugin                   |最基本的 interface，所有的 Volume Plugin 都基于 VolumePlugin|
|PersistentVolumePlugin         |可以提供持久化的存储(这个是相对于临时存储而言的)|
|RecyclableVolumePlugin     	|支持 Recycle 类型的 Volume 回收策略|
|DeletableVolumePlugin      	|支持 Delete 类型的 Volume 回收策略|
|ProvisionableVolumePlugin  	|支持创建 PV 和 PVC 对应的底层存储资源|
|AttachableVolumePlugin     	|支持 attach 和 detach 操作|
|DeviceMountableVolumePlugin	|Volume 在挂载到 Pod 对应的目录之前，需要先将 Volume 从远端 mount 到节点上的全局挂载点，这个 interface 就是执行全局挂载点的 mount/umount 操作|
|ExpandableVolumePlugin     	|支持扩容|
|NodeExpandableVolumePlugin 	|支持使用 `NodeExpand()` 进行扩容|
|VolumePluginWithAttachLimits   |对 attach 到每个节点上的 Volume 数量有限制|
|BlockVolumePlugin          	|支持 Block 类型的存储，即将存储作为块设备提供给 Pod|

这些 plugin interface 中，除了 AttachableVolumePlugin 继承于 DeviceMountableVolumePlugin 之外，其它所有 plugin 都继承于基础的 VolumePlugin，也包括 DeviceMountableVolumePlugin，也就意味着 AttachableVolumePlugin 间接继承于 VolumePlugin。

我们先来看最基础的 `VolumePlugin`。

``` go
type VolumePlugin interface {
	Init(host VolumeHost) error
	GetPluginName() string
	GetVolumeName(spec *Spec) (string, error)
	CanSupport(spec *Spec) bool
    ...
	NewMounter(spec *Spec, podRef *v1.Pod, opts VolumeOptions) (Mounter, error)
	NewUnmounter(name string, podUID types.UID) (Unmounter, error)
    ...
}
```

这个 interface 定义了所有 Volume Plugin 都必须实现的一些函数。例如其中的 `NewMounter()` 返回一个上一节提到的 `Mounter` 对象，用于执行将存储挂载到 pod 对应的目录。

注意 `VolumePlugin` 中并没有获取 attach/detach 对象的函数，原因是并非所有的存储都需要将存储 attach 到 pod 所在节点或者从节点上 detach。注意上表中有一个 `AttachableVolumePlugin` 对象。

``` go
type AttachableVolumePlugin interface {
	DeviceMountableVolumePlugin
	NewAttacher() (Attacher, error)
	NewDetacher() (Detacher, error)
    ...
}
```

它提供了我们需要的函数。上面表格中的其它对象也是类似的，例如要实现一个新的 Volume Plugin，需要实现 `Recyc` 类型的 Volume 回收策略的功能，那么这个 Plugin 就需要实现表格中的 `RecyclableVolumePlugin`，以及其中的 `Recycle()` 函数。

``` go
type RecyclableVolumePlugin interface {
	VolumePlugin

	Recycle(pvName string, spec *Spec, eventRecorder recyclerclient.RecycleEventRecorder) error
}
```

RequiresRemount() 判断是否需要重新执行 mount 操作，例如 DownAPI 相关的

# 参考 #

- https://github.com/kubernetes/kubernetes/issues/51329
