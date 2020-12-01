# desiredStateOfWorldPopulator #

desiredStateOfWorldPopulator 会周期性地根据 Kube API Server 的内容来同步 desiredStateOfWorld 中的信息，同步周期为一分钟。代码位于 `pkg/controller/volume/attachdetach/populator`。

执行 `Run()` 进行启动。

``` go
func (dswp *desiredStateOfWorldPopulator) Run(stopCh <-chan struct{}) {
	wait.Until(dswp.populatorLoopFunc(), dswp.loopSleepDuration, stopCh)
}

func (dswp *desiredStateOfWorldPopulator) populatorLoopFunc() func() {
	return func() {
		dswp.findAndRemoveDeletedPods()

		// findAndAddActivePods is called periodically, independently of the main
		// populator loop.
		if time.Since(dswp.timeOfLastListPods) < dswp.listPodsRetryDuration {
			klog.V(5).Infof(
				"Skipping findAndAddActivePods(). Not permitted until %v (listPodsRetryDuration %v).",
				dswp.timeOfLastListPods.Add(dswp.listPodsRetryDuration),
				dswp.listPodsRetryDuration)

			return
		}
		dswp.findAndAddActivePods()
	}
}
```

这里主要执行 `findAndRemoveDeletedPods()` 和 `findAndAddActivePods()`。

其中涉及到了两个时间，`dswp.loopSleepDuration` 和 `dswp.listPodsRetryDuration`，他们分别对应于：

``` go
	DesiredStateOfWorldPopulatorLoopSleepPeriod:       1 * time.Minute,
	DesiredStateOfWorldPopulatorListPodsRetryDuration: 3 * time.Minute,
```

第一个表示执行 `populatorLoopFunc()` 的间隔时间，第二个表示执行 `dswp.findAndAddActivePods()` 的间隔时间。

下面对这里涉及的两个主要的函数进行分析。
  
## findAndRemoveDeletedPods() ##

`findAndRemoveDeletedPods()` 会通过 informer 从 API Server 中找出已经不存在关联关系的 Pod 和 Volume，将其从 desiredStateOfWorld 中删除。

``` go
func (dswp *desiredStateOfWorldPopulator) findAndRemoveDeletedPods() {
	for dswPodUID, dswPodToAdd := range dswp.desiredStateOfWorld.GetPodToAdd() {
		dswPodKey, err := kcache.MetaNamespaceKeyFunc(dswPodToAdd.Pod)
		if err != nil {
			klog.Errorf("MetaNamespaceKeyFunc failed for pod %q (UID %q) with: %v", dswPodKey, dswPodUID, err)
			continue
		}

		// Retrieve the pod object from pod informer with the namespace key
		namespace, name, err := kcache.SplitMetaNamespaceKey(dswPodKey)
		if err != nil {
			utilruntime.HandleError(fmt.Errorf("error splitting dswPodKey %q: %v", dswPodKey, err))
			continue
		}
		informerPod, err := dswp.podLister.Pods(namespace).Get(name)
```

会遍历 desiredStateOfWorld 中的每一个 Pod ，然后用 informer 去 API Server 查询此 Pod 是否还存在。

```
		switch {
		case errors.IsNotFound(err):
			// if we can't find the pod, we need to delete it below
		case err != nil:
			klog.Errorf("podLister Get failed for pod %q (UID %q) with %v", dswPodKey, dswPodUID, err)
			continue
```

如果 Pod 已经不存在，则不执行特别的操作，后续结束了 switch 语句后会执行删除操作，将这个 Pod 和它引用的 Volume 从 DesiredStateOfWorld 中删除。

``` go
		default:
			volumeActionFlag := util.DetermineVolumeAction(
				informerPod,
				dswp.desiredStateOfWorld,
				true /* default volume action */)

			if volumeActionFlag {
				informerPodUID := volutil.GetUniquePodName(informerPod)
				// Check whether the unique identifier of the pod from dsw matches the one retrieved from pod informer
				if informerPodUID == dswPodUID {
					klog.V(10).Infof("Verified pod %q (UID %q) from dsw exists in pod informer.", dswPodKey, dswPodUID)
					continue
				}
			}
		}
```

如果从 API Server 正确地查询到了 Pod，并且它有关联的 Volume 需要执行 attach 操作，这是会将查询到的 Pod id 与 desiredStateOfWorld 中 id 进行比较，如果一样，则相当于执行了一次验证操作，会直接返回。

``` go
		// the pod from dsw does not exist in pod informer, or it does not match the unique identifier retrieved
		// from the informer, delete it from dsw
		klog.V(1).Infof("Removing pod %q (UID %q) from dsw because it does not exist in pod informer.", dswPodKey, dswPodUID)
		dswp.desiredStateOfWorld.DeletePod(dswPodUID, dswPodToAdd.VolumeName, dswPodToAdd.NodeName)
	}
}
```

这里是执行完 switch 语句后需要执行的代码，会通过执行 `desiredStateOfWorld.DeletePod()` 将这个 Pod 和它引用的 Volume 从 DesiredStateOfWorld 中删除。

## findAndAddActivePods() ##

`findAndAddActivePods()` 会通过 informer 从 API Server 遍历所有的 active 的 pod，然后调用 `ProcessPodVolumes()` 处理这个 Pod，如果 Pod 有关联的 Volume 则把他们加入到 desiredStateOfWorld 中。关于 `ProcessPodVolumes()` 的详细分析可参见 [Pod 的处理](attachdetach.md#Pod-的处理)。

``` go
func (dswp *desiredStateOfWorldPopulator) findAndAddActivePods() {
	pods, err := dswp.podLister.List(labels.Everything())
	if err != nil {
		klog.Errorf("podLister List failed: %v", err)
		return
	}
	dswp.timeOfLastListPods = time.Now()

	for _, pod := range pods {
		if volutil.IsPodTerminated(pod, pod.Status) {
			// Do not add volumes for terminated pods
			continue
		}
		util.ProcessPodVolumes(pod, true,
			dswp.desiredStateOfWorld, dswp.volumePluginMgr, dswp.pvcLister, dswp.pvLister, dswp.csiMigratedPluginManager, dswp.intreeToCSITranslator)

	}

}
```
