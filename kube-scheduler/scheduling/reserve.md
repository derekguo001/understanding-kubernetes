# Reserve 扩展点的执行 #

Reserve 扩展点的作用是在 Pod 与节点绑定之前，做一些相关附属资源的预留，例如 PVC。然后可以在 PreBind 扩展点对这些已经预留的资源进行预绑定，再在 Bind 扩展点对 Pod 和节点之间完成绑定。

## 数据结构 ##

``` go
type ReservePlugin interface {
	Plugin
	Reserve(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) *Status
	Unreserve(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string)
}
```

实现了 Reserve 扩展点的插件必须同时实现 `Reserve()` 和 `Unreserve()` 函数。前者就是预留资源的作用；而后者相当于在执行失败的时候执行相应的回退操作。例如将预留的 PVC 等资源从预留缓存中删除，即不再预留这些资源。

在 Reserver 以及后续所有操作执行失败时都需要执行 `Unreserve()`。

## Assume ##

在执行 Reserve 扩展点之前，有一个 Assume 的操作。

``` go
	assumedPodInfo := podInfo.DeepCopy()
	assumedPod := assumedPodInfo.Pod

	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
	if err != nil {
        ...
		sched.recordSchedulingFailure(fwk, assumedPodInfo, err, SchedulerError, "")
		return
	}
```

首先将 Pod 进行深度复制，在后续的所有扩展点执行过程中，修改的都是复制之后的 Pod 对象，这样在发生错误的时候就可以保证原始 Pod 信息仍然维持原样。

``` go
func (sched *Scheduler) assume(assumed *v1.Pod, host string) error {
	assumed.Spec.NodeName = host

	if err := sched.SchedulerCache.AssumePod(assumed); err != nil {
		klog.Errorf("scheduler cache AssumePod failed: %v", err)
		return err
	}
	// if "assumed" is a nominated pod, we should remove it from internal cache
	if sched.SchedulingQueue != nil {
		sched.SchedulingQueue.DeleteNominatedPodIfExists(assumed)
	}

	return nil
}
```

然后执行 `AssumePod()`，作用是将深度复制之后的 Pod加入到缓存中，以便后续的绑定等操作可以异步执行。


## Reserve 扩展点 ##

执行完 Assume 操作后，会接着执行 Reserve 扩展点。

``` go
	// Run the Reserve method of reserve plugins.
	if sts := fwk.RunReservePluginsReserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
        ...
		fwk.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
			klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
		}
		sched.recordSchedulingFailure(fwk, assumedPodInfo, sts.AsError(), SchedulerError, "")
		return
	}
```

调用 `RunReservePluginsReserve()` 来预留资源，如果执行失败，则需要执行 `Unreserve()` 将预先消费的资源释放掉，并将 Pod 从调度队列中删除，然后返回错误。

``` go
func (f *frameworkImpl) RunReservePluginsReserve(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (status *framework.Status) {
    ...
	for _, pl := range f.reservePlugins {
		status = f.runReservePluginReserve(ctx, pl, state, pod, nodeName)
		if !status.IsSuccess() {
			err := status.AsError()
			klog.ErrorS(err, "Failed running Reserve plugin", "plugin", pl.Name(), "pod", klog.KObj(pod))
			return framework.AsStatus(fmt.Errorf("running Reserve plugin %q: %w", pl.Name(), err))
		}
	}
	return nil
}

func (f *frameworkImpl) runReservePluginReserve(ctx context.Context, pl framework.ReservePlugin, state *framework.CycleState, pod *v1.Pod, nodeName string) *framework.Status {
	if !state.ShouldRecordPluginMetrics() {
		return pl.Reserve(ctx, state, pod, nodeName)
	}
	startTime := time.Now()
	status := pl.Reserve(ctx, state, pod, nodeName)
	f.metricsRecorder.observePluginDurationAsync(reserve, pl.Name(), status, metrics.SinceInSeconds(startTime))
	return status
}
```

会遍历所有实现了 Reserve 扩展点的插件，调用其中的 `Reserve()` 函数。

目前，只有一个内置插件实现了 Reserve 扩展点，即 VolumeBinding 插件。

``` go
func (pl *VolumeBinding) Reserve(ctx context.Context, cs *framework.CycleState, pod *v1.Pod, nodeName string) *framework.Status {
	state, err := getStateData(cs)
	if err != nil {
		return framework.AsStatus(err)
	}
	// we don't need to hold the lock as only one node will be reserved for the given pod
	podVolumes, ok := state.podVolumesByNode[nodeName]
	if ok {
		allBound, err := pl.Binder.AssumePodVolumes(pod, nodeName, podVolumes)
		if err != nil {
			return framework.AsStatus(err)
		}
		state.allBound = allBound
	} else {
		// may not exist if the pod does not reference any PVC
		state.allBound = true
	}
	return nil
}
```

通过调用 `pl.Binder.AssumePodVolumes(pod, nodeName, podVolumes)` 将当前 Pod 所使用的的所有 PVC 和对应的 PV 在缓存中进行绑定。注意这里的绑定只是发生在缓存中的，并未提交到 Kubernetes API Server 进行持久化。

``` go
func (pl *VolumeBinding) Unreserve(ctx context.Context, cs *framework.CycleState, pod *v1.Pod, nodeName string) {
	s, err := getStateData(cs)
	if err != nil {
		return
	}
	// we don't need to hold the lock as only one node may be unreserved
	podVolumes, ok := s.podVolumesByNode[nodeName]
	if !ok {
		return
	}
	pl.Binder.RevertAssumedPodVolumes(podVolumes)
	return
}

func (b *volumeBinder) RevertAssumedPodVolumes(podVolumes *PodVolumes) {
	b.revertAssumedPVs(podVolumes.StaticBindings)
	b.revertAssumedPVCs(podVolumes.DynamicProvisions)
}

func (b *volumeBinder) revertAssumedPVs(bindings []*BindingInfo) {
	for _, BindingInfo := range bindings {
		b.pvCache.Restore(BindingInfo.pv.Name)
	}
}

func (b *volumeBinder) revertAssumedPVCs(claims []*v1.PersistentVolumeClaim) {
	for _, claim := range claims {
		b.pvcCache.Restore(getPVCName(claim))
	}
}
```

这是 VolumeBinding 插件的 `Unreserve()` 函数的实现。如果后续的 Pod 调度过程失败，则只需要将缓存中的数据重置即可。
