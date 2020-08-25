# Volume in Kubelet #

## 概述 ##

本节主要分析 Kubelet 组件中 Volume 相关的操作，主要是将挂载到节点上的存储 mount 到 pod 对应的目录，以及相反的 umount 操作。

在 Kubelet 中对 Volume 进行管理的组件被称为 Volume Manager。

``` go
type volumeManager struct {
	volumePluginMgr *volume.VolumePluginMgr

	desiredStateOfWorld cache.DesiredStateOfWorld
	actualStateOfWorld cache.ActualStateOfWorld
	reconciler reconciler.Reconciler

	desiredStateOfWorldPopulator populator.DesiredStateOfWorldPopulator
    ...
}
```

最核心的有几个：

- **desiredStateOfWorld** 一个缓存对象，用来保存当前节点上所有 Volume 期望的状态。
- **actualStateOfWorld** 另一个缓存对象，用来保存当前节点上所有 Volume 实际的状态。
- **reconciler** 用来周期性地比较 desiredStateOfWorld 和 actualStateOfWorld，然后对其中需要处理的 Volume 执行相应的操作。
- **desiredStateOfWorldPopulator** 周期性地处理当前节点上 active 状态的 pod，将需要处理的 Volume 加入 desiredStateOfWorld。
- **volumePluginMgr** 即 Volume Plugin Manager，用来管理所有的 Volume Plugin，可参见 [Volume Plugin Manager](../../volume/plugin-manager.md)。

## 启动流程 ##

Kubelet 启动后会首先载入所有 in tree 类型的 Volume Plugin，将其存入一个列表中。

``` go
// ProbeVolumePlugins collects all volume plugins into an easy to use list.
func ProbeVolumePlugins(featureGate featuregate.FeatureGate) ([]volume.VolumePlugin, error) {
	allPlugins := []volume.VolumePlugin{}
    ...
	allPlugins, err = appendLegacyProviderVolumes(allPlugins, featureGate)
	if err != nil {
		return allPlugins, err
	}

	allPlugins = append(allPlugins, emptydir.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, git_repo.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, hostpath.ProbeVolumePlugins(volume.VolumeConfig{})...)
	allPlugins = append(allPlugins, nfs.ProbeVolumePlugins(volume.VolumeConfig{})...)
	allPlugins = append(allPlugins, secret.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, iscsi.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, glusterfs.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, rbd.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, quobyte.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, cephfs.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, downwardapi.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, fc.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, flocker.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, configmap.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, projected.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, portworx.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, scaleio.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, local.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, storageos.ProbeVolumePlugins()...)
	allPlugins = append(allPlugins, csi.ProbeVolumePlugins()...)
	return allPlugins, nil
}
```

然后，使用这个列表作为参数创建一个 Volume Plugin Manager 对象。

``` go
	klet.volumePluginMgr, err =
		NewInitializedVolumePluginMgr(klet, secretManager, configMapManager, tokenManager, kubeDeps.VolumePlugins, kubeDeps.DynamicPluginProber)
	if err != nil {
		return nil, err
	}
```

接着，会使用 Volume Plugin Manager 作为参数和其它参数一起创建 Volume Manager 对象。

``` go
	klet.volumeManager = volumemanager.NewVolumeManager(
		kubeCfg.EnableControllerAttachDetach,
		nodeName,
		klet.podManager,
		klet.statusManager,
		klet.kubeClient,
		klet.volumePluginMgr,
		klet.containerRuntime,
		kubeDeps.Mounter,
		kubeDeps.HostUtil,
		klet.getPodsDir(),
		kubeDeps.Recorder,
		experimentalCheckNodeCapabilitiesBeforeMount,
		keepTerminatedPodVolumes,
		volumepathhandler.NewBlockVolumePathHandler())
```

现在所有的 Volume 相关的对象的初始化已经完成，接下来会开始启动。

``` go
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
    ...

	// Start volume manager
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

    ...
}
```

`Kubelet.Run()` 启动后会执行 `volumeManager.Run()`。

``` go
func (vm *volumeManager) Run(sourcesReady config.SourcesReady, stopCh <-chan struct{}) {
	defer runtime.HandleCrash()

	go vm.desiredStateOfWorldPopulator.Run(sourcesReady, stopCh)
	klog.V(2).Infof("The desired_state_of_world populator starts")

	klog.Infof("Starting Kubelet Volume Manager")
	go vm.reconciler.Run(stopCh)

	metrics.Register(vm.actualStateOfWorld, vm.desiredStateOfWorld, vm.volumePluginMgr)

	if vm.kubeClient != nil {
		// start informer for CSIDriver
		vm.volumePluginMgr.Run(stopCh)
	}

	<-stopCh
	klog.Infof("Shutting down Kubelet Volume Manager")
}
```

会依次启动 `desiredStateOfWorldPopulator.Run()` 和 `reconciler.Run()`，这两个 goroutine 会处理所有 Volume 相关的 mount/umount 等核心操作。最后还会启动 `volumePluginMgr.Run()`，这个是与 CSI 相关的，会在后面 CSI 部分中详细分析。
