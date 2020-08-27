# 概述 #

底层的存储资源在 Kubernetes 中是通过 Volume 的形式来使用的，在上层提供了 PV 和 PVC 对象来供用户直接配置和使用。

先看一个典型的使用流程，用户首先创建 PV 和 PVC 对象，然后创建 Pod，同时将 PVC 关联到 Pod。
对于底层的存储一般需要经历几个阶段：

1. 创建底层的存储资源，例如云硬盘。
2. 将底层的资源 attach 到 Pod 所在的节点上。
3. 在 Pod 所在节点找到对应的存储资源，然后将其格式化并 mount 到 Pod 在节点上对应的目录。

删除时则按照相反的步骤执行。

前两个阶段的工作由于不在 Pod 所在的节点上运行，因此可以由 Controller 完成，而第三个阶段则必须在 Pod 所在节点上运行。对应的，前两个阶段的代码都位于 Kubernetes Controller 中，而第三阶段的代码则位于 Kubelet 中。注意这里有一个例外，也就是第一阶段也可以由 Kubelet 完成，详见 [关于 attach/detach 操作](../kubelet/volume/overview.md#关于-attachdetach-操作)
