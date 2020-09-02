# PV Controller #

为了便于描述，我们在本节中将 PV 对象的相关处理逻辑称为 PV Controller，它属于 Kube Controller Manager 中 PersistentVolumeController 的一部分，相关的代码位于 `pkg/controller/volume/persistentvolume/` 下面的 `pv_controller_base.go` 和 `pv_controller.go` 中。

它的主要作用是更新 PV 对象的状态，执行从 PV 到对应的 PVC 的绑定操作。

``` go
func (ctrl *PersistentVolumeController) Run(stopCh <-chan struct{}) {
    ...
	go wait.Until(ctrl.volumeWorker, time.Second, stopCh)
    ...
	<-stopCh
}
```

在 `PersistentVolumeController` 启动后，会分别启动两个 goroutine `volumeWorker` 处理 PV 对象。这个可以看做是 PV Controller 的入口函数。它每秒执行一次。

## 入口函数 `volumeWorker()` ##

``` go
func (ctrl *PersistentVolumeController) volumeWorker() {
	workFunc := func() bool {
		keyObj, quit := ctrl.volumeQueue.Get()
        ...
	}
	for {
		if quit := workFunc(); quit {
			klog.Infof("volume worker queue shutting down")
			return
		}
	}
}
```

它会每次从 PV 队列里取一个 PV 对象，下面是 `ctrl.volumeQueue.Get()`：

``` go
func (q *Type) Get() (item interface{}, shutdown bool) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	for len(q.queue) == 0 && !q.shuttingDown {
		q.cond.Wait()
	}
	if len(q.queue) == 0 {
		// We must be shutting down.
		return nil, true
	}

	item, q.queue = q.queue[0], q.queue[1:]

	q.metrics.get(item)

	q.processing.insert(item)
	q.dirty.delete(item)

	return item, false
}
```

如果队列为空，则 `volumeWorker()` 会直接返回，注意这里不会阻塞。

取到 PV 对象后，`volumeWorker()` 会根据 PersistentVolumeController.volumeLister 中还有没有这个对象，再分别进行处理，代码如下：

``` go
	workFunc := func() bool {
		keyObj, quit := ctrl.volumeQueue.Get()
		if quit {
			return true
		}
		defer ctrl.volumeQueue.Done(keyObj)
		key := keyObj.(string)
		klog.V(5).Infof("volumeWorker[%s]", key)

		_, name, err := cache.SplitMetaNamespaceKey(key)
		if err != nil {
			klog.V(4).Infof("error getting name of volume %q to get volume from informer: %v", key, err)
			return false
		}
		volume, err := ctrl.volumeLister.Get(name)
		if err == nil {
			// The volume still exists in informer cache, the event must have
			// been add/update/sync
			ctrl.updateVolume(volume)
			return false
		}
		if !errors.IsNotFound(err) {
			klog.V(2).Infof("error getting volume %q from informer: %v", key, err)
			return false
		}

		// The volume is not in informer cache, the event must have been
		// "delete"
		volumeObj, found, err := ctrl.volumes.store.GetByKey(key)
		if err != nil {
			klog.V(2).Infof("error getting volume %q from cache: %v", key, err)
			return false
		}
		if !found {
			// The controller has already processed the delete event and
			// deleted the volume from its cache
			klog.V(2).Infof("deletion of volume %q was already processed", key)
			return false
		}
		volume, ok := volumeObj.(*v1.PersistentVolume)
		if !ok {
			klog.Errorf("expected volume, got %+v", volumeObj)
			return false
		}
		ctrl.deleteVolume(volume)
		return false
	}
```

如果 PersistentVolumeController.volumeLister 有当前这个 PV 对象，则说明这个 PV 仍然存在于 Kube API Server 中，这个 PV 对应的操作就是 Add 或者 Update，也可能是周期的同步操作，见上一节 [PersistentVolumeController](./overview.md) 中的描述。这种情况下，使用 `ctrl.updateVolume()` 对其进行处理。

如果 PersistentVolumeController.volumeLister 中没有当前这个 PV 对象，则说明它已经在 Kube API Server 中被删除，那么这个 PV 对应的操作就是 Delete，对应的函数是 `ctrl.deleteVolume()`。

``` go
func (ctrl *PersistentVolumeController) deleteVolume(volume *v1.PersistentVolume) {
	if err := ctrl.volumes.store.Delete(volume); err != nil {
       ...
    }

	if volume.Spec.ClaimRef == nil {
		return
	}
    ...
	claimKey := claimrefToClaimKey(volume.Spec.ClaimRef)
	klog.V(5).Infof("deleteVolume[%s]: scheduling sync of claim %q", volume.Name, claimKey)
	ctrl.claimQueue.Add(claimKey)
}
```

PV Controller 会先删除本地的缓存，即把它从 `ctrl.volumes.store` 中删除，然后看当前 PV 是否有关联的 PVC 对象，如果有的话，则将对应的 PVC 加入 PVC 队列，再由 PVC Controller 对其进行处理。

现在回头看 `ctrl.updateVolume()`，它会处理 PV 的 Add 或者 Update 等操作。

``` go
func (ctrl *PersistentVolumeController) updateVolume(volume *v1.PersistentVolume) {
	// Store the new volume version in the cache and do not process it if this
	// is an old version.
	new, err := ctrl.storeVolumeUpdate(volume)
	if err != nil {
		klog.Errorf("%v", err)
	}
	if !new {
		return
	}

	err = ctrl.syncVolume(volume)
	if err != nil {
        ...
	}
}
```

它首先更新本地的缓存，然后调用 `ctrl.syncVolume(volume)` 对 PV 进行处理，这个函数是 PV Controller 的核心实现，PV 处理过程中主要的逻辑都在其中。
