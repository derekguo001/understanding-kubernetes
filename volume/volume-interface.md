# Volume 相关的数据结构 #

在 `pkg/volume/plugins.go` 中定义了 Volume 以及 Volume Plugin 相关的一些 interface。在 Kubernetes 中处理 PV 和 PVC 对象时都是使用这些抽象的 interface 进行操作。

|Interface                     |对应的 Plugin|说明|
|---|---|---|
|Volume                        ||所有其它 interface 的 base|
|Attacher                      |AttachableVolumePlugin|将 Volume attach 到 pod 所在的节点上|
|Detacher                      |AttachableVolumePlugin|将 Volume 从 pod所在的节点上 detach|
|Mounter                       |VolumePlugin|将 pod 所在节点上找到的底层存储挂载到 pod 对应的目录中|
|Unmounter                     |VolumePlugin|同上，执行反向操作|
|DeviceMounter                 |DeviceMountableVolumePlugin|将存储设备 mount 到节点上的全局挂载点|
|DeviceUnmounter               |DeviceMountableVolumePlugin|同上，执行反向操作|
|BlockVolumeMapper             |BlockVolumePlugin|存储作为块设备提供给 Pod，将存储映射到 Pod 中|
|BlockVolumeUnmapper           |BlockVolumePlugin|同上，执行反向操作|
|CustomBlockVolumeMapper       |BlockVolumePlugin|实现了 BlockVolumeMapper interface，提供了额外的函数来实现一些额外的工作，主要用于 CSI|
|CustomBlockVolumeUnmapper     |BlockVolumePlugin|同上，执行反向操作|
|BulkVolumeVerifier            |||
|Provisioner                   |ProvisionableVolumePlugin||
|Deleter                       |DeletableVolumePlugin|当指定 Delete 类型的 volume 回收策略时用这个对象去执行 Volume Delete 操作||
|Metrics                       |||
|MetricsProvider               |||
