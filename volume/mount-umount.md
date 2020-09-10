# 关于 mount 和 umount 操作

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

下表是各种 interface 和其对应的函数与各种 umount 操作的对应关系，可参考 [Volume 相关的数据结构](volume-interface.md) 和 [Volume Plugin 框架](plugin.md)

|存储提供方式|挂载点类型|实现函数所属的interface|实现函数|格式(都位于`/var/lib/kubelet`目录下)|
|---|---|---|---|---|
|文件系统|局部挂载点|Volume|`GetPath()`|`pods/{podUid}/volume/{escapeQualifiedPluginName}/{volumeName}`|
|块设备|局部挂载点|BlockVolume|`GetPodDeviceMapPath()`|`pods/{podUid}/volumeDevices/{escapeQualifiedPluginName}/{volumeName}`|
|文件系统|全局挂载点|DeviceMounter|`GetDeviceMountPath()`|`plugins/kubernetes.io/{PluginName}/{DefaultKubeletVolumeDevicesDirName}/{volumePluginDependentPath}`|
|块设备|全局挂载点|BlockVolume|`GetGlobalMapPath()`|`plugins/kubernetes.io/{PluginName}/{DefaultKubeletVolumeDevicesDirName}/{volumePluginDependentPath}`|
