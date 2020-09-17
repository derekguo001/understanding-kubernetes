# PV Controller #

目录

- [概述](#概述)
- [入口函数 `volumeWorker()`](#入口函数-volumeWorker())
- [核心实现 `syncVolume()`](#核心实现-syncVolume())
  - [PV 未与 PVC 绑定](#PV-未与-PVC-绑定)
  - [PV 已经与 PVC 绑定](#PV-已经与-PVC-绑定)
    - [情景一、对应的 PVC 已被删除](#情景一、对应的-PVC-已被删除)
    - [情景二、绑定完成一半](#情景二、绑定完成一半)
    - [情景三、已正确绑定](#情景三、已正确绑定)
    - [情景四、发生了错误的绑定](#情景四、发生了错误的绑定)

## 概述 ##

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

在 `PersistentVolumeController` 启动后，会启动一个 goroutine `volumeWorker` 处理 PV 对象。这个可以看做是 PV Controller 的入口函数。它每秒执行一次。

PV Controller 完成的函数调用栈如下所示：

``` go
volumeWorker()
  updateVolume()
    syncVolume()
  deleteVolume()
```

## 入口函数 volumeWorker() ##

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

如果队列为空，则 `volumeWorker()` 会阻塞，直到有数据或者 shutdown 为止。

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

## 核心实现 syncVolume() ##

``` go
func (ctrl *PersistentVolumeController) syncVolume(volume *v1.PersistentVolume) error {
    ...
	newVolume, err := ctrl.updateVolumeMigrationAnnotations(volume)
    ...
```

这个函数首先更新 PV 对象的 Annotation，这个具体与 CSI 关系比较密切，在相关章节中再详细分析。

接下来分为两种情况：

1. PV 未与 PVC 绑定
2. PV 已经与 PVC 绑定

### PV 未与 PVC 绑定 ###

``` go
	volume = newVolume

	if volume.Spec.ClaimRef == nil {
        ...
		if _, err := ctrl.updateVolumePhase(volume, v1.VolumeAvailable, ""); err != nil {
            ...
			return err
		}
		return nil
	} else /* pv.Spec.ClaimRef != nil */ {
        ...
	}
}
```

接着根据当前 PV 是否有绑定的 PVC 再分别进行处理，如果没有的话，则简单地将其状态更新为 `Available` 状态，然后立刻返回，整个处理流程结束。

> 当 PV 关联的 PVC 被删除后这里的 `volume.Spec.ClaimRef` 仍然保留着对删掉的 PVC 的引用，因此不属于当前这种情况。

如果有绑定的 PVC 对象的话，再进行更进一步的处理。处理逻辑如下：

``` go
	} else /* pv.Spec.ClaimRef != nil */ {
		// Volume is bound to a claim.
		if volume.Spec.ClaimRef.UID == "" {
			// The PV is reserved for a PVC; that PVC has not yet been
			// bound to this PV; the PVC sync will handle it.
			klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume is pre-bound to claim %s", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef))
			if _, err := ctrl.updateVolumePhase(volume, v1.VolumeAvailable, ""); err != nil {
				// Nothing was saved; we will fall back into the same
				// condition in the next call to this method
				return err
			}
			return nil
		}
```

先判断 `volume.Spec.ClaimRef.UID` 是否为空，如果为空，说明这个 PV 的 `Spec.ClaimRef` 信息是由用户自己在创建时配置的，即 `volume.Spec.ClaimRef.Name` 不为空且 `volume.Spec.ClaimRef.UID` 为空。可参见 [手动指定绑定对象](../../../volume/pv-pvc-bind.md#手动指定绑定对象)

这个时候由于还没有完成绑定，因此对应的 PVC 的 UID 为空，这种情况下也会简单地将 PV 状态更新为 `Available` 状态，然后立刻返回。

### PV 已经与 PVC 绑定 ###

接下来的逻辑都是 `volume.Spec.ClaimRef` 不为空的情况下，即已经完成了从 PV 到 PVC 的绑定。

``` go
		// Get the PVC by _name_
		var claim *v1.PersistentVolumeClaim
		claimName := claimrefToClaimKey(volume.Spec.ClaimRef)
		obj, found, err := ctrl.claims.GetByKey(claimName)
```

先根据 `volume.Spec.ClaimRef` 找到对应的 PVC Name，再进一步找到对应的 PVC 对象。

注意这里有一种特殊情况需要处理，即根据 PVC Name 找到了对应的 PVC 对象，但是找到的 PVC 的 UID 与当前 PV 对象中的 `Spec.ClaimRef.UID` 不相等，相关代码如下：

``` go
		if claim != nil && claim.UID != volume.Spec.ClaimRef.UID {
			// The claim that the PV was pointing to was deleted, and another
			// with the same name created.
			klog.V(4).Infof("synchronizing PersistentVolume[%s]: claim %s has different UID, the old one must have been deleted", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef))
			// Treat the volume as bound to a missing claim.
			claim = nil
		}
```

这种情况会发生是因为用户将原先的绑定好的 PVC 对象删除掉然后立即创建了一个新的同名的 PVC 对象。这种在处理时，会将 PVC 对象置空，也就意味着原来绑定的 PVC 对象已经被删除了，这样的处理逻辑也符合实际的情况。

下面根据找到的 PVC 对象，分四种情况分别进行处理。

1. PVC 为空，表明与当前 PV 绑定的 PVC 对象已被删除。
2. 绑定进行了一半，即 PV 的 `Spec.ClaimRef` 已经被正确设置，但对应的 PVC 的 `Spec.VolumeName` 为空。也就意味着完成了从 PV 到 PVC 的绑定，但未完成从 PVC 到 PV 的绑定。
3. 已正确绑定。
4. 发生了错误的绑定，即对应的 PVC 的 `Spec.VolumeName` 为与当前 PV 对象不符。

#### 情景一、对应的 PVC 已被删除 ####

这种情况是根据当前 PV 的 `Spec.ClaimRef` 找不到对应的 PVC 对象，表明与当前 PV 绑定的 PVC 对象已被删除。

``` go
		if claim == nil {
			// If we get into this block, the claim must have been deleted;
			// NOTE: reclaimVolume may either release the PV back into the pool or
			// recycle it or do nothing (retain)

			// Do not overwrite previous Failed state - let the user see that
			// something went wrong, while we still re-try to reclaim the
			// volume.
			if volume.Status.Phase != v1.VolumeReleased && volume.Status.Phase != v1.VolumeFailed {
				// Also, log this only once:
				klog.V(2).Infof("volume %q is released and reclaim policy %q will be executed", volume.Name, volume.Spec.PersistentVolumeReclaimPolicy)
				if volume, err = ctrl.updateVolumePhase(volume, v1.VolumeReleased, ""); err != nil {
					// Nothing was saved; we will fall back into the same condition
					// in the next call to this method
					return err
				}
			}

			if err = ctrl.reclaimVolume(volume); err != nil {
				// Release failed, we will fall back into the same condition
				// in the next call to this method
				return err
			}
			return nil
```

进行处理时，会将 PV 对象更新为 `Released` 状态，然后在 `reclaimVolume()` 中根据 PV 的回收策略进一步执行。注意如果 PV 是 `Failed` 状态的时候不会将其更新为 `Released` 状态。因为在底层存储不可用等情况下才会进入 `Failed` 状态，用户需要知道这些情况以便人工介入。

下面来分析 `reclaimVolume()`。这个函数会根据 PV 回收策略执行相应的操作。

``` go
func (ctrl *PersistentVolumeController) reclaimVolume(volume *v1.PersistentVolume) error {
	if migrated := volume.Annotations[pvutil.AnnMigratedTo]; len(migrated) > 0 {
		// PV is Migrated. The PV controller should stand down and the external
		// provisioner will handle this PV
		return nil
	}
```

首先，检测 PV 的 Annotations ，如果包含有 `pv.kubernetes.io/migrated-to`，它应该由对应的 CSI Driver 动态地进行处理，因此这里直接返回。

接下来判断回收策略。

```
	switch volume.Spec.PersistentVolumeReclaimPolicy {
	case v1.PersistentVolumeReclaimRetain:
		klog.V(4).Infof("reclaimVolume[%s]: policy is Retain, nothing to do", volume.Name)

	case v1.PersistentVolumeReclaimRecycle:
		klog.V(4).Infof("reclaimVolume[%s]: policy is Recycle", volume.Name)
		opName := fmt.Sprintf("recycle-%s[%s]", volume.Name, string(volume.UID))
		ctrl.scheduleOperation(opName, func() error {
			ctrl.recycleVolumeOperation(volume)
			return nil
		})

	case v1.PersistentVolumeReclaimDelete:
		klog.V(4).Infof("reclaimVolume[%s]: policy is Delete", volume.Name)
		opName := fmt.Sprintf("delete-%s[%s]", volume.Name, string(volume.UID))
		// create a start timestamp entry in cache for deletion operation if no one exists with
		// key = volume.Name, pluginName = provisionerName, operation = "delete"
		ctrl.operationTimestamps.AddIfNotExist(volume.Name, ctrl.getProvisionerNameFromVolume(volume), "delete")
		ctrl.scheduleOperation(opName, func() error {
			_, err := ctrl.deleteVolumeOperation(volume)
			if err != nil {
				// only report error count to "volume_operation_total_errors"
				// latency reporting will happen when the volume get finally
				// deleted and a volume deleted event is captured
				metrics.RecordMetric(volume.Name, &ctrl.operationTimestamps, err)
			}
			return err
		})

	default:
		// Unknown PersistentVolumeReclaimPolicy
		if _, err := ctrl.updateVolumePhaseWithEvent(volume, v1.VolumeFailed, v1.EventTypeWarning, "VolumeUnknownReclaimPolicy", "Volume has unrecognized PersistentVolumeReclaimPolicy"); err != nil {
			return err
		}
	}
	return nil
}
```

PV 的回收策略包括三种：`Retain`、`Recycle` 或者 `Delete`。

- **`Retain`**。意味着不做任何操作，代码中也只会简单打印一条日志。
- **`Recycle`**。会执行一个异步的操作去清理 PV 中的数据。具体执行的函数为 `ctrl.recycleVolumeOperation()`。
- **`Delete`**。则表明要删除此 PV 对应和它底层对应的存储，具体执行函数为 `deleteVolumeOperation()`。

如果回收策略不属于以上三种，说明发生了错误，将 PV 状态更新为 `Failed`。

下面来详细看一下后两种情况，即清理 PV 和删除 PV 的操作。

##### PV 清理操作 #####

当 PV 回收策略是 `Recycle`的情况下，会执行一个异步的操作去清理 PV 中的数据，然后将 PV 更新为 `Available` 状态，以便后续与其它 PVC 重新绑定和使用。具体的逻辑在 `recycleVolumeOperation()` 中。

``` go
func (ctrl *PersistentVolumeController) recycleVolumeOperation(volume *v1.PersistentVolume) {
    ...
	newVolume, err := ctrl.kubeClient.CoreV1().PersistentVolumes().Get(context.TODO(), volume.Name, metav1.GetOptions{})
	if err != nil {
		klog.V(3).Infof("error reading persistent volume %q: %v", volume.Name, err)
		return
	}
	needsReclaim, err := ctrl.isVolumeReleased(newVolume)
	if err != nil {
		klog.V(3).Infof("error reading claim for volume %q: %v", volume.Name, err)
		return
	}
	if !needsReclaim {
		klog.V(3).Infof("volume %q no longer needs recycling, skipping", volume.Name)
		return
	}
	pods, used, err := ctrl.isVolumeUsed(newVolume)
	if err != nil {
		klog.V(3).Infof("can't recycle volume %q: %v", volume.Name, err)
		return
	}

	// Verify the claim is in cache: if so, then it is a different PVC with the same name
	// since the volume is known to be released at this moment. The new (cached) PVC must use
	// a different PV -- we checked that the PV is unused in isVolumeReleased.
	// So the old PV is safe to be recycled.
	claimName := claimrefToClaimKey(volume.Spec.ClaimRef)
	_, claimCached, err := ctrl.claims.GetByKey(claimName)
	if err != nil {
		klog.V(3).Infof("error getting the claim %s from cache", claimName)
		return
	}

	if used && !claimCached {
		msg := fmt.Sprintf("Volume is used by pods: %s", strings.Join(pods, ","))
		klog.V(3).Infof("can't recycle volume %q: %s", volume.Name, msg)
		ctrl.eventRecorder.Event(volume, v1.EventTypeNormal, events.VolumeFailedRecycle, msg)
		return
	}
```

首先会检测当前 PV 的状态，是否满足进行清理的条件，以及有没有被 Pod 引用等等。

```
	// Use the newest volume copy, this will save us from version conflicts on
	// saving.
	volume = newVolume

	// Find a plugin.
	spec := vol.NewSpecFromPersistentVolume(volume, false)
	plugin, err := ctrl.volumePluginMgr.FindRecyclablePluginBySpec(spec)
	if err != nil {
		// No recycler found. Emit an event and mark the volume Failed.
		if _, err = ctrl.updateVolumePhaseWithEvent(volume, v1.VolumeFailed, v1.EventTypeWarning, events.VolumeFailedRecycle, "No recycler plugin found for the volume!"); err != nil {
			klog.V(4).Infof("recycleVolumeOperation [%s]: failed to mark volume as failed: %v", volume.Name, err)
			// Save failed, retry on the next deletion attempt
			return
		}
		// Despite the volume being Failed, the controller will retry recycling
		// the volume in every syncVolume() call.
		return
	}
```

接着，用 spec 找到对应的 Volume Plugin，注意这个 Volume Plugin 必须实现 RecyclableVolumePlugin interface，因为 RecyclableVolumePlugin interface 的 `Recycle()` 定义了如何对 PV 进行数据清理从操作。

``` go
	// Plugin found
	recorder := ctrl.newRecyclerEventRecorder(volume)

	if err = plugin.Recycle(volume.Name, spec, recorder); err != nil {
		// Recycler failed
		strerr := fmt.Sprintf("Recycle failed: %s", err)
		if _, err = ctrl.updateVolumePhaseWithEvent(volume, v1.VolumeFailed, v1.EventTypeWarning, events.VolumeFailedRecycle, strerr); err != nil {
			klog.V(4).Infof("recycleVolumeOperation [%s]: failed to mark volume as failed: %v", volume.Name, err)
			// Save failed, retry on the next deletion attempt
			return
		}
		// Despite the volume being Failed, the controller will retry recycling
		// the volume in every syncVolume() call.
		return
	}
```

然后，调用 Volume Plugin 对应的 `Recycle()` 操作，执行真正的清理操作。这个由 Volume Plugin 自己进行实现，常规的方式是将这个 PV 重新绑定到一个 Pod 上，例如 busybox，然后执行类似于 `rm -rf /pv/*` 的操作。

``` go
	klog.V(2).Infof("volume %q recycled", volume.Name)
	// Send an event
	ctrl.eventRecorder.Event(volume, v1.EventTypeNormal, events.VolumeRecycled, "Volume recycled")
	// Make the volume available again
	if err = ctrl.unbindVolume(volume); err != nil {
		// Oops, could not save the volume and therefore the controller will
		// recycle the volume again on next update. We _could_ maintain a cache
		// of "recently recycled volumes" and avoid unnecessary recycling, this
		// is left out as future optimization.
		klog.V(3).Infof("recycleVolumeOperation [%s]: failed to make recycled volume 'Available' (%v), we will recycle the volume again", volume.Name, err)
		return
	}
	return
}
```

最后，执行 PV 的 umound 操作，代码如下：

``` go
// unbindVolume rolls back previous binding of the volume. This may be necessary
// when two controllers bound two volumes to single claim - when we detect this,
// only one binding succeeds and the second one must be rolled back.
// This method updates both Spec and Status.
// It returns on first error, it's up to the caller to implement some retry
// mechanism.
func (ctrl *PersistentVolumeController) unbindVolume(volume *v1.PersistentVolume) error {
	klog.V(4).Infof("updating PersistentVolume[%s]: rolling back binding from %q", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef))

	// Save the PV only when any modification is necessary.
	volumeClone := volume.DeepCopy()

	if metav1.HasAnnotation(volume.ObjectMeta, pvutil.AnnBoundByController) {
		// The volume was bound by the controller.
		volumeClone.Spec.ClaimRef = nil
		delete(volumeClone.Annotations, pvutil.AnnBoundByController)
		if len(volumeClone.Annotations) == 0 {
			// No annotations look better than empty annotation map (and it's easier
			// to test).
			volumeClone.Annotations = nil
		}
	} else {
		// The volume was pre-bound by user. Clear only the binding UID.
		volumeClone.Spec.ClaimRef.UID = ""
	}

	newVol, err := ctrl.kubeClient.CoreV1().PersistentVolumes().Update(context.TODO(), volumeClone, metav1.UpdateOptions{})
	if err != nil {
		klog.V(4).Infof("updating PersistentVolume[%s]: rollback failed: %v", volume.Name, err)
		return err
	}
```

如果有 `pv.kubernetes.io/bound-by-controller` Annotation，则表明它是由 Controller 进行绑定的，这时需要删除这个 Annotation；如果没有这个 Annotation，则表明是由用户手动指定与 PVC 的绑定关系的，这时需要将引用的 PVC UID 置空。

``` go
	_, err = ctrl.storeVolumeUpdate(newVol)
	if err != nil {
		klog.V(4).Infof("updating PersistentVolume[%s]: cannot update internal cache: %v", volume.Name, err)
		return err
	}
	klog.V(4).Infof("updating PersistentVolume[%s]: rolled back", newVol.Name)

	// Update the status
	_, err = ctrl.updateVolumePhase(newVol, v1.VolumeAvailable, "")
	return err
}
```

接着更新本地的 PV 魂村，然后将 PV 更新为 `Available` 状态。

##### PV 删除操作 #####

当 PV 回收策略是 `Delete`的情况下，会执行一个异步的操作去删除 PV 以及对应的底层存储。具体的逻辑在 `deleteVolumeOperation()` 中。

``` go
// deleteVolumeOperation deletes a volume. This method is running in standalone
// goroutine and already has all necessary locks.
func (ctrl *PersistentVolumeController) deleteVolumeOperation(volume *v1.PersistentVolume) (string, error) {
	klog.V(4).Infof("deleteVolumeOperation [%s] started", volume.Name)

	// This method may have been waiting for a volume lock for some time.
	// Previous deleteVolumeOperation might just have saved an updated version, so
	// read current volume state now.
	newVolume, err := ctrl.kubeClient.CoreV1().PersistentVolumes().Get(context.TODO(), volume.Name, metav1.GetOptions{})
	if err != nil {
		klog.V(3).Infof("error reading persistent volume %q: %v", volume.Name, err)
		return "", nil
	}

	if newVolume.GetDeletionTimestamp() != nil {
		klog.V(3).Infof("Volume %q is already being deleted", volume.Name)
		return "", nil
	}
	needsReclaim, err := ctrl.isVolumeReleased(newVolume)
	if err != nil {
		klog.V(3).Infof("error reading claim for volume %q: %v", volume.Name, err)
		return "", nil
	}
	if !needsReclaim {
		klog.V(3).Infof("volume %q no longer needs deletion, skipping", volume.Name)
		return "", nil
	}
```

首先，做一些预检测，如果 PV 已经在删除了或者 PV 还被现有的 PVC 引用等情况，则不进行处理，直接返回。

``` go
	pluginName, deleted, err := ctrl.doDeleteVolume(volume)
	if err != nil {
		// Delete failed, update the volume and emit an event.
		klog.V(3).Infof("deletion of volume %q failed: %v", volume.Name, err)
		if volerr.IsDeletedVolumeInUse(err) {
			// The plugin needs more time, don't mark the volume as Failed
			// and send Normal event only
			ctrl.eventRecorder.Event(volume, v1.EventTypeNormal, events.VolumeDelete, err.Error())
		} else {
			// The plugin failed, mark the volume as Failed and send Warning
			// event
			if _, err := ctrl.updateVolumePhaseWithEvent(volume, v1.VolumeFailed, v1.EventTypeWarning, events.VolumeFailedDelete, err.Error()); err != nil {
				klog.V(4).Infof("deleteVolumeOperation [%s]: failed to mark volume as failed: %v", volume.Name, err)
				// Save failed, retry on the next deletion attempt
				return pluginName, err
			}
		}

		// Despite the volume being Failed, the controller will retry deleting
		// the volume in every syncVolume() call.
		return pluginName, err
	}
	if !deleted {
		// The volume waits for deletion by an external plugin. Do nothing.
		return pluginName, nil
	}
```

接着，调用 `doDeleteVolume()` 执行底层存储的删除操作，如果删除失败，则可能 PV 仍在使用，这时会直接返回，如果 PV 没有被使用，则将 PV 更新为 `Fail` 状态，表示在删除底层存储时发生了某些错误。

``` go
	klog.V(4).Infof("deleteVolumeOperation [%s]: success", volume.Name)
	// Delete the volume
	if err = ctrl.kubeClient.CoreV1().PersistentVolumes().Delete(context.TODO(), volume.Name, metav1.DeleteOptions{}); err != nil {
		// Oops, could not delete the volume and therefore the controller will
		// try to delete the volume again on next update. We _could_ maintain a
		// cache of "recently deleted volumes" and avoid unnecessary deletion,
		// this is left out as future optimization.
		klog.V(3).Infof("failed to delete volume %q from database: %v", volume.Name, err)
		return pluginName, nil
	}
	return pluginName, nil
}
```

最后，从 Kube API Server 中删除 PV 对象本身。

现在来分析刚才提到的 `doDeleteVolume()`，它用来执行底层存储的删除操作。

``` go
// doDeleteVolume finds appropriate delete plugin and deletes given volume, returning
// the volume plugin name. Also, it returns 'true', when the volume was deleted and
// 'false' when the volume cannot be deleted because the deleter is external. No
// error should be reported in this case.
func (ctrl *PersistentVolumeController) doDeleteVolume(volume *v1.PersistentVolume) (string, bool, error) {
	klog.V(4).Infof("doDeleteVolume [%s]", volume.Name)
	var err error

	plugin, err := ctrl.findDeletablePlugin(volume)
	if err != nil {
		return "", false, err
	}
	if plugin == nil {
		// External deleter is requested, do nothing
		klog.V(3).Infof("external deleter for volume %q requested, ignoring", volume.Name)
		return "", false, nil
	}
```

先使用 spec 获取 DeletableVolumePlugin 这个 Volume Plugin 对象，它的 `NewDeleter()` 返回一个 `Deleter` 对象，而这个 `Deleter` 对象的 `Delete()` 会执行真正的删除存储的操作，相关数据结构如下：

``` go
type DeletableVolumePlugin interface {
	VolumePlugin
	NewDeleter(spec *Spec) (Deleter, error)
}

type Deleter interface {
	Volume
	Delete() error
}
```

回到 `doDeleteVolume()`

	// Plugin found
	pluginName := plugin.GetPluginName()
	klog.V(5).Infof("found a deleter plugin %q for volume %q", pluginName, volume.Name)
	spec := vol.NewSpecFromPersistentVolume(volume, false)
	deleter, err := plugin.NewDeleter(spec)
	if err != nil {
		// Cannot create deleter
		return pluginName, false, fmt.Errorf("Failed to create deleter for volume %q: %v", volume.Name, err)
	}
```

获取到 Volume Plugin 后，通过它获取 `Deleter` 对象。


``` go
	opComplete := util.OperationCompleteHook(pluginName, "volume_delete")
	err = deleter.Delete()
	opComplete(&err)
	if err != nil {
		// Deleter failed
		return pluginName, false, err
	}

	klog.V(2).Infof("volume %q deleted", volume.Name)
	return pluginName, true, nil
}
```

然后指定 `Deleter.Delete()` 进行删除底层的存储。这个函数需要由 Volume Plugin 自己实现。

这里在删除前会执行相关的 hook 调用，主要是与 Metrics 相关。

#### 情景二、绑定完成一半 ####

绑定进行了一半，即 PV 的 `Spec.ClaimRef` 已经被正确设置，但对应的 PVC 的 `Spec.VolumeName` 为空。表明从 PV 到 PVC 的绑定已经完成，但是从 PVC 到 PV 的绑定还未完成。

``` go
		} else if claim.Spec.VolumeName == "" {
			if pvutil.CheckVolumeModeMismatches(&claim.Spec, &volume.Spec) {
				// Binding for the volume won't be called in syncUnboundClaim,
				// because findBestMatchForClaim won't return the volume due to volumeMode mismatch.
				volumeMsg := fmt.Sprintf("Cannot bind PersistentVolume to requested PersistentVolumeClaim %q due to incompatible volumeMode.", claim.Name)
				ctrl.eventRecorder.Event(volume, v1.EventTypeWarning, events.VolumeMismatch, volumeMsg)
				claimMsg := fmt.Sprintf("Cannot bind PersistentVolume %q to requested PersistentVolumeClaim due to incompatible volumeMode.", volume.Name)
				ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.VolumeMismatch, claimMsg)
				// Skipping syncClaim
				return nil
			}
```

首先进行模式检测，比较 PV 和 PVC 是否都是作为文件系统或者块设备使用，如果不一致，则无法对其进行绑定，直接返回。

``` go
            ...

			ctrl.claimQueue.Add(claimToClaimKey(claim))
			return nil
		} else if claim.Spec.VolumeName == volume.Name {
```

接着，将 PVC 加入到 PVC 队列中，以便完成 PVC 到 PV 的绑定操作。

#### 情景三、已正确绑定 ####

这种情况，说明从 PV 到 PVC 以及从 PVC 到 PV 的绑定都已经完成，只需要将 PV 状态更新为 `Bound`。

``` go
			// Volume is bound to a claim properly, update status if necessary
			klog.V(4).Infof("synchronizing PersistentVolume[%s]: all is bound", volume.Name)
			if _, err = ctrl.updateVolumePhase(volume, v1.VolumeBound, ""); err != nil {
				// Nothing was saved; we will fall back into the same
				// condition in the next call to this method
				return err
			}
			return nil
```

#### 情景四、发生了错误的绑定 ####

发生了错误的绑定，即 PV 对应的 PVC 的 `Spec.VolumeName` 为与当前 PV 对象不符。

``` go
		} else {
			// Volume is bound to a claim, but the claim is bound elsewhere
			if metav1.HasAnnotation(volume.ObjectMeta, pvutil.AnnDynamicallyProvisioned) && volume.Spec.PersistentVolumeReclaimPolicy == v1.PersistentVolumeReclaimDelete {
				// This volume was dynamically provisioned for this claim. The
				// claim got bound elsewhere, and thus this volume is not
				// needed. Delete it.
				// Mark the volume as Released for external deleters and to let
				// the user know. Don't overwrite existing Failed status!
				if volume.Status.Phase != v1.VolumeReleased && volume.Status.Phase != v1.VolumeFailed {
					// Also, log this only once:
					klog.V(2).Infof("dynamically volume %q is released and it will be deleted", volume.Name)
					if volume, err = ctrl.updateVolumePhase(volume, v1.VolumeReleased, ""); err != nil {
						// Nothing was saved; we will fall back into the same condition
						// in the next call to this method
						return err
					}
				}
				if err = ctrl.reclaimVolume(volume); err != nil {
					// Deletion failed, we will fall back into the same condition
					// in the next call to this method
					return err
				}
				return nil
			} else {
```

如果 PV 是动态管理且 PV 回收策略是 `Delete`，说明当前 PV 绑定已经不再需要了，就通过 `reclaimVolume()` 执行 PV 的删除操作。关于 `reclaimVolume()` 请参见 [情景一、对应的 PVC 已被删除](#情景一、对应的-PVC-已被删除) 中的分析。

<TODO>哪种操作可以导致这种情况？对于非 `Delete` 回收策略的 PV 如何处理？</TODO>

``` go
				// Volume is bound to a claim, but the claim is bound elsewhere
				// and it's not dynamically provisioned.
				if metav1.HasAnnotation(volume.ObjectMeta, pvutil.AnnBoundByController) {
					// This is part of the normal operation of the controller; the
					// controller tried to use this volume for a claim but the claim
					// was fulfilled by another volume. We did this; fix it.
					klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume is bound by controller to a claim that is bound to another volume, unbinding", volume.Name)
					if err = ctrl.unbindVolume(volume); err != nil {
						return err
					}
					return nil
				} else {
					// The PV must have been created with this ptr; leave it alone.
					klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume is bound by user to a claim that is bound to another volume, waiting for the claim to get unbound", volume.Name)
					// This just updates the volume phase and clears
					// volume.Spec.ClaimRef.UID. It leaves the volume pre-bound
					// to the claim.
					if err = ctrl.unbindVolume(volume); err != nil {
						return err
					}
					return nil
				}
			}
		}
```

接着，对 PV 执行 unbind 操作，不管 PV 是动态创建还是由用户手动创建，执行的操作都是一样的。具体逻辑可参加 [PV 清理操作](#PV-清理操作) 中对 `unbindVolume()` 的分析。
