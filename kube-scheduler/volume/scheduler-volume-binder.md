# Volume 相关操作在调度过程中的执行时机 #

![volume-scheduler](volume-scheduler.png)

这是 Kubenetes 调度框架图，每个阶段都有可以插入自定义的逻辑进而控制整个调度的流程。

其中涉及到四个与 Volume 相关的操作，其中 `DeletePodBindings()` 是调度失败后执行的，没有在图中标示出来。

接下来会针对这四个操作的执行时机详细分析。

## FindPodVolumes() ##

`FindPodVolumes()` 是在 **预选** 阶段执行的，这一阶段主要筛选出哪些节点可以运行当前的 Pod。而这里的 `FindPodVolumes()` 会作为其中的一个筛选条件。它会检测当前 Pod 上所有的 PVC 对象能否在当前节点上运行，即 PVC 和对应的 PV 能否运行在当前节点上。

在预选阶段执行的插件需要实现下面这个 interface 以及内部的 `Filter()` 函数。

``` go
type FilterPlugin interface {
	Plugin
	Filter(ctx context.Context, state *CycleState, pod *v1.Pod, nodeInfo *schedulernodeinfo.NodeInfo) *Status
}
```

`FindPodVolumes()` 对应的 Filter 名称为 "VolumeBinding"，代码位于 `pkg/scheduler/framework/plugins/volumebinding`。

``` go
type VolumeBinding struct {
	binder scheduling.SchedulerVolumeBinder
}

...
const Name = "VolumeBinding"

...

func (pl *VolumeBinding) Filter(ctx context.Context, cs *framework.CycleState, pod *v1.Pod, nodeInfo *schedulernodeinfo.NodeInfo) *framework.Status {
    ...
	if !podHasPVCs(pod) {
		return nil
	}

	reasons, err := pl.binder.FindPodVolumes(pod, node)
    ...
}

// New initializes a new plugin with volume binder and returns it.
func New(_ *runtime.Unknown, fh framework.FrameworkHandle) (framework.Plugin, error) {
	return &VolumeBinding{
		binder: fh.VolumeBinder(),
	}, nil
}
```

可以看出 VolumeBinding Filter 实现了 `Filter()` 函数。`Filter()` 执行时首先会检测当前 Pod 是否关联有 PVC 对象，如果没有的话则不需要处理，否则的话就会调用 `FindPodVolumes()` 进行验证。

VolumeBinding Filter 会在创建 Scheduler 对象的时候进行注册。

``` go
func NewInTreeRegistry() framework.Registry {
	return framework.Registry{
        ...
		volumebinding.Name:                         volumebinding.New,
        ...
	}
}
```

然后在预选节点会执行其中的 `Filter()` 函数。相关的函数调用栈如下：

``` go
scheduleOne()
  sched.Algorithm.Schedule()
    g.findNodesThatFitPod()
      findNodesThatPassFilters()
        g.podPassesFiltersOnNode()
          prof.RunFilterPlugins()
            RunFilterPlugins()
              runFilterPlugin()
                Filter()
```

其中更加详细的分析请参见 Kube-Scheduler 的相关章节。
