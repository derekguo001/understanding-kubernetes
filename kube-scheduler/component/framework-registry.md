# framework.Registry #

本节分析另外一个和插件管理相关的组件 `framework.Registry`。

``` go
type Registry map[string]PluginFactory

type PluginFactory = func(configuration *runtime.Unknown, f FrameworkHandle) (Plugin, error)
```

framework.Registry 本身是一个 map 对象，key 为插件名称，value 是这个插件对应的回调函数。在 `scheduler.New()` 内部进行创建 scheduler 对象的时候，会获取这个 framework.Registry。

``` go
// New returns a Scheduler
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	podInformer coreinformers.PodInformer,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {

    ...
	options := defaultSchedulerOptions
	for _, opt := range opts {
		opt(&options)
	}

    ...
	registry := frameworkplugins.NewInTreeRegistry()
```

即代码里的 `frameworkplugins.NewInTreeRegistry()`。

``` go
func NewInTreeRegistry() framework.Registry {
	return framework.Registry{
		defaultpodtopologyspread.Name:              defaultpodtopologyspread.New,
		imagelocality.Name:                         imagelocality.New,
		tainttoleration.Name:                       tainttoleration.New,
		nodename.Name:                              nodename.New,
		nodeports.Name:                             nodeports.New,
		nodepreferavoidpods.Name:                   nodepreferavoidpods.New,
		nodeaffinity.Name:                          nodeaffinity.New,
		podtopologyspread.Name:                     podtopologyspread.New,
		nodeunschedulable.Name:                     nodeunschedulable.New,
		noderesources.FitName:                      noderesources.NewFit,
		noderesources.BalancedAllocationName:       noderesources.NewBalancedAllocation,
		noderesources.MostAllocatedName:            noderesources.NewMostAllocated,
		noderesources.LeastAllocatedName:           noderesources.NewLeastAllocated,
		noderesources.RequestedToCapacityRatioName: noderesources.NewRequestedToCapacityRatio,
		noderesources.ResourceLimitsName:           noderesources.NewResourceLimits,
		volumebinding.Name:                         volumebinding.New,
		volumerestrictions.Name:                    volumerestrictions.New,
		volumezone.Name:                            volumezone.New,
		nodevolumelimits.CSIName:                   nodevolumelimits.NewCSI,
		nodevolumelimits.EBSName:                   nodevolumelimits.NewEBS,
		nodevolumelimits.GCEPDName:                 nodevolumelimits.NewGCEPD,
		nodevolumelimits.AzureDiskName:             nodevolumelimits.NewAzureDisk,
		nodevolumelimits.CinderName:                nodevolumelimits.NewCinder,
		interpodaffinity.Name:                      interpodaffinity.New,
		nodelabel.Name:                             nodelabel.New,
		serviceaffinity.Name:                       serviceaffinity.New,
		queuesort.Name:                             queuesort.New,
		defaultbinder.Name:                         defaultbinder.New,
	}
}
```

可以看出每个插件的 value 实际上是当前插件的构造函数。返回的这个总的 `registry` 对象最终会存储到 `Scheduler Framework` 中的 `registry Registry` 字段中。目前，实际上这个字段只在初始化 `Scheduler Framework` 的过程中使用到。
