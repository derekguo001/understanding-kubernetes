# 概述 #

底层的存储资源在 Kubernetes 中是通过 Volume 的形式来使用的，在上层提供了 PV 和 PVC 对象来供用户直接配置和使用。

先看一个典型的使用流程，用户首先创建 PV 和 PVC 对象，然后创建 Pod，同时将 PVC 关联到 Pod。
对于底层的存储一般需要经历几个阶段：

1. 创建底层的存储资源，例如云硬盘。
2. 将底层的资源 attach 到 Pod 所在的节点上。
3. 在 Pod 所在节点找到对应的存储资源，然后将其格式化并 mount 到 Pod 在节点上对应的目录。

删除时则按照相反的步骤执行。

前两个阶段的工作由于不在 Pod 所在的节点上运行，因此可以由 Controller 完成，而第三个阶段则必须在 Pod 所在节点上运行。对应的，前两个阶段的代码都位于 Kubernetes Controller 中，而第三阶段的代码则位于 Kubelet 中。注意这里有一个例外，也就是第一阶段也可以由 Kubelet 完成，详见 [关于 attach 和 detach 操作](attach-detach.md)

有两个需要注意的地方：
1. 上面的流程中，如果 Volume Plugin 支持动态创建存储资源的话，用户只需要声明 PVC 对象，而 PV 对象则会自动创建。
1. 并非所有类型的存储都要有这三个阶段，例如 Local Volume 不需要经历前两个阶段。

存储相关的代码在 Kubernetes 中主要位于三个地方：

- `pkg/controller/volume` 位于 Kubernetes Controller Manager 中，包含 Volume 相关的 Controller。
- `pkg/kubelet/volumemanager` 位于 Kubelet 中， Kubelet 用来执行 mount/umount 操作。
- `pkg/volume` Volume 相关的公共目录，被上面两个所引用。

## Volume in Kube-Controller-Manager ##

在 Kubernetes Controller Manager 组件中主要包含以下几个与 Volume 相关的 Controller。

- **AttachDetach Controller**

  顾名思义，它用于第二阶段，将底层的存储 attach 到 Pod 所在节点上或者将其从节点上 detach 掉。

  代码位于 `pkg/controller/volume/attachdetach`。

- **PersistentVolume Controller**

  这个 Controller 的作用是维护 PV 和 PVC 对象之间的状态和绑定关系，可以细分为 PV Controller 和 PVC Controller，会分别 watch PV 对象和 PVC 对象，然后分别进行处理。用于上文的第一阶段中。

  代码位于 `pkg/controller/volume/persistentvolume`。

- **Expand Controller**

  用于磁盘的扩容操作。代码位于 `pkg/controller/volume/expand`。

- **PVProtectionController** 和 **PVCProtectionController**

  用于防止正在使用中的 PV 和 PVC 被删除。代码位于 `pkg/controller/volume/pvprotection` 和 `pkg/controller/volume/pvcprotection`。

## Volume in Kubelet ##

在 Kubelet 中包含有一个 Volume Manager 对象，用于管理节点上 Volume，执行相应的 mount/umount 操作，对应于上文中的第三阶段。代码位于 `pkg/kubelet/volumemanager`。

## Volume Plugin ##

在 Kubernetes 的源代码目录中，有一个总的 Volume 目录，具体路径为 `pkg/volume`，这个里面包含了以下几个部分：

- Volume 相关的 interface 的定义。
- Volume Plugin 框架的实现。
- Volume Plugin Manager，用来管理各种 Volume Plugin。
- 具体的 in-tree 类型的 Volume Plugin，包括 Local Volume、NFS 等。
- OperationExecutor，具体 attach/detach、mount/umount 等操作的执行者。

这个文件夹里面的内容不会作为一个组件单独出现，其中的各种 Volume Plugin 会由 Volume Plugin Manager 进行管理，然后 Volume Plugin Manager 则被包含在 Controller 和 Kubelet 中。

在具体执行某个操作时，首先会通过这里的 Volume Plugin Manager 来获取当前 Volume 对应的 Volume Plugin，然后使用这里的 OperationExecutor 来获取 Volume Plugin 中具体的执行函数，最后执行相应的动作。
