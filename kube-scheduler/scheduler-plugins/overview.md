# 调度插件详解 #

|名称|实现的扩展点|默认启用|说明|
|---|---|---|---|
|[PrioritySort](priority-sort.md)|QueueSort|True|提供默认的基于优先级的排序|
|[DefaultBinder](default-binder.md)|Bind|True|提供默认的绑定机制|
|DefaultPreemption|PostFilter|True|提供默认的抢占机制|
|||||
|[NodeName](node-name.md)|Filter|True|检查Pod指定的节点名称与当前节点是否匹配|
|[NodePorts](node-ports.md)|PreFilter、Filter|True|检查Pod请求的端口在节点上是否可用|
|[SelectorSpread](selector-spread.md)|PreScore、Score、NormalizeScore|True|对于属于Services、ReplicaSets和StatefulSets的Pod，偏好跨多个节点部署|
|ImageLocality|Score|True|选择已经存在Pod运行所需容器镜像的节点|
|TaintToleration|Filter、Prescore、Score|True|实现了污点和容忍|
|NodePreferAvoidPods|Score|True|基于节点的注解scheduler.alpha.kubernetes.io/preferAvoidPods打分|
|NodeAffinity|Filter、Score|True|实现了节点选择器和节点亲和性|
|PodTopologySpread|PreFilter、Filter、PreScore、Score|True|实现了Pod拓扑分布|
|[NodeUnschedulable](node-unschedulable.md)|Filter|True|过滤.spec.unschedulable值为true的节点|
|NodeResourcesFit|PreFilter、Filter|True|检查节点是否拥有Pod请求的所有资源|
|NodeResourcesBalancedAllocation|Score|True|调度Pod时，选择资源使用更为均衡的节点|
|NodeResourcesLeastAllocated|Score|True|选择资源分配较少的节点|
|InterPodAffinity|PreFilter、Filter、PreScore、Score|True|实现Pod间亲和性与反亲和性|
|NodeResourcesMostAllocated|Score|False|选择已分配资源多的节点|
|RequestedToCapacityRatio|Score|False|根据已分配资源的某函数设置选择节点|
|NodeResourceLimits|PreScore、Score|False|选择满足Pod资源限制的节点|
|[NodeLabel](node-label.md)|Filter、Score|False|根据配置的标签过滤节点和/或给节点打分|
|ServiceAffinity|PreFilter、Filter、Score|False|检查属于某个Service的Pod与配置的标签所定义的节点集是否适配。这个插件还支持将属于某个Service的Pod分散到各个节点|
|||||
|VolumeBinding|PreFilter、Filter、Reserve、PreBind|True|检查节点是否有请求的卷，或是否可以绑定请求的卷|
|VolumeRestrictions|Filter|True|检查挂载到节点上的卷是否满足卷提供程序的限制|
|VolumeZone|Filter|True|检查请求的卷是否在任何区域都满足|
|NodeVolumeLimits|Filter|True|检查该节点是否满足CSI卷限制|
|EBSLimits|Filter|True|检查节点是否满足AWS EBS卷限制|
|GCEPDLimits|Filter|True|检查该节点是否满足GCP-PD卷限制|
|AzureDiskLimits|Filter|True|检查该节点是否满足Azure卷限制|
|CinderVolume|Filter|False|检查该节点是否满足OpenStackCinder卷限制|
