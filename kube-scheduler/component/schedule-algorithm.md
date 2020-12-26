## Schedule Algorithm ##

`Algorithm` 成员的类型是 `core.ScheduleAlgorithm`，它是 interface 类型。这个成员的使用有一些历史原因，在引入 scheduler framework 之前主要是使用它来对 Pod 进行调度的。

``` go
type ScheduleAlgorithm interface {
	Schedule(context.Context, *profile.Profile, *framework.CycleState, *v1.Pod) (scheduleResult ScheduleResult, err error)
	Preempt(context.Context, *profile.Profile, *framework.CycleState, *v1.Pod, error) (selectedNode *v1.Node, preemptedPods []*v1.Pod, cleanupNominatedPods []*v1.Pod, err error)
    ...
}
```

其中的 `Schedule()` 会执行传统的调度操作，即预选和优选。在引入了 scheduler framework 之后，对整个函数进行了重构，目前它会执行 `PreFilter`、`Filter`、`PreScore`、`Score` 和 `Normalize Score` 等插入点的工作，在具体执行时会通过 scheduler framework 来调用这些插入点的插件完成具体的工作。

其中的 `Preempt()` 会执行具体的抢占操作。

实现了 `core.ScheduleAlgorithm` 接口的对象是 `genericScheduler`。

``` go
type genericScheduler struct {
	cache                    internalcache.Cache
	schedulingQueue          internalqueue.SchedulingQueue
	extenders                []SchedulerExtender
	nodeInfoSnapshot         *internalcache.Snapshot
	pvcLister                corelisters.PersistentVolumeClaimLister
	pdbLister                policylisters.PodDisruptionBudgetLister
	disablePreemption        bool
	percentageOfNodesToScore int32
	enableNonPreempting      bool
	nextStartNodeIndex       int
}
```

关于它是如何实现 `Schedule()` 和 `Preempt()` 的，请参见 [调度过程分析](../../scratch/kube-scheduler/scheduling.md) 和 [抢占](../../scratch/kube-scheduler/preemption.md)。
