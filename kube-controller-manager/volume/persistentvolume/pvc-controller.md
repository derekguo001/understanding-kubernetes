# PVC Controller #

目录

- [概述](#概述)
- [入口函数 `claimWorker()`](#入口函数-claimWorker())
- [核心实现 `syncClaim()`](#核心实现-syncClaim())

## 概述 ##

为了便于描述，我们在本节中将 PVC 对象的相关处理逻辑称为 PVC Controller，它和 PV Controller 类似，都属于 Kube Controller Manager 中 PersistentVolumeController 的一部分，相关的代码位于 `pkg/controller/volume/persistentvolume/` 下面的 `pv_controller_base.go` 和 `pv_controller.go` 中。

它的主要作用是更新 PVC 对象的状态，执行从 PVC 到对应的 PV 的绑定操作。

``` go
func (ctrl *PersistentVolumeController) Run(stopCh <-chan struct{}) {
    ...
	go wait.Until(ctrl.claimWorker, time.Second, stopCh)
    ...
	<-stopCh
}
```

在 `PersistentVolumeController` 启动后，会启动一个 goroutine `claimWorker` 处理 PV 对象。这个可以看做是 PVC Controller 的入口函数。它每秒执行一次。

PVC Controller 完成的函数调用栈如下所示：

``` go
claimWorker()
  updateClaim()
    syncClaim()
      syncUnboundClaim()
      syncBoundClaim()
```

## 入口函数 claimWorker() ##

``` go

func (ctrl *PersistentVolumeController) claimWorker() {
	workFunc := func() bool {
		keyObj, quit := ctrl.claimQueue.Get()
		if quit {
			return true
		}
        ...
	}
	for {
		if quit := workFunc(); quit {
			klog.Infof("claim worker queue shutting down")
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

如果队列为空，则 `claimWorker()` 会阻塞，直到有数据或者 shutdown 为止。

取到 PVC 对象后，`claimWorker()` 会根据 PersistentVolumeController.claimLister 中还有没有这个对象，再分别进行处理，代码如下：

``` go
	workFunc := func() bool {
		keyObj, quit := ctrl.claimQueue.Get()
		if quit {
			return true
		}
		defer ctrl.claimQueue.Done(keyObj)
		key := keyObj.(string)
		klog.V(5).Infof("claimWorker[%s]", key)

		namespace, name, err := cache.SplitMetaNamespaceKey(key)
		if err != nil {
			klog.V(4).Infof("error getting namespace & name of claim %q to get claim from informer: %v", key, err)
			return false
		}
		claim, err := ctrl.claimLister.PersistentVolumeClaims(namespace).Get(name)
		if err == nil {
			// The claim still exists in informer cache, the event must have
			// been add/update/sync
			ctrl.updateClaim(claim)
			return false
		}
		if !errors.IsNotFound(err) {
			klog.V(2).Infof("error getting claim %q from informer: %v", key, err)
			return false
		}

		// The claim is not in informer cache, the event must have been "delete"
		claimObj, found, err := ctrl.claims.GetByKey(key)
		if err != nil {
			klog.V(2).Infof("error getting claim %q from cache: %v", key, err)
			return false
		}
		if !found {
			// The controller has already processed the delete event and
			// deleted the claim from its cache
			klog.V(2).Infof("deletion of claim %q was already processed", key)
			return false
		}
		claim, ok := claimObj.(*v1.PersistentVolumeClaim)
		if !ok {
			klog.Errorf("expected claim, got %+v", claimObj)
			return false
		}
		ctrl.deleteClaim(claim)
		return false
	}
```

如果 PersistentVolumeController.claimLister 有当前这个 PVC 对象，则说明这个 PVC 仍然存在于 Kube API Server 中，这个 PVC 对应的操作就是 Add 或者 Update，也可能是周期的同步操作，见上一节 [PersistentVolumeController](./overview.md) 中的描述。这种情况下，使用 `ctrl.updateClaim()` 对其进行处理。

如果 PersistentVolumeController.claimLister 中没有当前这个 PVC 对象，则说明它已经在 Kube API Server 中被删除，那么这个 PVC 对应的操作就是 Delete，对应的函数是 `ctrl.deleteClaim()`。

``` go
func (ctrl *PersistentVolumeController) deleteClaim(claim *v1.PersistentVolumeClaim) {
	if err := ctrl.claims.Delete(claim); err != nil {
        ...
	}
	claimKey := claimToClaimKey(claim)
	klog.V(4).Infof("claim %q deleted", claimKey)
    ...
	ctrl.operationTimestamps.Delete(claimKey)

	volumeName := claim.Spec.VolumeName
	if volumeName == "" {
		klog.V(5).Infof("deleteClaim[%q]: volume not bound", claimKey)
		return
	}

    ...
	klog.V(5).Infof("deleteClaim[%q]: scheduling sync of volume %s", claimKey, volumeName)
	ctrl.volumeQueue.Add(volumeName)
}
```

PVC Controller 会先删除本地的缓存，即把它从 `ctrl.claims` 中删除，然后看当前 PVC 是否有关联的 PV 对象，如果有的话，则将对应的 PV 加入 PV 队列，再由 PV Controller 对其进行处理。

现在回头看 `ctrl.updateClaim()`，它会处理 PVC 的 Add 或者 Update 等操作。

``` go
func (ctrl *PersistentVolumeController) updateClaim(claim *v1.PersistentVolumeClaim) {
	// Store the new claim version in the cache and do not process it if this is
	// an old version.
	new, err := ctrl.storeClaimUpdate(claim)
	if err != nil {
		klog.Errorf("%v", err)
	}
	if !new {
		return
	}
	err = ctrl.syncClaim(claim)
    ...
}
```

它首先更新本地的缓存，然后调用 `ctrl.syncClaim(claim)` 对 PVC 进行处理，这个函数是 PVC Controller 的核心实现，PVC 处理过程中主要的逻辑都在其中。

## 核心实现 syncClaim() ##

``` go
func (ctrl *PersistentVolumeController) syncClaim(claim *v1.PersistentVolumeClaim) error {
    ...
	newClaim, err := ctrl.updateClaimMigrationAnnotations(claim)
    ...
	claim = newClaim

	if !metav1.HasAnnotation(claim.ObjectMeta, pvutil.AnnBindCompleted) {
		return ctrl.syncUnboundClaim(claim)
	} else {
		return ctrl.syncBoundClaim(claim)
	}
}
```

这个函数首先更新 PVC 对象的 Annotation，这个具体与 CSI 关系比较密切，在相关章节中再详细分析。

根据当前 PVC 对象是否与 PV 绑定，接下来分为两种情况分别进行处理：

1. PVC 未与 PV 绑定。
2. PVC 已经与 PV 绑定。

判断是否绑定的依据是根据 PVC 对象是否包含 `pv.kubernetes.io/bind-completed` 这个 Annotation，注意对应的值是什么没有关系。如果已绑定，则使用 `syncUnboundClaim()` 处理，否则使用 `syncBoundClaim()` 处理。下面针对这两种情况进行详细的分析。
