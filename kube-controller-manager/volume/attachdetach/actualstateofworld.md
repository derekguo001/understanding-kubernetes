# actualStateOfWorld #

## 数据结构 ##

actualStateOfWorld 对象是 AttachDetach Controller 内部的一个缓存对象，它保存着每个 Node 上已经挂载的 Volume 信息。

它包含有从 Volume 到 Node 的映射关系信息。

``` go
type actualStateOfWorld struct {
	attachedVolumes map[v1.UniqueVolumeName]attachedVolume
    ...
    ...
}
```

它的结构中包含一个 map，key 为 Volume 名称，value 为 attachedVolume 结构。

``` GO
type attachedVolume struct {
	volumeName v1.UniqueVolumeName
	spec *volume.Spec
	nodesAttachedTo map[types.NodeName]nodeAttachedTo
	devicePath string
}
```

每一个 attachedVolume 结构中除了包含当前 Volume 自身的信息之外，还有当前 Volume 挂载到的 Node 的信息，这也是一个 map 类型，key 为 Node 名称，Value 为 nodeAttachedTo。

``` go
type nodeAttachedTo struct {
	nodeName types.NodeName
	mountedByNode bool
	attachedConfirmed bool
    ...
}
```

nodeAttachedTo 结构主要包括 Node 自身的信息。

本节内容主要分析一下 actualStateOfWorld 的数据结构和主要函数的使用场景，具体的每个函数的实现逻辑不太复杂，就不在这里做详细的分析了，主要就是维护以上三个对象的相互关系。

## 使用场景 ##

actualStateOfWorld 对象实现了两个 interface：

- **ActualStateOfWorldAttacherUpdater**，这个 interface 位于 `pkg/volume/util/operationexecutor/operation_executor.go`，它属于 Volume OperationExecutor 的一部分，详见 [Volume OperationExecutor](../../../volume/operationexecutor.md)。
- **ActualStateOfWorld**，这个 interface 就位于当前文件中。

下面分别进行分析。
