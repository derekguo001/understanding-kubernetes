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

然后使用 Mounter 对象的 `CanMount()` 来做一些预检测，判断是否可以执行 mount 操作，如果一切顺利，执行 Mounter 对象的 `Setup()` 进行 mount。

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
