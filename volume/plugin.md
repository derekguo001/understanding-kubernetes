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

## 挂载点 ##

在执行 mount/umount 操作的时候，会涉及到很多挂载点，容易混淆。

1. 局部挂载点

   即与 Pod 关联的 Volume 挂载点，这个就是某个 Pod 关联某个 PVC 时，Pod 对应的 Volume 在节点上的路径。具体分为两种情况：

   - 当存储作为文件系统提供给 Pod 使用时，格式为 `pods/{podUid}/volume/{escapeQualifiedPluginName}/{volumeName}`。
     例如 FC SAN 的局部挂载点为 `/var/lib/kubelet/pods/50fbcc62-0cb2-4b19-975b-542ae1cedbca/volumes/kubernetes.io~fc/pv1`。

   - 当存储作为块设备提供给 Pod 使用时，格式为 `pods/{podUid}/volumeDevices/{escapeQualifiedPluginName}/{volumeName}`。

2. 全局挂载点

   由于一个存储设备同时只能挂载到一个目录中，但在 Kubernetes 中一个 Volume 却可以同时被一个节点上的多个 Pod 使用，为了解决这个问题，可以先将存储设备挂载到节点上一个全局挂载点，然后再将这个全局挂载点以 bind 的模式挂载到各个 Pod 的 Volume 对应的目录。因此对于底层需要执行 mount 操作的存储类型，在执行 mount 操作时实际上内部执行了两次 mount。

   ``` go
   type DeviceMounter interface {
       GetDeviceMountPath(spec *Spec) (string, error)
       MountDevice(spec *Spec, devicePath string, deviceMountPath string) error
   }
   ```

   当存储作为文件系统提供给 Pod 使用时，`DeviceMounter` interface 可以用来执行全局挂载操作，其中的 `GetDeviceMountPath()` 返回的就是文件系统的全局挂载点。

   当存储作为块设备提供给 Pod 使用时，`BlockVolume` interface 的 `GetGlobalMapPath()` 返回的就是块设备的全局挂载点。

   全局挂载点的具体位置与 Volume Plugin 的实现有关，当存储无论作为文件系统还是作为块设备提供给 Pod 使用时，全局路径都为 `plugins/kubernetes.io/{PluginName}/{DefaultKubeletVolumeDevicesDirName}/{volumePluginDependentPath}`，也就是说最后一个字段由 Volume Plugin 自己决定。例如：FC SAN 的全局挂载点为 `/var/lib/kubelet/plugins/kubernetes.io/fc/201290b11c051614-203290b11c051614-lun-0`。

|存储提供方式|挂载点类型|实现函数所属的interface|实现函数|格式(都位于`/var/lib/kubelet`目录下)|
|---|---|---|---|---|
|文件系统|局部挂载点|Volume|`GetPath()`|`pods/{podUid}/volume/{escapeQualifiedPluginName}/{volumeName}`|
|块设备|局部挂载点|BlockVolume|`GetPodDeviceMapPath()`|`pods/{podUid}/volumeDevices/{escapeQualifiedPluginName}/{volumeName}`|
|文件系统|全局挂载点|DeviceMounter|`GetDeviceMountPath()`|`plugins/kubernetes.io/{PluginName}/{DefaultKubeletVolumeDevicesDirName}/{volumePluginDependentPath}`|
|块设备|全局挂载点|BlockVolume|`GetGlobalMapPath()`|`plugins/kubernetes.io/{PluginName}/{DefaultKubeletVolumeDevicesDirName}/{volumePluginDependentPath}`|

# 参考 #

- https://github.com/kubernetes/kubernetes/issues/51329
