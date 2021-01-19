# 抢占功能概述 #

抢占功能现在已经抽象到了名为 `DefaultPreemption` 的调度插件中，调度框架通过执行 PostFilter 扩展点来调用 `DefaultPreemption` 的抢占调度功能。

在代码层面，在 Kubernetes Scheduler 调度框架的代码中仍然有一些和抢占功能耦合得比较紧的资源对象，一起协助 `DefaultPreemption` 插件来实现抢占调度功能。

最重要的是在调度框架中的 `preemptHandle framework.PreemptHandle` 成员。

``` go
type frameworkImpl struct {
    ...
	preemptHandle framework.PreemptHandle
    ...
}

type PreemptHandle interface {
	PodNominator
	PluginsRunner
	Extenders() []Extender
}
```

包含以下几个成员：

- Extenders() []Extender

  这是为了兼容旧的扩展方式的抢占功能而存在的，不会对其进行详细分析。

- PodNominator

  ``` go
  type PodNominator interface {
      AddNominatedPod(pod *v1.Pod, nodeName string)
      DeleteNominatedPodIfExists(pod *v1.Pod)
      UpdateNominatedPod(oldPod, newPod *v1.Pod)
      NominatedPodsForNode(nodeName string) []*v1.Pod
  }
  ```

  这是用来管理抢占 Pod 的一个缓存对象。

  - 当一个 Pod 完成抢占之后会通过 `AddNominatedPod()` 将其加入缓存中
  - 当调度器检测到 Pod 更新或者删除后，也会尝试在这个缓存中将其更新或者删除
  - 当 Pod 对其它优先级更低的 Pod 进行抢占时，会调用这里的 `NominatedPodsForNode()` 取得当前节点上也是通过抢占操作完成调度的其它优先级更低的 Pod，然后将这些 Pod 的 `Status.NominatedNodeName` 置空

- PluginsRunner

  ``` go
  type PluginsRunner interface {
      RunPreScorePlugins(context.Context, *CycleState, *v1.Pod, []*v1.Node) *Status
      RunScorePlugins(context.Context, *CycleState, *v1.Pod, []*v1.Node) (PluginToNodeScores, *Status)
      RunFilterPlugins(context.Context, *CycleState, *v1.Pod, *NodeInfo) PluginToStatus
      RunPreFilterExtensionAddPod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToAdd *v1.Pod, nodeInfo *NodeInfo) *Status
      RunPreFilterExtensionRemovePod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToRemove *v1.Pod, nodeInfo *NodeInfo) *Status
  }
  ```

  顾名思义，这是一个 Runner 对象，在进行抢占操作时，会通过这个对象执行模拟的抢占，这几个函数分别对应于 PreScore、Score、Filter等扩展点。注意最后的 `RunPreFilterExtensionAddPod()` 和 `RunPreFilterExtensionRemovePod()` 分别实现的是 PreFilter 扩展点的`额外的扩展功能`。

  ``` go
  type PreFilterPlugin interface {
      Plugin
      PreFilter(ctx context.Context, state *CycleState, p *v1.Pod) *Status
      PreFilterExtensions() PreFilterExtensions
  }
  ```

  这是实现 PreFilter 扩展点的插件需要实现的功能。其中第三个会返回一个 `PreFilterExtensions` interface。

  ``` go
  type PreFilterExtensions interface {
      AddPod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToAdd *v1.Pod, nodeInfo *NodeInfo) *Status
      RemovePod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToRemove *v1.Pod, nodeInfo *NodeInfo) *Status
  }
  ```

  这个 interface 实现了 `AddPod()` 和 `RemovePod()` 的功能，这是为了在实现模拟抢占过程中提供了额外的扩展点。上面的  `RunPreFilterExtensionAddPod()` 和 `RunPreFilterExtensionRemovePod()` 分别调用的就是这里的两个功能。

上面分析了抢占调度过程中使用到的核心对象，接下来会分成几节内容，对抢占的整个过程进行代码层面的分析。

具体而言，包括三个大的步骤。

- [第一步:找到所有可以抢占的目标节点](1.md)
- [第二步:选择一个最佳的抢占目标节点](2.md)
- [第三步:实施抢占动作](3.md)

