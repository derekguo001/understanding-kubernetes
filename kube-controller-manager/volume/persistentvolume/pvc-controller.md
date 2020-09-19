# PVC Controller #

目录

- [概述](#概述)
- [入口函数 `claimWorker()`](#入口函数-claimWorker())
- [核心实现 `syncClaim()`](#核心实现-syncClaim())
  - [PVC 未与 PV 绑定](#PVC-未与-PV-绑定)
    - [未明确指定 PV 名称](#未明确指定-PV-名称)
    - [明确指定了 PV 名称](#明确指定了-PV-名称)
  - [PVC 已经与 PV 绑定](#PVC-已经与-PV-绑定)

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

### PVC 未与 PV 绑定 ###

`syncUnboundClaim()` 处理和 PV 之间没有绑定关系的 PVC 对象。用于新建的 PVC，这时 PVC 为 Pending 状态，按照用户是否手动指定 `claim.Spec.VolumeName` 又细分为两类：

1. 未明确指定 PV 名称。
2. 明确指定了 PV 名称。

这里 `claim.Spec.VolumeName` 是用户在创建 PVC 的时候指定的，可参见 [手动指定绑定对象](../../../volume/pv-pvc-bind.md#手动指定绑定对象)。下面分析这两种情况。

#### 未明确指定 PV 名称 ####

如果用户没有手动指定当前 PVC 关联的 PV 对象，则执行下面的逻辑。

``` go
	if claim.Spec.VolumeName == "" {
		delayBinding, err := pvutil.IsDelayBindingMode(claim, ctrl.classLister)
        ...
		volume, err := ctrl.volumes.findBestMatchForClaim(claim, delayBinding)
        ...
```

首先会尝试使用 `findBestMatchForClaim()` 查找一个已创建的 PV。

``` go
		if volume == nil {
            ...
			switch {
			case delayBinding && !pvutil.IsDelayBindingProvisioning(claim):
				ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, events.WaitForFirstConsumer, "waiting for first consumer to be created before binding")
			case v1helper.GetPersistentVolumeClaimClass(claim) != "":
				if err = ctrl.provisionClaim(claim); err != nil {
					return err
				}
				return nil
			default:
				ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, events.FailedBinding, "no persistent volumes available for this claim and no storage class is set")
			}

			// Mark the claim as Pending and try to find a match in the next
			// periodic syncClaim
			if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
				return err
			}
			return nil
		} else /* pv != nil */ {
```

如果找不到，则会尝试通过调用 `provisionClaim()` 来使用 PVC 中的 StorageClass 动态创建一个 PV 然后与其绑定。如果动态创建出错，则将 PVC 设置为 Pending 状态。另外对于延迟绑定的 PVC 在这个地方也不会做任何操作，会直接返回。

上面代码中，查找已存在的 PV 的函数为 `findByClaim()`，具体查找算法是找到一个大小大于等于 PVC 大小的 PV，并且访问模式需要一致。

``` go
			// Found a PV for this claim
			// OBSERVATION: pvc is "Pending", pv is "Available"
			claimKey := claimToClaimKey(claim)
			klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q found: %s", claimKey, volume.Name, getVolumeStatusForLogging(volume))
			if err = ctrl.bind(volume, claim); err != nil {
				// On any error saving the volume or the claim, subsequent
				// syncClaim will finish the binding.
				// record count error for provision if exists
				// timestamp entry will remain in cache until a success binding has happened
				metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, err)
				return err
			}
			// OBSERVATION: claim is "Bound", pv is "Bound"
			// if exists a timestamp entry in cache, record end to end provision latency and clean up cache
			// End of the provision + binding operation lifecycle, cache will be cleaned by "RecordMetric"
			// [Unit test 12-1, 12-2, 12-4]
			metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, nil)
			return nil
		}
	} else /* pvc.Spec.VolumeName != nil */ {
```

如果使用 `findByClaim()` 可以找到匹配的 PV 对象，则使用 `bind()` 进行绑定。

#### 明确指定了 PV 名称 ####

如果用户手动指定了当前 PVC 关联的 PV 对象，则执行下面的逻辑。

``` go
	} else /* pvc.Spec.VolumeName != nil */ {
		// [Unit test set 2]
		// User asked for a specific PV.
		klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested", claimToClaimKey(claim), claim.Spec.VolumeName)
		obj, found, err := ctrl.volumes.store.GetByKey(claim.Spec.VolumeName)
		if err != nil {
			return err
		}
		if !found {
			// User asked for a PV that does not exist.
			// OBSERVATION: pvc is "Pending"
			// Retry later.
			klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested and not found, will try again next time", claimToClaimKey(claim), claim.Spec.VolumeName)
			if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
				return err
			}
			return nil
		} else {
```

如果用户指定的 PV 不存在，则将当前 PVC 更新为 Pending 状态后直接返回。

``` go
        ...
		} else {
			volume, ok := obj.(*v1.PersistentVolume)
			if !ok {
				return fmt.Errorf("Cannot convert object from volume cache to volume %q!?: %+v", claim.Spec.VolumeName, obj)
			}
			klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested and found: %s", claimToClaimKey(claim), claim.Spec.VolumeName, getVolumeStatusForLogging(volume))
```

如果用户指定的 PV 存在，则进一步检查其状态。

``` go
			if volume.Spec.ClaimRef == nil {
				// User asked for a PV that is not claimed
				// OBSERVATION: pvc is "Pending", pv is "Available"
				klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume is unbound, binding", claimToClaimKey(claim))
				if err = checkVolumeSatisfyClaim(volume, claim); err != nil {
					klog.V(4).Infof("Can't bind the claim to volume %q: %v", volume.Name, err)
					// send an event
					msg := fmt.Sprintf("Cannot bind to requested volume %q: %s", volume.Name, err)
					ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.VolumeMismatch, msg)
					// volume does not satisfy the requirements of the claim
					if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
						return err
					}
				} else if err = ctrl.bind(volume, claim); err != nil {
					// On any error saving the volume or the claim, subsequent
					// syncClaim will finish the binding.
					return err
				}
				// OBSERVATION: pvc is "Bound", pv is "Bound"
				return nil
			} else if pvutil.IsVolumeBoundToClaim(volume, claim) {
```

如果 PV 对象没有绑定的 PVC 对象，则先比较 PV 和当前 PVC 是否匹配，会比较 StorageClass、大小、访问模式等信息，如果不匹配，则将当前 PVC 设置为 Pending 状态后返回。如果可以匹配，则执行 `bind()` 进行绑定。

``` go
			} else if pvutil.IsVolumeBoundToClaim(volume, claim) {
				// User asked for a PV that is claimed by this PVC
				// OBSERVATION: pvc is "Pending", pv is "Bound"
				klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound, finishing the binding", claimToClaimKey(claim))

				// Finish the volume binding by adding claim UID.
				if err = ctrl.bind(volume, claim); err != nil {
					return err
				}
				// OBSERVATION: pvc is "Bound", pv is "Bound"
				return nil
			} else {
```

如果用户在创建 PV 对象的时候也手动指明了让其与当前 PVC 进行绑定，则使用 `bind()` 对其进行绑定。

``` go
			} else {
				// User asked for a PV that is claimed by someone else
				// OBSERVATION: pvc is "Pending", pv is "Bound"
				if !metav1.HasAnnotation(claim.ObjectMeta, pvutil.AnnBoundByController) {
					klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound to different claim by user, will retry later", claimToClaimKey(claim))
					// User asked for a specific PV, retry later
					if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
						return err
					}
					return nil
				} else {
					// This should never happen because someone had to remove
					// AnnBindCompleted annotation on the claim.
					klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound to different claim %q by controller, THIS SHOULD NEVER HAPPEN", claimToClaimKey(claim), claimrefToClaimKey(volume.Spec.ClaimRef))
					return fmt.Errorf("Invalid binding of claim %q to volume %q: volume already claimed by %q", claimToClaimKey(claim), claim.Spec.VolumeName, claimrefToClaimKey(volume.Spec.ClaimRef))
				}
			}
		}
	}
```

如果发现 PV 对象绑定了其它 PVC 对象，则将当前 PVC 对象设置为 Pending 状态，以便下一次处理，然后返回。

### PVC 已经与 PV 绑定 ###

`syncBoundClaim()` 处理和 PV 之间已经有了绑定关系的 PVC 对象。分为几种情况：

``` go
func (ctrl *PersistentVolumeController) syncBoundClaim(claim *v1.PersistentVolumeClaim) error {
	// HasAnnotation(pvc, pvutil.AnnBindCompleted)
	// This PVC has previously been bound
	// OBSERVATION: pvc is not "Pending"
	// [Unit test set 3]
	if claim.Spec.VolumeName == "" {
		// Claim was bound before but not any more.
		if _, err := ctrl.updateClaimStatusWithEvent(claim, v1.ClaimLost, nil, v1.EventTypeWarning, "ClaimLost", "Bound claim has lost reference to PersistentVolume. Data on the volume is lost!"); err != nil {
			return err
		}
		return nil
	}
```

首先，如果当前 PVC 对于的 PV 绑定信息中的 Name 为空，则说明 PVC 与 PV 已经失去了关联关系，意味着所有的数据都已经丢失，这时将 PVC 设置为 Lost 状态，同时将它关联的 PV 信息置空，然后返回。

``` go
	obj, found, err := ctrl.volumes.store.GetByKey(claim.Spec.VolumeName)
	if err != nil {
		return err
	}
	if !found {
		// Claim is bound to a non-existing volume.
		if _, err = ctrl.updateClaimStatusWithEvent(claim, v1.ClaimLost, nil, v1.EventTypeWarning, "ClaimLost", "Bound claim has lost its PersistentVolume. Data on the volume is lost!"); err != nil {
			return err
		}
		return nil
	} else {
```

如果当前 PVC 对于的 PV 绑定信息中的 Name 不为空，则根据它找到对应的 PV 对象，如果找不到，说明 PV 已删除，这时将 PVC 设置为 Lost 状态，同时将它关联的 PV 信息置空，然后并返回。

``` go
	} else {
		volume, ok := obj.(*v1.PersistentVolume)
		if !ok {
			return fmt.Errorf("Cannot convert object from volume cache to volume %q!?: %#v", claim.Spec.VolumeName, obj)
		}
```

如果找到了 PV 对象，则进一步检查其相关信息。

``` go
		klog.V(4).Infof("synchronizing bound PersistentVolumeClaim[%s]: volume %q found: %s", claimToClaimKey(claim), claim.Spec.VolumeName, getVolumeStatusForLogging(volume))
		if volume.Spec.ClaimRef == nil {
			// Claim is bound but volume has come unbound.
			// Or, a claim was bound and the controller has not received updated
			// volume yet. We can't distinguish these cases.
			// Bind the volume again and set all states to Bound.
			klog.V(4).Infof("synchronizing bound PersistentVolumeClaim[%s]: volume is unbound, fixing", claimToClaimKey(claim))
			if err = ctrl.bind(volume, claim); err != nil {
				// Objects not saved, next syncPV or syncClaim will try again
				return err
			}
			return nil
		} else if volume.Spec.ClaimRef.UID == claim.UID {
```

首先，检查 PV 中关联的 PVC 的信息，如果为空，说明 PVC 到 PV 的关联关系已经有了，但是从 PV 到 PVC 的关联关系没有，即绑定操作完成了一半，则执行 `bind()` 重新进行绑定。

``` go
		} else if volume.Spec.ClaimRef.UID == claim.UID {
			// All is well
			// NOTE: syncPV can handle this so it can be left out.
			// NOTE: bind() call here will do nothing in most cases as
			// everything should be already set.
			klog.V(4).Infof("synchronizing bound PersistentVolumeClaim[%s]: claim is already correctly bound", claimToClaimKey(claim))
			if err = ctrl.bind(volume, claim); err != nil {
				// Objects not saved, next syncPV or syncClaim will try again
				return err
			}
			return nil
		} else {
```

如果 PV 中关联的 PVC 就是当前 PVC，说明一切正常，这时仍然会调用 `bind()` 进行绑定。<TODO> why ? </TODO>

``` go
		} else {
			// Claim is bound but volume has a different claimant.
			// Set the claim phase to 'Lost', which is a terminal
			// phase.
			if _, err = ctrl.updateClaimStatusWithEvent(claim, v1.ClaimLost, nil, v1.EventTypeWarning, "ClaimMisbound", "Two claims are bound to the same volume, this one is bound incorrectly"); err != nil {
				return err
			}
			return nil
		}
	}
}
```

如果 PV 中关联的 PVC 不是当前 PVC，说明当前 PVC 的关联的 PV 信息已经失去了意义，这时将其状态更新为 Lost，同时将它关联的 PV 信息置空，然后返回。
