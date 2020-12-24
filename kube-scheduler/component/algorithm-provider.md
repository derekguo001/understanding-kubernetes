# Algorithm Provider #

kubernetes Scheduler 提供了一个名为 `--algorithm-provider string` 的参数，它的可选值包括 `DefaultProvider` 和 `ClusterAutoscalerProvider`，可以通过它来指定所使用的的调度插件列表。

注意这个参数在目前版本中已经被标记为过期，在未来的版本中会被废弃掉，即便如此，我们仍然会在这里分析一下它的内部逻辑，因为它对于整个 kubernetes Scheduler 的配置和使用至关重要。代码位于 `pkg/scheduler/algorithmprovider/registry.go`。

在 kubernetes Scheduler 初始化过程中，会使用下面的代码片段。

``` go
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	klog.V(2).Infof("Creating scheduler from algorithm provider '%v'", providerName)
	r := algorithmprovider.NewRegistry()
	defaultPlugins, exist := r[providerName]
    ...
}
```

首先，会创建一个 registry 对象，然后使用配置的 `providerName` 来获取插件列表，接着会用这个插件列表对象进行后续的初始化操作。

这段代码其实就是 Algorithm Provider 的全部使用。下面来分析其具体实现。

``` go
type Registry map[string]*schedulerapi.Plugins
```

首先，Registry 对象是一个 map，key 为 Algorithm Provider 名称，value 为其对应的调度插件列表。

``` go
func NewRegistry() Registry {
	defaultConfig := getDefaultConfig()
	applyFeatureGates(defaultConfig)

	caConfig := getClusterAutoscalerConfig()
	applyFeatureGates(caConfig)

	return Registry{
		schedulerapi.SchedulerDefaultProviderName: defaultConfig,
		ClusterAutoscalerProvider:                 caConfig,
	}
}
```

这里返回的 Registry 的 key 就是命令行选项中的 `DefaultProvider` 和 `ClusterAutoscalerProvider`，接下来看它们的 value。

`DefaultProvider` 对应的 value 是使用 `getDefaultConfig()` 返回的，内容就是按照不同的插入点分类的不同的插件名称，代码如下：

``` go
func getDefaultConfig() *schedulerapi.Plugins {
	return &schedulerapi.Plugins{
		QueueSort: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: queuesort.Name},
			},
		},
		PreFilter: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: noderesources.FitName},
				{Name: nodeports.Name},
				{Name: interpodaffinity.Name},
			},
		},
		Filter: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: nodeunschedulable.Name},
				{Name: noderesources.FitName},
				{Name: nodename.Name},
				{Name: nodeports.Name},
				{Name: nodeaffinity.Name},
				{Name: volumerestrictions.Name},
				{Name: tainttoleration.Name},
				{Name: nodevolumelimits.EBSName},
				{Name: nodevolumelimits.GCEPDName},
				{Name: nodevolumelimits.CSIName},
				{Name: nodevolumelimits.AzureDiskName},
				{Name: volumebinding.Name},
				{Name: volumezone.Name},
				{Name: interpodaffinity.Name},
			},
		},
		PreScore: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: interpodaffinity.Name},
				{Name: defaultpodtopologyspread.Name},
				{Name: tainttoleration.Name},
			},
		},
		Score: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: noderesources.BalancedAllocationName, Weight: 1},
				{Name: imagelocality.Name, Weight: 1},
				{Name: interpodaffinity.Name, Weight: 1},
				{Name: noderesources.LeastAllocatedName, Weight: 1},
				{Name: nodeaffinity.Name, Weight: 1},
				{Name: nodepreferavoidpods.Name, Weight: 10000},
				{Name: defaultpodtopologyspread.Name, Weight: 1},
				{Name: tainttoleration.Name, Weight: 1},
			},
		},
		Bind: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: defaultbinder.Name},
			},
		},
	}
}
```

需要注意的是，有可能同一个插件实现了多个插入点，因此会出现多次。

`ClusterAutoscalerProvider` 对应的 value：

``` go
func getClusterAutoscalerConfig() *schedulerapi.Plugins {
	caConfig := getDefaultConfig()
	// Replace least with most requested.
	for i := range caConfig.Score.Enabled {
		if caConfig.Score.Enabled[i].Name == noderesources.LeastAllocatedName {
			caConfig.Score.Enabled[i].Name = noderesources.MostAllocatedName
		}
	}
	return caConfig
}
```

可以看出它的插件列表和 `DefaultProvider` 的插件列表几乎是一样的，只是将 Score 插入点的 `NodeResourcesLeastAllocated` 插件替换为 `NodeResourcesMostAllocated`。

在获取了插件列表后，还会使用 `applyFeatureGates()` 做一些额外的修改。

``` go
func applyFeatureGates(config *schedulerapi.Plugins) {
	if utilfeature.DefaultFeatureGate.Enabled(features.EvenPodsSpread) {
		klog.Infof("Registering EvenPodsSpread predicate and priority function")
		f := schedulerapi.Plugin{Name: podtopologyspread.Name}
		config.PreFilter.Enabled = append(config.PreFilter.Enabled, f)
		config.Filter.Enabled = append(config.Filter.Enabled, f)
		config.PreScore.Enabled = append(config.PreScore.Enabled, f)
		s := schedulerapi.Plugin{Name: podtopologyspread.Name, Weight: 1}
		config.Score.Enabled = append(config.Score.Enabled, s)
	}

	if utilfeature.DefaultFeatureGate.Enabled(features.ResourceLimitsPriorityFunction) {
		klog.Infof("Registering resourcelimits priority function")
		s := schedulerapi.Plugin{Name: noderesources.ResourceLimitsName}
		config.PreScore.Enabled = append(config.PreScore.Enabled, s)
		s = schedulerapi.Plugin{Name: noderesources.ResourceLimitsName, Weight: 1}
		config.Score.Enabled = append(config.Score.Enabled, s)
	}
}
```

这里的内容是根据是否开启了 `EvenPodsSpread` 和 `ResourceLimitsPriorityFunction` 特性来决定是否加入额外的插件。

调用 `applyFeatureGates()` 修改完插件列表后，`NewRegistry()` 返回 Registry 对象。

现在回到本文开头的代码中。

``` go
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	klog.V(2).Infof("Creating scheduler from algorithm provider '%v'", providerName)
	r := algorithmprovider.NewRegistry()
	defaultPlugins, exist := r[providerName]
    ...
}
```

这里的参数 `providerName` 在初始化的时候已经被定义为 `DefaultProvider`，相关的初始化结果存储在 `defaultSchedulerOptions` 中。

``` go
var defaultSchedulerOptions = schedulerOptions{
    ...
	schedulerAlgorithmSource: schedulerapi.SchedulerAlgorithmSource{
		Provider: defaultAlgorithmSourceProviderName(),
	},
    ...
}
```

其中的 `defaultAlgorithmSourceProviderName()`。

``` go
func defaultAlgorithmSourceProviderName() *string {
	provider := schedulerapi.SchedulerDefaultProviderName
	return &provider
}
```

``` go
const (
    ...

	// SchedulerDefaultProviderName defines the default provider names
	SchedulerDefaultProviderName = "DefaultProvider"
)
```

至此，已经全部分析完了关于 kubernetes Scheduler 中 Algorithm Provider 的内容。

kubernetes Scheduler 初始化时定义了 Provider 名称为 `DefaultProvider`(如果没有在命令行使用参数指定其它值)，接着使用 `algorithmprovider.NewRegistry()` 返回 Registry 对象，然后使用 `DefaultProvider` 返回其中对应的插件列表。
