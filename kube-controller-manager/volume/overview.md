# PersistentVolumeController #

在 Kube Controller Manager 中有一个名为 PersistentVolumeController 的 Controller 用于维护 PV 和 PVC 的状态。代码位于 `pkg/controller/volume/persistentvolume`。它会对 PV 和 PVC 对象分别进行处理以达到用户的最终期望状态。

下面先来看它的内部结构：

``` go
type PersistentVolumeController struct {
    ...
	volumeQueue *workqueue.Type
	claimQueue  *workqueue.Type
    ...
}
```

在 PersistentVolumeController 中有两个队列分别用来存储需要处理的 PV 和 PVC 对象。

在创建 PersistentVolumeController 的时候需要传入一个 ControllerParameters 对象作为参数。

``` go
type ControllerParameters struct {
    ...
	VolumeInformer            coreinformers.PersistentVolumeInformer
	ClaimInformer             coreinformers.PersistentVolumeClaimInformer
    ...
}
```

其中有一个 PersistentVolumeInformer 和一个 PersistentVolumeClaimInformer，分别用于 watch PV 和 PVC 对象的改动。

下面是创建 PersistentVolumeController 对象的函数：

``` go
func NewController(p ControllerParameters) (*PersistentVolumeController, error) {
    ...
	controller := &PersistentVolumeController{
        ...
	}

    ...
	p.VolumeInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    func(obj interface{}) { controller.enqueueWork(controller.volumeQueue, obj) },
			UpdateFunc: func(oldObj, newObj interface{}) { controller.enqueueWork(controller.volumeQueue, newObj) },
			DeleteFunc: func(obj interface{}) { controller.enqueueWork(controller.volumeQueue, obj) },
		},
	)
    ...
	p.ClaimInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    func(obj interface{}) { controller.enqueueWork(controller.claimQueue, obj) },
			UpdateFunc: func(oldObj, newObj interface{}) { controller.enqueueWork(controller.claimQueue, newObj) },
			DeleteFunc: func(obj interface{}) { controller.enqueueWork(controller.claimQueue, obj) },
		},
	)
    ...
	return controller, nil
}
```

可以看出它启动后，会对 PV 和 PVC 对象进行 watch，发现 PV 和 PVC 对象有改动时，分别将其加入上文提到的 PersistentVolumeController 对象的 PV 和 PVC 队列中。

``` go
func (ctrl *PersistentVolumeController) Run(stopCh <-chan struct{}) {
    ...
	go wait.Until(ctrl.resync, ctrl.resyncPeriod, stopCh)
	go wait.Until(ctrl.volumeWorker, time.Second, stopCh)
	go wait.Until(ctrl.claimWorker, time.Second, stopCh)
    ...
	<-stopCh
}
```

在随后执行 `PersistentVolumeController.Run()` 时，会分别启动两个 goroutine，`volumeWorker` 和 `claimWorker` 从 PV 和 PVC 队列中取出数据进行处理。

和这两个同时启动的还有另外一个 goroutine `resync`，它会周期性地进行执行，这个执行周期由 Kube Controller Manager 的 `--pvclaimbinder-sync-period` 参数指定，默认为15秒。它的内容如下：

``` go
func (ctrl *PersistentVolumeController) resync() {
	klog.V(4).Infof("resyncing PV controller")

	pvcs, err := ctrl.claimLister.List(labels.NewSelector())
	if err != nil {
		klog.Warningf("cannot list claims: %s", err)
		return
	}
	for _, pvc := range pvcs {
		ctrl.enqueueWork(ctrl.claimQueue, pvc)
	}

	pvs, err := ctrl.volumeLister.List(labels.NewSelector())
	if err != nil {
		klog.Warningf("cannot list persistent volumes: %s", err)
		return
	}
	for _, pv := range pvs {
		ctrl.enqueueWork(ctrl.volumeQueue, pv)
	}
}
```

这个函数的逻辑也非常简单，就是将所有的 PV 和 PVC 对象分别加入到各自的队列中进行处理。因此它的作用就是周期性地同步所有的 PV 和 PVC 对象。

下一节我们针对 `volumeWorker` 和 `claimWorker 这两个 goroutine 分别进行分析，看 PersistentVolumeController 是如何处理 PV 和 PVC 对象的。

## 关于绑定操作 ##

<TODO></TODO>

## 关于 Annotation ##

<TODO></TODO>
