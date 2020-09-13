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

需要特别说明的是最后一个，`UnmountDevice()` 是指将存储设备从节点上的全局挂载点执行 umount 操作。关于全局挂载点的说明，可参见 [关于 mount 和 umount 操作](mount-umount.md)。还有一点就是为什么这里只有 `UnmountDevice()` 操作，而没有对应的 `MountDevice()`，这是因为将存储设备 mount 到节点上的全局挂载点这个操作是在 `MountVolume()` 中实现的。

除了这几个核心函数之外，还有一些辅助的非核心函数，具体逻辑在这里就不再继续展开了。下面对这几个核心函数进行详细的分析。

## AttachVolume() ##

``` go
func (oe *operationExecutor) AttachVolume(
	volumeToAttach VolumeToAttach,
	actualStateOfWorld ActualStateOfWorldAttacherUpdater) error {
	generatedOperations :=
		oe.operationGenerator.GenerateAttachVolumeFunc(volumeToAttach, actualStateOfWorld)

	if util.IsMultiAttachAllowed(volumeToAttach.VolumeSpec) {
		return oe.pendingOperations.Run(
			volumeToAttach.VolumeName, "" /* podName */, volumeToAttach.NodeName, generatedOperations)
	}

	return oe.pendingOperations.Run(
		volumeToAttach.VolumeName, "" /* podName */, "" /* nodeName */, generatedOperations)
}
```

`operationExecutor.AttachVolume()`使用 `operationGenerator.GenerateAttachVolumeFunc()` 获取具体的 attach 函数，然后看是否允许当前 Volume 同时 attach 到多个节点上，按不同的参数来对 `generatedOperations` 进行调用。重点在于 `operationGenerator.GenerateAttachVolumeFunc()` 返回的 attach 函数。下面来分析 `GenerateAttachVolumeFunc()`。

``` go
func (og *operationGenerator) GenerateAttachVolumeFunc(
	volumeToAttach VolumeToAttach,
	actualStateOfWorld ActualStateOfWorldAttacherUpdater) volumetypes.GeneratedOperations {

	attachVolumeFunc := func() (error, error) {
		attachableVolumePlugin, err :=
			og.volumePluginMgr.FindAttachablePluginBySpec(volumeToAttach.VolumeSpec)
		if err != nil || attachableVolumePlugin == nil {
			return volumeToAttach.GenerateError("AttachVolume.FindAttachablePluginBySpec failed", err)
		}

		volumeAttacher, newAttacherErr := attachableVolumePlugin.NewAttacher()
		if newAttacherErr != nil {
			return volumeToAttach.GenerateError("AttachVolume.NewAttacher failed", newAttacherErr)
		}

		// Execute attach
		devicePath, attachErr := volumeAttacher.Attach(
			volumeToAttach.VolumeSpec, volumeToAttach.NodeName)
```

首先，会使用 Volume Plugin 获取当前 Volume 对应的 Plugin 对象，再通过这个 Plugin 获取 Attacher 对象，接着执行 `Attacher.Attach()`，这个函数由 Plugin 自己定义。

``` go
		if attachErr != nil {
			uncertainNode := volumeToAttach.NodeName
			if derr, ok := attachErr.(*volerr.DanglingAttachError); ok {
				uncertainNode = derr.CurrentNode
			}
			addErr := actualStateOfWorld.MarkVolumeAsUncertain(
				v1.UniqueVolumeName(""),
				volumeToAttach.VolumeSpec,
				uncertainNode)
			if addErr != nil {
				klog.Errorf("AttachVolume.MarkVolumeAsUncertain fail to add the volume %q to actual state with %s", volumeToAttach.VolumeName, addErr)
			}

			// On failure, return error. Caller will log and retry.
			return volumeToAttach.GenerateError("AttachVolume.Attach failed", attachErr)
		}
```

接着，如果 `Attacher.Attach()` 执行失败，则调用 `actualStateOfWorld.MarkVolumeAsUncertain()` 更新 Volume 的状态。

``` go
		// Successful attach event is useful for user debugging
		simpleMsg, _ := volumeToAttach.GenerateMsg("AttachVolume.Attach succeeded", "")
		for _, pod := range volumeToAttach.ScheduledPods {
			og.recorder.Eventf(pod, v1.EventTypeNormal, kevents.SuccessfulAttachVolume, simpleMsg)
		}
		klog.Infof(volumeToAttach.GenerateMsgDetailed("AttachVolume.Attach succeeded", ""))

		// Update actual state of world
		addVolumeNodeErr := actualStateOfWorld.MarkVolumeAsAttached(
			v1.UniqueVolumeName(""), volumeToAttach.VolumeSpec, volumeToAttach.NodeName, devicePath)
		if addVolumeNodeErr != nil {
			// On failure, return error. Caller will log and retry.
			return volumeToAttach.GenerateError("AttachVolume.MarkVolumeAsAttached failed", addVolumeNodeErr)
		}

		return nil, nil
	}
```

如果执行成功，调用 `actualStateOfWorld.MarkVolumeAsAttached()` 更新 Volume 的状态。

```
    ...
	return volumetypes.GeneratedOperations{
		OperationName:     "volume_attach",
		OperationFunc:     attachVolumeFunc,
		EventRecorderFunc: eventRecorderFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(attachableVolumePluginName, volumeToAttach.VolumeSpec), "volume_attach"),
	}
}
```

最后返回包含 attachVolumeFunc 的 GeneratedOperations 对象。

## DetachVolume() ##

``` go
func (oe *operationExecutor) DetachVolume(
	volumeToDetach AttachedVolume,
	verifySafeToDetach bool,
	actualStateOfWorld ActualStateOfWorldAttacherUpdater) error {
	generatedOperations, err :=
		oe.operationGenerator.GenerateDetachVolumeFunc(volumeToDetach, verifySafeToDetach, actualStateOfWorld)
	if err != nil {
		return err
	}

	if util.IsMultiAttachAllowed(volumeToDetach.VolumeSpec) {
		return oe.pendingOperations.Run(
			volumeToDetach.VolumeName, "" /* podName */, volumeToDetach.NodeName, generatedOperations)
	}
	return oe.pendingOperations.Run(
		volumeToDetach.VolumeName, "" /* podName */, "" /* nodeName */, generatedOperations)

}
```

`operationExecutor.DetachVolume()`使用 `operationGenerator.GenerateDetachVolumeFunc()` 获取具体的 detach 函数，然后看是否允许当前 Volume 同时 attach 到多个节点上，按不同的参数来对 `generatedOperations` 进行调用。重点在于 `operationGenerator.GenerateDetachVolumeFunc()` 返回的 detach 函数。下面来分析 `GenerateDetachVolumeFunc()`。

``` go
func (og *operationGenerator) GenerateDetachVolumeFunc(
	volumeToDetach AttachedVolume,
	verifySafeToDetach bool,
	actualStateOfWorld ActualStateOfWorldAttacherUpdater) (volumetypes.GeneratedOperations, error) {
	var volumeName string
	var attachableVolumePlugin volume.AttachableVolumePlugin
	var pluginName string
	var err error

	if volumeToDetach.VolumeSpec != nil {
		attachableVolumePlugin, err =
			og.volumePluginMgr.FindAttachablePluginBySpec(volumeToDetach.VolumeSpec)
		if err != nil || attachableVolumePlugin == nil {
			return volumetypes.GeneratedOperations{}, volumeToDetach.GenerateErrorDetailed("DetachVolume.FindAttachablePluginBySpec failed", err)
		}

		volumeName, err =
			attachableVolumePlugin.GetVolumeName(volumeToDetach.VolumeSpec)
		if err != nil {
			return volumetypes.GeneratedOperations{}, volumeToDetach.GenerateErrorDetailed("DetachVolume.GetVolumeName failed", err)
		}
	} else {
		// Get attacher plugin and the volumeName by splitting the volume unique name in case
		// there's no VolumeSpec: this happens only on attach/detach controller crash recovery
		// when a pod has been deleted during the controller downtime
		pluginName, volumeName, err = util.SplitUniqueName(volumeToDetach.VolumeName)
		if err != nil {
			return volumetypes.GeneratedOperations{}, volumeToDetach.GenerateErrorDetailed("DetachVolume.SplitUniqueName failed", err)
		}

		attachableVolumePlugin, err = og.volumePluginMgr.FindAttachablePluginByName(pluginName)
		if err != nil || attachableVolumePlugin == nil {
			return volumetypes.GeneratedOperations{}, volumeToDetach.GenerateErrorDetailed("DetachVolume.FindAttachablePluginByName failed", err)
		}

	}
```

首先根据 Volume 的 AttachedVolume.VolumeSpec 获取当前 Volume 的 Volume Plugin 对象。注意如果 attach/detach Controller 处于 down 的状态，此时一个带有 Volume 的 Pod 被删除，attach/detach Controller 重新恢复 up 状态并开始工作的时候，会尝试执行 Volume 的 detach 操作，这种情况下 AttachedVolume.VolumeSpec 为空，此时就需要根据 Plugin Name 来获取 Volume Plugin 对象。

``` go
	if pluginName == "" {
		pluginName = attachableVolumePlugin.GetPluginName()
	}

	volumeDetacher, err := attachableVolumePlugin.NewDetacher()
	if err != nil {
		return volumetypes.GeneratedOperations{}, volumeToDetach.GenerateErrorDetailed("DetachVolume.NewDetacher failed", err)
	}

	getVolumePluginMgrFunc := func() (error, error) {
		var err error
		if verifySafeToDetach {
			err = og.verifyVolumeIsSafeToDetach(volumeToDetach)
		}
		if err == nil {
			err = volumeDetacher.Detach(volumeName, volumeToDetach.NodeName)
		}
```

接着通过 Volume Plugin 获取 Detacher。这时定义 getVolumePluginMgrFunc 作为返回的函数，在其中执行 `Detacher.Detach()`，这个函数需要由各个 Volume Plugin 自己实现。

``` go
		if err != nil {
			// On failure, add volume back to ReportAsAttached list
			actualStateOfWorld.AddVolumeToReportAsAttached(
				volumeToDetach.VolumeName, volumeToDetach.NodeName)
			return volumeToDetach.GenerateError("DetachVolume.Detach failed", err)
		}

		klog.Infof(volumeToDetach.GenerateMsgDetailed("DetachVolume.Detach succeeded", ""))

		// Update actual state of world
		actualStateOfWorld.MarkVolumeAsDetached(
			volumeToDetach.VolumeName, volumeToDetach.NodeName)

		return nil, nil
	}

	return volumetypes.GeneratedOperations{
		OperationName:     "volume_detach",
		OperationFunc:     getVolumePluginMgrFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(pluginName, volumeToDetach.VolumeSpec), "volume_detach"),
		EventRecorderFunc: nil, // nil because we do not want to generate event on error
	}, nil
}
```

如果 `Detacher.Detach()` 执行成功，则使用 `actualStateOfWorld.MarkVolumeAsDetached()` 更新状态，否则使用 `actualStateOfWorld.AddVolumeToReportAsAttached()` 更新状态。

最后返回 GeneratedOperations 对象，里面包含了刚才定义的 getVolumePluginMgrFunc 函数。

## MountVolume() ##

``` go
func (oe *operationExecutor) MountVolume(
	waitForAttachTimeout time.Duration,
	volumeToMount VolumeToMount,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	isRemount bool) error {
	fsVolume, err := util.CheckVolumeModeFilesystem(volumeToMount.VolumeSpec)
	if err != nil {
		return err
	}
	var generatedOperations volumetypes.GeneratedOperations
	if fsVolume {
		// Filesystem volume case
		// Mount/remount a volume when a volume is attached
		generatedOperations = oe.operationGenerator.GenerateMountVolumeFunc(
			waitForAttachTimeout, volumeToMount, actualStateOfWorld, isRemount)

	} else {
		// Block volume case
		// Creates a map to device if a volume is attached
		generatedOperations, err = oe.operationGenerator.GenerateMapVolumeFunc(
			waitForAttachTimeout, volumeToMount, actualStateOfWorld)
	}
    ...
	return oe.pendingOperations.Run(
		volumeToMount.VolumeName, podName, "" /* nodeName */, generatedOperations)
}
```

`MountVolume()` 根据底层存储作为文件系统还是块设备提供给 Pod 使用，分为了两种情况。分类的依据是 `volumeSpec.PersistentVolume.Spec.VolumeMode` 为 "Block" 的时候作为块设备，其它情况都作为文件系统。根据这两种情况，分别调用 `GenerateMountVolumeFunc()` 和 `GenerateMapVolumeFunc()` 生成对应的函数。下面对这两种情况分别进行分析。

### GenerateMountVolumeFunc() ###

``` go
func (og *operationGenerator) GenerateMountVolumeFunc(
	waitForAttachTimeout time.Duration,
	volumeToMount VolumeToMount,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	isRemount bool) volumetypes.GeneratedOperations {
    ...
	mountVolumeFunc := func() (error, error) {
		// Get mounter plugin
		volumePlugin, err := og.volumePluginMgr.FindPluginBySpec(volumeToMount.VolumeSpec)
		if err != nil || volumePlugin == nil {
			return volumeToMount.GenerateError("MountVolume.FindPluginBySpec failed", err)
		}
```

创建需要返回的 mountVolumeFunc 函数，首先使用 VolumeSpec 来获取 Volume Plugin 对象

``` go
		affinityErr := checkNodeAffinity(og, volumeToMount)
		if affinityErr != nil {
			return volumeToMount.GenerateError("MountVolume.NodeAffinity check failed", affinityErr)
		}

```

然后对 Node 和 PV 做亲和性检测，注意只针对 `required` 类型的亲和性做检测，也就是 **强亲和性策略**。

``` go
		volumeMounter, newMounterErr := volumePlugin.NewMounter(
			volumeToMount.VolumeSpec,
			volumeToMount.Pod,
			volume.VolumeOptions{})
		if newMounterErr != nil {
			return volumeToMount.GenerateError("MountVolume.NewMounter initialization failed", newMounterErr)

		}

		mountCheckError := checkMountOptionSupport(og, volumeToMount, volumePlugin)

		if mountCheckError != nil {
			return volumeToMount.GenerateError("MountVolume.MountOptionSupport check failed", mountCheckError)
		}

```

通过 Volume Plugin 来获取 Mounter 对象，接着检测当前 Volume Plugin 是否支持 mount 选项的功能，具体会通过 Volume Plugin 的 `SupportsMountOption()` 来进行。注意这里的这个检测函数 `SupportsMountOption()` 并没有传入具体的 mount option 参数，整个检测逻辑也仅仅是判断 Volume Plugin 是否支持，而并非检测是否支持具体的某一个 mount option。

``` go
		// Get attacher, if possible
		attachableVolumePlugin, _ :=
			og.volumePluginMgr.FindAttachablePluginBySpec(volumeToMount.VolumeSpec)
		var volumeAttacher volume.Attacher
		if attachableVolumePlugin != nil {
			volumeAttacher, _ = attachableVolumePlugin.NewAttacher()
		}

		// get deviceMounter, if possible
		deviceMountableVolumePlugin, _ := og.volumePluginMgr.FindDeviceMountablePluginBySpec(volumeToMount.VolumeSpec)
		var volumeDeviceMounter volume.DeviceMounter
		if deviceMountableVolumePlugin != nil {
			volumeDeviceMounter, _ = deviceMountableVolumePlugin.NewDeviceMounter()
		}
```

然后通过 VolumeSpec 获取 Attacher 对象和 DeviceMounter 对象，后者负责把存储挂载到节点上的全局挂载点。

``` go
        ...
		devicePath := volumeToMount.DevicePath
		if volumeAttacher != nil {
			// Wait for attachable volumes to finish attaching
			klog.Infof(volumeToMount.GenerateMsgDetailed("MountVolume.WaitForAttach entering", fmt.Sprintf("DevicePath %q", volumeToMount.DevicePath)))

			devicePath, err = volumeAttacher.WaitForAttach(
				volumeToMount.VolumeSpec, devicePath, volumeToMount.Pod, waitForAttachTimeout)
			if err != nil {
				// On failure, return error. Caller will log and retry.
				return volumeToMount.GenerateError("MountVolume.WaitForAttach failed", err)
			}

			klog.Infof(volumeToMount.GenerateMsgDetailed("MountVolume.WaitForAttach succeeded", fmt.Sprintf("DevicePath %q", devicePath)))
		}
```

接着会尝试使用 Attacher 的 `WaitForAttach()` 来获取存储在节点上的具体设备文件路径，例如 `/dev/sda` 。

这里有一个非常重要的点，就是 Attacher 对象的代码和调用时机，Attacher 对象顾名思义，就是将外部存储 attach 到 Pod 所在节点上或者执行反向的 detach 操作。因此对应于 [Kubernetes 中的 Volume 概述](overview.md) 中的第二阶段，并且代码位于 Kube Controller Manager 中。其实这并不完全正确，只是为了不涉及太多细节而进行的简化叙述。实际上 Attacher 代码也必须同时存在于 Kubelet 中，因为 Attacher 的 `WaitForAttach()` 是在 Kubelet 执行 mount 操作的时候进行调用的。

这个也从另外一个方面解释了为什么 Attacher 对象可以使用 `WaitForAttach()` 返回底层存储设备在节点上的实际路径，如果 Attacher 代码只存在于 Kube Controller Manager 中的，它是无论如何也没有办法到某一个节点上执行对应操作的。这个问题的根本原因在于调用 `WaitForAttach()` 的时机，就是现在，在 Kubelet 中执行 mount 操作的时候，因此 Kubelet 中的 Attacher 知道如何通过 Kubelet 与当前节点上的文件系统等底层资源交互从而获取对应的信息。

现在回到 `GenerateMountVolumeFunc()`。

``` go
		var resizeDone bool
		var resizeError error
		resizeOptions := volume.NodeResizeOptions{
			DevicePath: devicePath,
		}

		if volumeDeviceMounter != nil {
			deviceMountPath, err :=
				volumeDeviceMounter.GetDeviceMountPath(volumeToMount.VolumeSpec)
			if err != nil {
				// On failure, return error. Caller will log and retry.
				return volumeToMount.GenerateError("MountVolume.GetDeviceMountPath failed", err)
			}

			// Mount device to global mount path
			err = volumeDeviceMounter.MountDevice(
				volumeToMount.VolumeSpec,
				devicePath,
				deviceMountPath)
			if err != nil {
				og.checkForFailedMount(volumeToMount, err)
				og.markDeviceErrorState(volumeToMount, devicePath, deviceMountPath, err, actualStateOfWorld)
				// On failure, return error. Caller will log and retry.
				return volumeToMount.GenerateError("MountVolume.MountDevice failed", err)
			}

			klog.Infof(volumeToMount.GenerateMsgDetailed("MountVolume.MountDevice succeeded", fmt.Sprintf("device mount path %q", deviceMountPath)))

			// Update actual state of world to reflect volume is globally mounted
			markDeviceMountedErr := actualStateOfWorld.MarkDeviceAsMounted(
				volumeToMount.VolumeName, devicePath, deviceMountPath)
			if markDeviceMountedErr != nil {
				// On failure, return error. Caller will log and retry.
				return volumeToMount.GenerateError("MountVolume.MarkDeviceAsMounted failed", markDeviceMountedErr)
			}

			resizeOptions.DeviceMountPath = deviceMountPath
			resizeOptions.CSIVolumePhase = volume.CSIVolumeStaged

			// NodeExpandVolume will resize the file system if user has requested a resize of
			// underlying persistent volume and is allowed to do so.
			resizeDone, resizeError = og.nodeExpandVolume(volumeToMount, resizeOptions)

			if resizeError != nil {
				klog.Errorf("MountVolume.NodeExpandVolume failed with %v", resizeError)
				return volumeToMount.GenerateError("MountVolume.MountDevice failed while expanding volume", resizeError)
			}
		}
```

接着，如果 DeviceMounter 对象存在的话，会使用它的 `MountDevice()` 将底层存储挂载到全局挂载点。执行成功后使用 `actualStateOfWorld.MarkDeviceAsMounted()` 来更新 actualStateOfWorld 的状态。这时，如果发现磁盘需要扩容，就执行 `nodeExpandVolume()` 对其进行扩容操作。具体是使用 NodeExpandableVolumePlugin 对象的 `NodeExpand()` 进行，具体代码不在这里展开。

```
		if og.checkNodeCapabilitiesBeforeMount {
			if canMountErr := volumeMounter.CanMount(); canMountErr != nil {
				err = fmt.Errorf(
					"Verify that your node machine has the required components before attempting to mount this volume type. %s",
					canMountErr)
				return volumeToMount.GenerateError("MountVolume.CanMount failed", err)
			}
		}

		// Execute mount
		mountErr := volumeMounter.SetUp(volume.MounterArgs{
			FsGroup:             fsGroup,
			DesiredSize:         volumeToMount.DesiredSizeLimit,
			FSGroupChangePolicy: fsGroupChangePolicy,
		})
```

然后使用 Mounter 对象的 `CanMount()` 来做一些预检测，判断是否可以执行 mount 操作，如果一切顺利，执行 Mounter 对象的 `Setup()` 进行 mount，这个函数由 Volume Plugin 自己进行定义，内部执行的操作一般是将全局挂载点以 bind 模式挂载到 Pod 关联的 Volume 的挂载点上。

``` go

        ...
		// We need to call resizing here again in case resizing was not done during device mount. There could be
		// two reasons of that:
		//	- Volume does not support DeviceMounter interface.
		//	- In case of CSI the volume does not have node stage_unstage capability.
		if !resizeDone {
			_, resizeError = og.nodeExpandVolume(volumeToMount, resizeOptions)
			if resizeError != nil {
				klog.Errorf("MountVolume.NodeExpandVolume failed with %v", resizeError)
				return volumeToMount.GenerateError("MountVolume.Setup failed while expanding volume", resizeError)
			}
		}
```

接着再次检测是否需要扩容，如果是的话则进行扩容操作。注意刚才的扩容操作是针对将存储设备挂载到全局挂载点的，有很多 Volume Plugin 实际上是没有全局挂载点的概念，也就不会执行相应的操作。因此真正的扩容就需要在这里执行。

```
		markVolMountedErr := actualStateOfWorld.MarkVolumeAsMounted(markOpts)
		if markVolMountedErr != nil {
			// On failure, return error. Caller will log and retry.
			return volumeToMount.GenerateError("MountVolume.MarkVolumeAsMounted failed", markVolMountedErr)
		}

		return nil, nil
	}

	eventRecorderFunc := func(err *error) {
		if *err != nil {
			og.recorder.Eventf(volumeToMount.Pod, v1.EventTypeWarning, kevents.FailedMountVolume, (*err).Error())
		}
	}

	return volumetypes.GeneratedOperations{
		OperationName:     "volume_mount",
		OperationFunc:     mountVolumeFunc,
		EventRecorderFunc: eventRecorderFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(volumePluginName, volumeToMount.VolumeSpec), "volume_mount"),
	}
}
```

全部执行成功后，使用 `actualStateOfWorld.MarkVolumeAsMounted()` 更新 actualStateOfWorld 的状态，并返回。

### GenerateMapVolumeFunc() ###

当 PV 作为块设备使用时，会调用 `GenerateMapVolumeFunc()` 进行相关的 mount 操作。由于块设备不同于文件系统，因此其 mount 操作又被称为 map。

``` go
func (og *operationGenerator) GenerateMapVolumeFunc(
	waitForAttachTimeout time.Duration,
	volumeToMount VolumeToMount,
	actualStateOfWorld ActualStateOfWorldMounterUpdater) (volumetypes.GeneratedOperations, error) {

	// Get block volume mapper plugin
	blockVolumePlugin, err :=
		og.volumePluginMgr.FindMapperPluginBySpec(volumeToMount.VolumeSpec)
	if err != nil {
		return volumetypes.GeneratedOperations{}, volumeToMount.GenerateErrorDetailed("MapVolume.FindMapperPluginBySpec failed", err)
	}
```

首先，根据 VolumeSpec 找到对应的 Volume Plugin 对象。

``` go
	if blockVolumePlugin == nil {
		return volumetypes.GeneratedOperations{}, volumeToMount.GenerateErrorDetailed("MapVolume.FindMapperPluginBySpec failed to find BlockVolumeMapper plugin. Volume plugin is nil.", nil)
	}

	affinityErr := checkNodeAffinity(og, volumeToMount)
	if affinityErr != nil {
		eventErr, detailedErr := volumeToMount.GenerateError("MapVolume.NodeAffinity check failed", affinityErr)
		og.recorder.Eventf(volumeToMount.Pod, v1.EventTypeWarning, kevents.FailedMountVolume, eventErr.Error())
		return volumetypes.GeneratedOperations{}, detailedErr
	}
```

然后对 Volume 和节点做亲和性检测，检测逻辑与 `GenerateMountVolumeFunc()` 中的检测逻辑相同。

``` go
	blockVolumeMapper, newMapperErr := blockVolumePlugin.NewBlockVolumeMapper(
		volumeToMount.VolumeSpec,
		volumeToMount.Pod,
		volume.VolumeOptions{})
	if newMapperErr != nil {
		eventErr, detailedErr := volumeToMount.GenerateError("MapVolume.NewBlockVolumeMapper initialization failed", newMapperErr)
		og.recorder.Eventf(volumeToMount.Pod, v1.EventTypeWarning, kevents.FailedMapVolume, eventErr.Error())
		return volumetypes.GeneratedOperations{}, detailedErr
	}

	// Get attacher, if possible
	attachableVolumePlugin, _ :=
		og.volumePluginMgr.FindAttachablePluginBySpec(volumeToMount.VolumeSpec)
	var volumeAttacher volume.Attacher
	if attachableVolumePlugin != nil {
		volumeAttacher, _ = attachableVolumePlugin.NewAttacher()
	}
```

创建 VolumeMapper 对象，根据 VolumeSpec 获取 AttachableVolumePlugin 对象，进而获取 attacher 对象。

``` go
	mapVolumeFunc := func() (simpleErr error, detailedErr error) {
		var devicePath string
		// Set up global map path under the given plugin directory using symbolic link
		globalMapPath, err :=
			blockVolumeMapper.GetGlobalMapPath(volumeToMount.VolumeSpec)
		if err != nil {
			// On failure, return error. Caller will log and retry.
			return volumeToMount.GenerateError("MapVolume.GetGlobalMapPath failed", err)
		}
		if volumeAttacher != nil {
			// Wait for attachable volumes to finish attaching
			klog.Infof(volumeToMount.GenerateMsgDetailed("MapVolume.WaitForAttach entering", fmt.Sprintf("DevicePath %q", volumeToMount.DevicePath)))

			devicePath, err = volumeAttacher.WaitForAttach(
				volumeToMount.VolumeSpec, volumeToMount.DevicePath, volumeToMount.Pod, waitForAttachTimeout)
			if err != nil {
				// On failure, return error. Caller will log and retry.
				return volumeToMount.GenerateError("MapVolume.WaitForAttach failed", err)
			}

			klog.Infof(volumeToMount.GenerateMsgDetailed("MapVolume.WaitForAttach succeeded", fmt.Sprintf("DevicePath %q", devicePath)))

		}
```

接下来开始返回函数 mapVolumeFunc 的定义，在其中，首先或者全局挂载点的路径。如果 attacher 对象非空的话，则调用它的 `WaitForAttach()` 获取设备的路径。

``` go
		// Call SetUpDevice if blockVolumeMapper implements CustomBlockVolumeMapper
		if customBlockVolumeMapper, ok := blockVolumeMapper.(volume.CustomBlockVolumeMapper); ok {
			mapErr := customBlockVolumeMapper.SetUpDevice()
			if mapErr != nil {
				og.markDeviceErrorState(volumeToMount, devicePath, globalMapPath, mapErr, actualStateOfWorld)
				// On failure, return error. Caller will log and retry.
				return volumeToMount.GenerateError("MapVolume.SetUpDevice failed", mapErr)
			}
		}
```

如果 BlockVolumeMapper 对象实现了 CustomBlockVolumeMapper interface 的话，则调用它的 `SetUpDevice()`，这个主要作为 CSI 的一个 hook 点，详见 CSI 部分。

``` go
		// Update actual state of world to reflect volume is globally mounted
		markedDevicePath := devicePath
		markDeviceMappedErr := actualStateOfWorld.MarkDeviceAsMounted(
			volumeToMount.VolumeName, markedDevicePath, globalMapPath)
		if markDeviceMappedErr != nil {
			// On failure, return error. Caller will log and retry.
			return volumeToMount.GenerateError("MapVolume.MarkDeviceAsMounted failed", markDeviceMappedErr)
		}
```

接下来，使用 `actualStateOfWorld.MarkDeviceAsMounted()` 将设备标记为全局挂载状态。

> 为什么在这个时间点就可以认为存储设备完成了全局挂载？推测原因是这样的，对于 in-tree 类型的 Volume Plugin，一般会在 `volumeAttacher.WaitForAttach()` 中便完成了全局挂载目录的创建等准备工作。而接下来的 `ioutil.MapBlockVolume()` 则会中将存储设备以 bind 模式 mount 这个全局挂载目录下面的子文件上，然后创建从存储设备到 Pod 上局部挂载点的符号链接。

``` go
		markVolumeOpts := MarkVolumeOpts{
			PodName:             volumeToMount.PodName,
			PodUID:              volumeToMount.Pod.UID,
			VolumeName:          volumeToMount.VolumeName,
			BlockVolumeMapper:   blockVolumeMapper,
			OuterVolumeSpecName: volumeToMount.OuterVolumeSpecName,
			VolumeGidVolume:     volumeToMount.VolumeGidValue,
			VolumeSpec:          volumeToMount.VolumeSpec,
			VolumeMountState:    VolumeMounted,
		}

		// Call MapPodDevice if blockVolumeMapper implements CustomBlockVolumeMapper
		if customBlockVolumeMapper, ok := blockVolumeMapper.(volume.CustomBlockVolumeMapper); ok {
			// Execute driver specific map
			pluginDevicePath, mapErr := customBlockVolumeMapper.MapPodDevice()
            。。。

			// if pluginDevicePath is provided, assume attacher may not provide device
			// or attachment flow uses SetupDevice to get device path
			if len(pluginDevicePath) != 0 {
				devicePath = pluginDevicePath
			}
            ...
		}
```

这段代码会检测 BlockVolumeMapper 是否实现了 CustomBlockVolumeMapper interface，如果是的话则调用它的 `MapPodDevice()` 将存储块设备映射到 Pod 中 。

``` go
		// When kubelet is containerized, devicePath may be a symlink at a place unavailable to
		// kubelet, so evaluate it on the host and expect that it links to a device in /dev,
		// which will be available to containerized kubelet. If still it does not exist,
		// AttachFileDevice will fail. If kubelet is not containerized, eval it anyway.
		kvh, ok := og.GetVolumePluginMgr().Host.(volume.KubeletVolumeHost)
		if !ok {
			return volumeToMount.GenerateError("MapVolume type assertion error", fmt.Errorf("volume host does not implement KubeletVolumeHost interface"))
		}
		hu := kvh.GetHostUtil()
		devicePath, err = hu.EvalHostSymlinks(devicePath)
		if err != nil {
			return volumeToMount.GenerateError("MapVolume.EvalHostSymlinks failed", err)
		}

		// Update actual state of world with the devicePath again, if devicePath has changed from markedDevicePath
		// TODO: This can be improved after #82492 is merged and ASW has state.
		if markedDevicePath != devicePath {
			markDeviceMappedErr := actualStateOfWorld.MarkDeviceAsMounted(
				volumeToMount.VolumeName, devicePath, globalMapPath)
			if markDeviceMappedErr != nil {
				// On failure, return error. Caller will log and retry.
				return volumeToMount.GenerateError("MapVolume.MarkDeviceAsMounted failed", markDeviceMappedErr)
			}
		}
```

在执行 mount 操作时，Kubelet 可能会以容器的方式在运行，因此需要获取底层存储设备在 host 上的真正路径，这个路径如果与 `Attacher.WaitForAttach()` 返回的不一致，则以前者为准，然后使用 `MarkDeviceAsMounted()` 更新 actualStateOfWorld 的状态。

``` go
		// Execute common map
		volumeMapPath, volName := blockVolumeMapper.GetPodDeviceMapPath()
		mapErr := ioutil.MapBlockVolume(og.blkUtil, devicePath, globalMapPath, volumeMapPath, volName, volumeToMount.Pod.UID)
		if mapErr != nil {
			// On failure, return error. Caller will log and retry.
			return volumeToMount.GenerateError("MapVolume.MapBlockVolume failed", mapErr)
		}

		// Device mapping for global map path succeeded
		simpleMsg, detailedMsg := volumeToMount.GenerateMsg("MapVolume.MapPodDevice succeeded", fmt.Sprintf("globalMapPath %q", globalMapPath))
		verbosity := klog.Level(4)
		og.recorder.Eventf(volumeToMount.Pod, v1.EventTypeNormal, kevents.SuccessfulMountVolume, simpleMsg)
		klog.V(verbosity).Infof(detailedMsg)

		// Device mapping for pod device map path succeeded
		simpleMsg, detailedMsg = volumeToMount.GenerateMsg("MapVolume.MapPodDevice succeeded", fmt.Sprintf("volumeMapPath %q", volumeMapPath))
		verbosity = klog.Level(1)
		og.recorder.Eventf(volumeToMount.Pod, v1.EventTypeNormal, kevents.SuccessfulMountVolume, simpleMsg)
		klog.V(verbosity).Infof(detailedMsg)
```

这里会通过 `ioutil.MapBlockVolume()` 真正执行存储设备的 mount 操作，如果执行成功，生成相应的事件。

而在 `ioutil.MapBlockVolume()` 内部则会执行两次 mount 操作。

``` go
func MapBlockVolume(
	blkUtil volumepathhandler.BlockVolumePathHandler,
	devicePath,
	globalMapPath,
	podVolumeMapPath,
	volumeMapName string,
	podUID utypes.UID,
) error {
	// map devicePath to global node path as bind mount
	mapErr := blkUtil.MapDevice(devicePath, globalMapPath, string(podUID), true /* bindMount */)
	if mapErr != nil {
		return fmt.Errorf("blkUtil.MapDevice failed. devicePath: %s, globalMapPath:%s, podUID: %s, bindMount: %v: %v",
			devicePath, globalMapPath, string(podUID), true, mapErr)
	}

	// map devicePath to pod volume path
	mapErr = blkUtil.MapDevice(devicePath, podVolumeMapPath, volumeMapName, false /* bindMount */)
	if mapErr != nil {
		return fmt.Errorf("blkUtil.MapDevice failed. devicePath: %s, podVolumeMapPath:%s, volumeMapName: %s, bindMount: %v: %v",
			devicePath, podVolumeMapPath, volumeMapName, false, mapErr)
	}

	// Take file descriptor lock to keep a block device opened. Otherwise, there is a case
	// that the block device is silently removed and attached another device with the same name.
	// Container runtime can't handle this problem. To avoid unexpected condition fd lock
	// for the block device is required.
	_, mapErr = blkUtil.AttachFileDevice(filepath.Join(globalMapPath, string(podUID)))
	if mapErr != nil {
		return fmt.Errorf("blkUtil.AttachFileDevice failed. globalMapPath:%s, podUID: %s: %v",
			globalMapPath, string(podUID), mapErr)
	}

	return nil
}
```

第一次会将存储设备以 bind 模式 mount 到全局挂载点目录下的一个文件，这个文件以当前 Pod UID 命名。第二次会将设备以符号链接的形式链接到 Pod 对应的 Volume 路径上。

``` go
		resizeOptions := volume.NodeResizeOptions{
			DevicePath:     devicePath,
			CSIVolumePhase: volume.CSIVolumePublished,
		}
		_, resizeError := og.nodeExpandVolume(volumeToMount, resizeOptions)
		if resizeError != nil {
			klog.Errorf("MapVolume.NodeExpandVolume failed with %v", resizeError)
			return volumeToMount.GenerateError("MapVolume.MarkVolumeAsMounted failed while expanding volume", resizeError)
		}

		markVolMountedErr := actualStateOfWorld.MarkVolumeAsMounted(markVolumeOpts)
		if markVolMountedErr != nil {
			// On failure, return error. Caller will log and retry.
			return volumeToMount.GenerateError("MapVolume.MarkVolumeAsMounted failed", markVolMountedErr)
		}

		return nil, nil
	}
```

所有 mount 操作全部执行完成之后，会检测是否需要做 expand 操作，如果需要的话则会调用 `nodeExpandVolume()` 执行。最后执行 `MarkVolumeAsMounted()` 更新 actualStateOfWorld。

``` go
	eventRecorderFunc := func(err *error) {
		if *err != nil {
			og.recorder.Eventf(volumeToMount.Pod, v1.EventTypeWarning, kevents.FailedMapVolume, (*err).Error())
		}
	}

	return volumetypes.GeneratedOperations{
		OperationName:     "map_volume",
		OperationFunc:     mapVolumeFunc,
		EventRecorderFunc: eventRecorderFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(blockVolumePlugin.GetPluginName(), volumeToMount.VolumeSpec), "map_volume"),
	}, nil
}
```

整个函数返回 GeneratedOperations 对象，其中包含有刚才的 mapVolumeFunc 回调函数。

## UnmountVolume() ##

`UnmountVolume()` 会将存储设备从 Pod 关联的 Volume 局部挂载点上执行 umount 操作。

``` go
func (oe *operationExecutor) UnmountVolume(
	volumeToUnmount MountedVolume,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	podsDir string) error {
	fsVolume, err := util.CheckVolumeModeFilesystem(volumeToUnmount.VolumeSpec)
	if err != nil {
		return err
	}
	var generatedOperations volumetypes.GeneratedOperations
	if fsVolume {
		// Filesystem volume case
		// Unmount a volume if a volume is mounted
		generatedOperations, err = oe.operationGenerator.GenerateUnmountVolumeFunc(
			volumeToUnmount, actualStateOfWorld, podsDir)
	} else {
		// Block volume case
		// Unmap a volume if a volume is mapped
		generatedOperations, err = oe.operationGenerator.GenerateUnmapVolumeFunc(
			volumeToUnmount, actualStateOfWorld)
	}
	if err != nil {
		return err
	}
	// All volume plugins can execute unmount/unmap for multiple pods referencing the
	// same volume in parallel
	podName := volumetypes.UniquePodName(volumeToUnmount.PodUID)

	return oe.pendingOperations.Run(
		volumeToUnmount.VolumeName, podName, "" /* nodeName */, generatedOperations)
}
```

类似于 `MountVolume()`，这里也会根据存储设备是作为文件系统使用还是作为块设备使用这两种不同的使用方式生成不同的函数：`GenerateUnmountVolumeFunc()` 和 `GenerateUnmapVolumeFunc()`，下面分别进行分析。

### GenerateUnmountVolumeFunc() ###

``` go
func (og *operationGenerator) GenerateUnmountVolumeFunc(
	volumeToUnmount MountedVolume,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	podsDir string) (volumetypes.GeneratedOperations, error) {
	// Get mountable plugin
	volumePlugin, err := og.volumePluginMgr.FindPluginByName(volumeToUnmount.PluginName)
	if err != nil || volumePlugin == nil {
		return volumetypes.GeneratedOperations{}, volumeToUnmount.GenerateErrorDetailed("UnmountVolume.FindPluginByName failed", err)
	}
	volumeUnmounter, newUnmounterErr := volumePlugin.NewUnmounter(
		volumeToUnmount.InnerVolumeSpecName, volumeToUnmount.PodUID)
	if newUnmounterErr != nil {
		return volumetypes.GeneratedOperations{}, volumeToUnmount.GenerateErrorDetailed("UnmountVolume.NewUnmounter failed", newUnmounterErr)
	}
```

首先根据 PluginName 获取 Plugin 对象，进而获取 Umounter 对象。

``` go
	unmountVolumeFunc := func() (error, error) {
		subpather := og.volumePluginMgr.Host.GetSubpather()

		// Remove all bind-mounts for subPaths
		podDir := filepath.Join(podsDir, string(volumeToUnmount.PodUID))
		if err := subpather.CleanSubPaths(podDir, volumeToUnmount.InnerVolumeSpecName); err != nil {
			return volumeToUnmount.GenerateError("error cleaning subPath mounts", err)
		}
```

然后删除所有的 subPath。关于 subPath 可参见 [Using subPath
](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath) 。

```
		// Execute unmount
		unmountErr := volumeUnmounter.TearDown()
		if unmountErr != nil {
			// On failure, return error. Caller will log and retry.
			return volumeToUnmount.GenerateError("UnmountVolume.TearDown failed", unmountErr)
		}

		klog.Infof(
			"UnmountVolume.TearDown succeeded for volume %q (OuterVolumeSpecName: %q) pod %q (UID: %q). InnerVolumeSpecName %q. PluginName %q, VolumeGidValue %q",
			volumeToUnmount.VolumeName,
			volumeToUnmount.OuterVolumeSpecName,
			volumeToUnmount.PodName,
			volumeToUnmount.PodUID,
			volumeToUnmount.InnerVolumeSpecName,
			volumeToUnmount.PluginName,
			volumeToUnmount.VolumeGidValue)

		// Update actual state of world
		markVolMountedErr := actualStateOfWorld.MarkVolumeAsUnmounted(
			volumeToUnmount.PodName, volumeToUnmount.VolumeName)
		if markVolMountedErr != nil {
			// On failure, just log and exit
			klog.Errorf(volumeToUnmount.GenerateErrorDetailed("UnmountVolume.MarkVolumeAsUnmounted failed", markVolMountedErr).Error())
		}

		return nil, nil
	}

	return volumetypes.GeneratedOperations{
		OperationName:     "volume_unmount",
		OperationFunc:     unmountVolumeFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(volumePlugin.GetPluginName(), volumeToUnmount.VolumeSpec), "volume_unmount"),
		EventRecorderFunc: nil, // nil because we do not want to generate event on error
	}, nil
}
```

接着，调用 Umounter 的 `TearDown()` 执行真正的 umount 操作，这个函数由 Volume Plugin 自己实现，内部逻辑是将存储设备在 Pod 关联的 Volume 的路径上执行 umount 操作。执行成功后，使用 `MarkVolumeAsUnmounted()` 更新 actualStateOfWorld，并返回。

### GenerateUnmapVolumeFunc() ###

``` go
func (og *operationGenerator) GenerateUnmapVolumeFunc(
	volumeToUnmount MountedVolume,
	actualStateOfWorld ActualStateOfWorldMounterUpdater) (volumetypes.GeneratedOperations, error) {

	// Get block volume unmapper plugin
	blockVolumePlugin, err :=
		og.volumePluginMgr.FindMapperPluginByName(volumeToUnmount.PluginName)
	if err != nil {
		return volumetypes.GeneratedOperations{}, volumeToUnmount.GenerateErrorDetailed("UnmapVolume.FindMapperPluginByName failed", err)
	}
	if blockVolumePlugin == nil {
		return volumetypes.GeneratedOperations{}, volumeToUnmount.GenerateErrorDetailed("UnmapVolume.FindMapperPluginByName failed to find BlockVolumeMapper plugin. Volume plugin is nil.", nil)
	}
	blockVolumeUnmapper, newUnmapperErr := blockVolumePlugin.NewBlockVolumeUnmapper(
		volumeToUnmount.InnerVolumeSpecName, volumeToUnmount.PodUID)
	if newUnmapperErr != nil {
		return volumetypes.GeneratedOperations{}, volumeToUnmount.GenerateErrorDetailed("UnmapVolume.NewUnmapper failed", newUnmapperErr)
	}
```

首先根据 PluginName 获取 Plugin 对象，进而获取 BlockVolumeUnmapper 对象。

``` go
	unmapVolumeFunc := func() (error, error) {
		// pods/{podUid}/volumeDevices/{escapeQualifiedPluginName}/{volumeName}
		podDeviceUnmapPath, volName := blockVolumeUnmapper.GetPodDeviceMapPath()
		// plugins/kubernetes.io/{PluginName}/volumeDevices/{volumePluginDependentPath}/{podUID}
		globalUnmapPath := volumeToUnmount.DeviceMountPath

		// Execute common unmap
		unmapErr := ioutil.UnmapBlockVolume(og.blkUtil, globalUnmapPath, podDeviceUnmapPath, volName, volumeToUnmount.PodUID)
		if unmapErr != nil {
			// On failure, return error. Caller will log and retry.
			return volumeToUnmount.GenerateError("UnmapVolume.UnmapBlockVolume failed", unmapErr)
		}
```

调用 `ioutil.UnmapBlockVolume()` 执行块设备的 umount 操作。在其内部会先删除文件锁，然后将块设备从 Pod 关联的 Volume 局部挂载点上执行 umount 操作，然后从全局挂载点再一次执行 umount 操作。

``` go
		// Call UnmapPodDevice if blockVolumeUnmapper implements CustomBlockVolumeUnmapper
		if customBlockVolumeUnmapper, ok := blockVolumeUnmapper.(volume.CustomBlockVolumeUnmapper); ok {
			// Execute plugin specific unmap
			unmapErr = customBlockVolumeUnmapper.UnmapPodDevice()
			if unmapErr != nil {
				// On failure, return error. Caller will log and retry.
				return volumeToUnmount.GenerateError("UnmapVolume.UnmapPodDevice failed", unmapErr)
			}
		}
```

如果 BlockVolumeUnmapper 对象实现了 CustomBlockVolumeUnmapper interface，则调用 `UnmapPodDevice()`，主要用于 CSI。

``` go
		klog.Infof(
			"UnmapVolume succeeded for volume %q (OuterVolumeSpecName: %q) pod %q (UID: %q). InnerVolumeSpecName %q. PluginName %q, VolumeGidValue %q",
			volumeToUnmount.VolumeName,
			volumeToUnmount.OuterVolumeSpecName,
			volumeToUnmount.PodName,
			volumeToUnmount.PodUID,
			volumeToUnmount.InnerVolumeSpecName,
			volumeToUnmount.PluginName,
			volumeToUnmount.VolumeGidValue)

		// Update actual state of world
		markVolUnmountedErr := actualStateOfWorld.MarkVolumeAsUnmounted(
			volumeToUnmount.PodName, volumeToUnmount.VolumeName)
		if markVolUnmountedErr != nil {
			// On failure, just log and exit
			klog.Errorf(volumeToUnmount.GenerateErrorDetailed("UnmapVolume.MarkVolumeAsUnmounted failed", markVolUnmountedErr).Error())
		}

		return nil, nil
	}

	return volumetypes.GeneratedOperations{
		OperationName:     "unmap_volume",
		OperationFunc:     unmapVolumeFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(blockVolumePlugin.GetPluginName(), volumeToUnmount.VolumeSpec), "unmap_volume"),
		EventRecorderFunc: nil, // nil because we do not want to generate event on error
	}, nil
}
```

执行成功后，使用 `MarkVolumeAsUnmounted()` 更新 actualStateOfWorld，并返回。

## UnmountDevice() ##

`UnmountDevice()` 将存储设备从节点上的全局挂载点执行 umount 操作。

``` go
func (oe *operationExecutor) UnmountDevice(
	deviceToDetach AttachedVolume,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	hostutil hostutil.HostUtils) error {
	fsVolume, err := util.CheckVolumeModeFilesystem(deviceToDetach.VolumeSpec)
	if err != nil {
		return err
	}
	var generatedOperations volumetypes.GeneratedOperations
	if fsVolume {
		// Filesystem volume case
		// Unmount and detach a device if a volume isn't referenced
		generatedOperations, err = oe.operationGenerator.GenerateUnmountDeviceFunc(
			deviceToDetach, actualStateOfWorld, hostutil)
	} else {
		// Block volume case
		// Detach a device and remove loopback if a volume isn't referenced
		generatedOperations, err = oe.operationGenerator.GenerateUnmapDeviceFunc(
			deviceToDetach, actualStateOfWorld, hostutil)
	}
	if err != nil {
		return err
	}
	// Avoid executing unmount/unmap device from multiple pods referencing
	// the same volume in parallel
	podName := nestedpendingoperations.EmptyUniquePodName

	return oe.pendingOperations.Run(
		deviceToDetach.VolumeName, podName, "" /* nodeName */, generatedOperations)
}
```

会根据存储设备是作为文件系统使用还是作为块设备使用这两种不同的使用方式生成不同的函数：`GenerateUnmountDeviceFunc()` 和 `GenerateUnmapDeviceFunc()`，下面分别进行分析。

### GenerateUnmountDeviceFunc() ###

``` go
func (og *operationGenerator) GenerateUnmountDeviceFunc(
	deviceToDetach AttachedVolume,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	hostutil hostutil.HostUtils) (volumetypes.GeneratedOperations, error) {
	// Get DeviceMounter plugin
	deviceMountableVolumePlugin, err :=
		og.volumePluginMgr.FindDeviceMountablePluginByName(deviceToDetach.PluginName)
	if err != nil || deviceMountableVolumePlugin == nil {
		return volumetypes.GeneratedOperations{}, deviceToDetach.GenerateErrorDetailed("UnmountDevice.FindDeviceMountablePluginByName failed", err)
	}

	volumeDeviceUmounter, err := deviceMountableVolumePlugin.NewDeviceUnmounter()
	if err != nil {
		return volumetypes.GeneratedOperations{}, deviceToDetach.GenerateErrorDetailed("UnmountDevice.NewDeviceUmounter failed", err)
	}

	volumeDeviceMounter, err := deviceMountableVolumePlugin.NewDeviceMounter()
	if err != nil {
		return volumetypes.GeneratedOperations{}, deviceToDetach.GenerateErrorDetailed("UnmountDevice.NewDeviceMounter failed", err)
	}
```

首先根据 PluginName 获取 DeviceMountableVolumePlugin 对象，进而获取 DeviceUnmounter 对象。

``` go
	unmountDeviceFunc := func() (error, error) {
		//deviceMountPath := deviceToDetach.DeviceMountPath
		deviceMountPath, err :=
			volumeDeviceMounter.GetDeviceMountPath(deviceToDetach.VolumeSpec)
		if err != nil {
			// On failure, return error. Caller will log and retry.
			return deviceToDetach.GenerateError("GetDeviceMountPath failed", err)
		}
		refs, err := deviceMountableVolumePlugin.GetDeviceMountRefs(deviceMountPath)

		if err != nil || util.HasMountRefs(deviceMountPath, refs) {
			if err == nil {
				err = fmt.Errorf("The device mount path %q is still mounted by other references %v", deviceMountPath, refs)
			}
			return deviceToDetach.GenerateError("GetDeviceMountRefs check failed", err)
		}
```

接着开始返回函数的定义，在函数中，首先确保存储设备没有再被引用。这里的 `GetDeviceMountRefs()` 参数是全局挂载路径，这个函数会根据这个全局挂载路径找到对应的存储设备，然后再查找此存储设备有没有其它的挂载点。

``` go
		// Execute unmount
		unmountDeviceErr := volumeDeviceUmounter.UnmountDevice(deviceMountPath)
		if unmountDeviceErr != nil {
			// On failure, return error. Caller will log and retry.
			return deviceToDetach.GenerateError("UnmountDevice failed", unmountDeviceErr)
		}
```

然后使用 `UnmountDevice()` 进行 umount 操作，这个函数由 Volume Plugin 自己定义。作用是将存储设备从节点的全局挂载点上 umount。

``` go
		// Before logging that UnmountDevice succeeded and moving on,
		// use hostutil.PathIsDevice to check if the path is a device,
		// if so use hostutil.DeviceOpened to check if the device is in use anywhere
		// else on the system. Retry if it returns true.
		deviceOpened, deviceOpenedErr := isDeviceOpened(deviceToDetach, hostutil)
		if deviceOpenedErr != nil {
			return nil, deviceOpenedErr
		}
		// The device is still in use elsewhere. Caller will log and retry.
		if deviceOpened {
			return deviceToDetach.GenerateError(
				"UnmountDevice failed",
				goerrors.New("the device is in use when it was no longer expected to be in use"))
		}
```

再一次检测存储设备是否还被系统上的其它地方使用。

``` go
		klog.Infof(deviceToDetach.GenerateMsg("UnmountDevice succeeded", ""))

		// Update actual state of world
		markDeviceUnmountedErr := actualStateOfWorld.MarkDeviceAsUnmounted(
			deviceToDetach.VolumeName)
		if markDeviceUnmountedErr != nil {
			// On failure, return error. Caller will log and retry.
			return deviceToDetach.GenerateError("MarkDeviceAsUnmounted failed", markDeviceUnmountedErr)
		}

		return nil, nil
	}

	return volumetypes.GeneratedOperations{
		OperationName:     "unmount_device",
		OperationFunc:     unmountDeviceFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(deviceMountableVolumePlugin.GetPluginName(), deviceToDetach.VolumeSpec), "unmount_device"),
		EventRecorderFunc: nil, // nil because we do not want to generate event on error
	}, nil
}
```

使用 `MarkDeviceAsUnmounted()` 更新 actualStateOfWorld 状态，并返回。

### GenerateUnmapDeviceFunc() ###

``` go
func (og *operationGenerator) GenerateUnmapDeviceFunc(
	deviceToDetach AttachedVolume,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	hostutil hostutil.HostUtils) (volumetypes.GeneratedOperations, error) {

	blockVolumePlugin, err :=
		og.volumePluginMgr.FindMapperPluginByName(deviceToDetach.PluginName)
	if err != nil {
		return volumetypes.GeneratedOperations{}, deviceToDetach.GenerateErrorDetailed("UnmapDevice.FindMapperPluginByName failed", err)
	}

	if blockVolumePlugin == nil {
		return volumetypes.GeneratedOperations{}, deviceToDetach.GenerateErrorDetailed("UnmapDevice.FindMapperPluginByName failed to find BlockVolumeMapper plugin. Volume plugin is nil.", nil)
	}

	blockVolumeUnmapper, newUnmapperErr := blockVolumePlugin.NewBlockVolumeUnmapper(
		deviceToDetach.VolumeSpec.Name(),
		"" /* podUID */)
	if newUnmapperErr != nil {
		return volumetypes.GeneratedOperations{}, deviceToDetach.GenerateErrorDetailed("UnmapDevice.NewUnmapper failed", newUnmapperErr)
	}
```

首先根据 PluginName 获取 BlockVolumePlugin 对象，进而获取 BlockVolumeUnmapper 对象。

``` go
	unmapDeviceFunc := func() (error, error) {
		// Search under globalMapPath dir if all symbolic links from pods have been removed already.
		// If symbolic links are there, pods may still refer the volume.
		globalMapPath := deviceToDetach.DeviceMountPath
		refs, err := og.blkUtil.GetDeviceBindMountRefs(deviceToDetach.DevicePath, globalMapPath)
		if err != nil {
			if os.IsNotExist(err) {
				// Looks like SetupDevice did not complete. Fall through to TearDownDevice and mark the device as unmounted.
				refs = nil
			} else {
				return deviceToDetach.GenerateError("UnmapDevice.GetDeviceBindMountRefs check failed", err)
			}
		}
		if len(refs) > 0 {
			err = fmt.Errorf("The device %q is still referenced from other Pods %v", globalMapPath, refs)
			return deviceToDetach.GenerateError("UnmapDevice failed", err)
		}
```

接着开始返回函数的定义，在函数中，首先确保存储设备没有再被引用。

``` go
		// Call TearDownDevice if blockVolumeUnmapper implements CustomBlockVolumeUnmapper
		if customBlockVolumeUnmapper, ok := blockVolumeUnmapper.(volume.CustomBlockVolumeUnmapper); ok {
			// Execute tear down device
			unmapErr := customBlockVolumeUnmapper.TearDownDevice(globalMapPath, deviceToDetach.DevicePath)
			if unmapErr != nil {
				// On failure, return error. Caller will log and retry.
				return deviceToDetach.GenerateError("UnmapDevice.TearDownDevice failed", unmapErr)
			}
		}
```

如果 BlockVolumeUnmapper 对象实现了 CustomBlockVolumeUnmapper interface，则调用 `TearDownDevice()`，主要用于 CSI。

``` go
		// Plugin finished TearDownDevice(). Now globalMapPath dir and plugin's stored data
		// on the dir are unnecessary, clean up it.
		removeMapPathErr := og.blkUtil.RemoveMapPath(globalMapPath)
		if removeMapPathErr != nil {
			// On failure, return error. Caller will log and retry.
			return deviceToDetach.GenerateError("UnmapDevice.RemoveMapPath failed", removeMapPathErr)
		}
```

删除全局挂载点目录以及其下面的的元数据等文件。

``` go
		// Before logging that UnmapDevice succeeded and moving on,
		// use hostutil.PathIsDevice to check if the path is a device,
		// if so use hostutil.DeviceOpened to check if the device is in use anywhere
		// else on the system. Retry if it returns true.
		deviceOpened, deviceOpenedErr := isDeviceOpened(deviceToDetach, hostutil)
		if deviceOpenedErr != nil {
			return nil, deviceOpenedErr
		}
		// The device is still in use elsewhere. Caller will log and retry.
		if deviceOpened {
			return deviceToDetach.GenerateError(
				"UnmapDevice failed",
				fmt.Errorf("the device is in use when it was no longer expected to be in use"))
		}

		klog.Infof(deviceToDetach.GenerateMsgDetailed("UnmapDevice succeeded", ""))
```

再一次检测存储设备是否还被系统上的其它地方使用。

``` go
		// Update actual state of world
		markDeviceUnmountedErr := actualStateOfWorld.MarkDeviceAsUnmounted(
			deviceToDetach.VolumeName)
		if markDeviceUnmountedErr != nil {
			// On failure, return error. Caller will log and retry.
			return deviceToDetach.GenerateError("MarkDeviceAsUnmounted failed", markDeviceUnmountedErr)
		}

		return nil, nil
	}

	return volumetypes.GeneratedOperations{
		OperationName:     "unmap_device",
		OperationFunc:     unmapDeviceFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(blockVolumePlugin.GetPluginName(), deviceToDetach.VolumeSpec), "unmap_device"),
		EventRecorderFunc: nil, // nil because we do not want to generate event on error
	}, nil
}
```

使用 `MarkDeviceAsUnmounted()` 更新 actualStateOfWorld 状态，并返回。

## 参考 ##

- [Using subPath](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath)
