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

### ActualStateOfWorldAttacherUpdater interface ###

``` go
type ActualStateOfWorldAttacherUpdater interface {
	MarkVolumeAsAttached(volumeName v1.UniqueVolumeName, volumeSpec *volume.Spec, nodeName types.NodeName, devicePath string) error
	MarkVolumeAsUncertain(volumeName v1.UniqueVolumeName, volumeSpec *volume.Spec, nodeName types.NodeName) error
	MarkVolumeAsDetached(volumeName v1.UniqueVolumeName, nodeName types.NodeName)
	RemoveVolumeFromReportAsAttached(volumeName v1.UniqueVolumeName, nodeName types.NodeName) error
	AddVolumeToReportAsAttached(volumeName v1.UniqueVolumeName, nodeName types.NodeName)
}
```

- `MarkVolumeAsAttached()`。

  作用是将 Volume 标记为已经 attach 的状态。它主要用在两个地方：

  1. 当执行完标准的 attach 之后调用。由于 attach 动作是由 Volume OperationExecutor 完成的，因此这些代码也位于 Volume OperationExecutor 中，具体而言是在 `AttachVolume()` 下的 [GenerateAttachVolumeFunc()](../../../volume/operationexecutor.md#AttachVolume()) 中。
  2. 当 AttachDetach Controller 启动后，需要填充 actualStateOfWorld 对象，这时候会遍历所有 Node，找出已经挂载的 Volume，然后调用此函数将其在 actualStateOfWorld 中标记为已 attach 的状态。详见 [populateActualStateOfWorld()](attachdetach.md#populateActualStateOfWorld())。

- `MarkVolumeAsUncertain()`。

  顾名思义是将 Volume 标记为不确定状态，这个函数只在一个地方调用，即当 attach 一个 Volume 的动作失败的时候，具体而言是在 `AttachVolume()` 下的 [GenerateAttachVolumeFunc()](../../../volume/operationexecutor.md#AttachVolume()) 中。。

- `MarkVolumeAsDetached()`。

  作用是将 Volume 标记为已经 detach 的状态。它主要用在两个地方：

  1. 当执行完标准的 detach 之后调用，由于 detach 动作是由 Volume OperationExecutor 完成的，因此这些代码也位于 Volume OperationExecutor 中，具体而言是在 `GenerateDetachVolumeFunc()` 中。
  2. AttachDetach Controller Reconciler 周期性地运行时，会找出 DesiredStateOfWorld 中已经不存在、但在 ActualStateOfWorld 中存在的 Volume，然后执行 detach 操作，接着会执行 `MarkVolumeAsDetached()`，详见 [Reconciler](reconciler.md)。

- `RemoveVolumeFromReportAsAttached()`

  AttachDetach Controller Reconciler 会找出 DesiredStateOfWorld 中已经不存在、但在 ActualStateOfWorld 中存在的 Volume，然后执行 detach 操作。在执行 detach 操作之前，会先调用本函数，将 Volume 从 ActualStateOfWorld 内部标记为 attach 状态的列表中移除。

- `AddVolumeToReportAsAttached()`

  当执行完标准的 detach 之后如果失败了则会调用此函数，由于 detach 动作是由 Volume OperationExecutor 完成的，因此这些代码也位于 Volume OperationExecutor 中，具体而言是在 `GenerateDetachVolumeFunc()` 中。这个函数会重新将 detach 失败的 Volume 在 ActualStateOfWorld 中标记为已经 attach 的状态。

### ActualStateOfWorld interface ###

ActualStateOfWorld interface包含的函数主要用于 AttachDetach Controller 中。

``` go
type ActualStateOfWorld interface {
	AddVolumeNode(uniqueName v1.UniqueVolumeName, volumeSpec *volume.Spec, nodeName types.NodeName, devicePath string, attached bool) (v1.UniqueVolumeName, error)
	SetVolumeMountedByNode(volumeName v1.UniqueVolumeName, nodeName types.NodeName, mounted bool) error
	SetNodeStatusUpdateNeeded(nodeName types.NodeName)
	DeleteVolumeNode(volumeName v1.UniqueVolumeName, nodeName types.NodeName)
    ...
}
```

- `AddVolumeNode()`

  `AddVolumeNode(uniqueName v1.UniqueVolumeName, volumeSpec *volume.Spec, nodeName types.NodeName, devicePath string, attached bool) (v1.UniqueVolumeName, error)`

  如果 `attached` 参数为 true，则将 Volume 和对应的 Node 信息添加到 ActualStateOfWorld 缓存中。否则将其从缓存中删除。

  这个函数作为上文中的 `MarkVolumeAsAttached()` 和 `MarkVolumeAsUncertain()` 的具体实现，使用场景请参见 [ActualStateOfWorldAttacherUpdater interface](#ActualStateOfWorldAttacherUpdater-interface)。

- `DeleteVolumeNode()`

  将 Volume 和对应的 Node 信息从 ActualStateOfWorld 缓存中删除。

  这个函数作为上文中的 `MarkVolumeAsDetached()` 的具体实现，使用场景请参见 [ActualStateOfWorldAttacherUpdater interface](#ActualStateOfWorldAttacherUpdater-interface)。

- `SetVolumeMountedByNode()`

  `SetVolumeMountedByNode(volumeName v1.UniqueVolumeName, nodeName types.NodeName, mounted bool) error`

  如果 `mounted` 参数为 true，则会把 Volume 标记为 mount 到 Node 上，此时如果执行 umount 操作会被视作不安全的。
  
  这个函数的参数来源于 Node 的 `Status.volumesInUse` 字段，详见 [AttachDetach Controller](overview.md)。

  这个函数被用于 `processVolumesInUse()` 中，会用于 `processVolumesInUse()` 中。主要用于三个地方：

  1. AttachDetach Controller 运行后填充 ActualStateOfWorld 的时候。
  2. 当 watch 到 Node Update 的事件后，执行相应的回调函数的时候。
  3. 当 watch 到 Node Delete 的事件后，执行相应的回调函数的时候。

