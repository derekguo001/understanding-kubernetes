# OperationExecutor #

## 概述 ##

在 `pkg/volume` 目录下，除了一些抽象的 Volume interface 以及 Volume Plugin 相关的代码之外，还有一个非常重要的对象叫做 OperationExecutor，代码位于 `pkg/volume/util/operationexecutor`。它是真正的动作执行者，在初始化的时候，它会作为 Kube Controller Manager 和 Kubelet 内部的一个对象，在 Kube Controller Manager 或者 Kubelet 执行 Volume 相关操作的时候，例如 attach/detach 和 mount/umount 等操作时，会调用 OperationExecutor 来具体负责执行，而 OperationExecutor 则会通过其内部的 operationGenerator 进一步调用当前 Volume 对应的 Volume Plugin 中的具体函数来完成相关操作。

这一节可以和 Kube Controller Manager、Kubelet 中 Volume 相关的内容结合起来看，这样才可以从全局来理解整个 Volume 处理流程。

```
type operationExecutor struct {
	pendingOperations nestedpendingoperations.NestedPendingOperations
	operationGenerator OperationGenerator
}
```

结构中有两个成员：其中的 pendingOperations 是为了防止同一个 Volume 的多个 attach/detach 操作同时执行，这个对象我们这里只是简单使用，不会深入分析其原理。另一个成员是 operationGenerator，它是 operationExecutor 的核心，它的作用是返回具体各种操作的函数，例如 attach/detach 和 mount/umount 具体执行哪些代码，都是由这个成员来实际返回的，返回后由 operationExecutor 负责具体执行。

Kube Controller Manager 和 Kubelet 在执行 operationExecutor 对象的具体的某个函数时，例如 `AttachVolume()`，一般会传入一个 actualStateOfWorld 参数，这个参数是由 Kube Controller Manager 和 Kubelet 进行定义的，operationExecutor 在执行完相应的操作后，会根据执行成功或者执行失败的情况来更新 actualStateOfWorld 对象，这样可以反映真实的执行结果以便进行后续的处理。

operationExecutor 核心函数有 5 个：

- **AttachVolume()**
- **DetachVolume()**
- **MountVolume()**
- **UnmountVolume()**
- **UnmountDevice()**

需要特别说明的是最后一个，`UnmountDevice()` 是指将存储设备从节点上的全局挂载点执行 umount 操作。关于全局挂载点的说明，可参见 [挂载点](./plugin.md#挂载点)。还有一点就是为什么这里只有 `UnmountDevice()` 操作，而没有对应的 `MountDevice()`，这是因为将存储设备 mount 到节点上的全局挂载点这个操作是在 `MountVolume()` 中实现的。

除了这几个核心函数之外，还有一些辅助的非核心函数，具体逻辑在这里就不再继续展开了。下面对这几个核心函数进行详细的分析。
