# Kubernetes Scheduler 概述 #

## 概述 ##

kubernetes Scheduler 的作用是将一个 Pod 调度到一个节点上，并将 Pod 与节点进行绑定。

kubernetes Scheduler 内部有一个调度框架，这个框架中内置了一些扩展点，如下图所示，我们可以通过在这些扩展点中插入需要执行的代码来对调度过程进行扩展。

![scheduler](https://raw.githubusercontent.com/kubernetes/enhancements/master/keps/sig-scheduling/624-scheduling-framework/scheduling-framework-extensions.png)

整个调度过程整体上可以分为 `调度周期` 和 `绑定周期` 两个大的阶段：前者包含：PreFilter、Filter、PostFilter、PreScore、Score、Normalize Score、Reserve、Permit，后者包括 WaitOnPermit、Prebind、Bind 和 PostBind。

如果调度成功，则在 `调度周期` 会筛选出一个合适的节点作为当前 Pod 的目标节点，然后在 `绑定周期` 会将当前 Pod 与选出的节点进行绑定。

绑定周期和调度周期是相互独立的，并且是异步的，在代码中表现为整个绑定周期的操作都是在一个单独的 goroutine 中执行的。

下面是这些过程的简单说明。

|扩展点名称|说明|
|---|---|
|Sort|对队列中的 Pod 进行排序，同时只能启用一个 Sort 扩展点的插件|
|PreFilter|对 Filter 扩展点的数据做一些预处理操作，然后将其存入缓存中待 Filter 扩展点执行的时候使用 |
|Filter|针对当前 Pod，对所有节点进行过滤，在这个扩展点会过滤掉那些不适合的节点|
|PostFilter|如果在 Filter 扩展点全部节点都被过滤掉了，没有合适的节点进行调度，才会执行 PostFilter 扩展点，如果启用了 Pod 抢占特性，那么会在这个扩展点进行抢占操作|
|PreScore|对 Score 扩展点的数据做一些预处理操作，然后将其存入缓存中待 Score 扩展点执行的时候使用 |
|Score|针对当前 Pod，对所有的节点进行打分|
|Normalize Score|针对 Score 扩展点的打分结果进行修正|
|Reserve|预留 PVC 等资源|
|Permit|在执行绑定操作之前对当前 Pod 的调度进行最后的决策，包括：批准、拒绝或者延时调度|
|WaitOnPermit|与 Permit 扩展点配合使用实现延时调度功能|
|PreBind|预绑定操作，可以为 Bind 扩展点做一些准备工作，也可以对附属资源进行绑定，例如 PVC 和 PV |
|Bind|将当前 Pod 与选出的节点进行绑定|
|PostBind|绑定的后置动作，可以执行通知操作，或者做一些资源清理工作|

在整个调度过程中，除了考虑 Pod 是否可以调度到特定的节点上，还需要考虑相关资源，例如 Pod 所使用的的 PVC 是否与目标节点匹配等。

接下来，会在 [Kubernetes Scheduler 中的组件和启动过程](../README.md#Kubernetes-Scheduler-中的组件和启动过程) 中对整个 Kubernetes Scheduler 所使用到的主要组件进行分析，以及它们的初始化过程。接着，会在 [调度流程分析](scheduling/overview.md) 中对整个调度流程以及其中的每一个扩展点进行代码层面的分析。最后，会在 [调度算法详解](scheduler-plugins/overview.md) 中分析常见的调度插件的实现逻辑。相对于前两部分的理论分析，这个部分的对象是具体的插件，包括详细的过滤和打分等过程。
