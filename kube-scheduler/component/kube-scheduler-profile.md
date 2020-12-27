# KubeSchedulerProfile #

在 Kubernetes Scheduler 的代码中有两个和 Profile 字样相关的对象：KubeSchedulerProfile 和 Profile。他们是两个完全不同的对象，但是又有一些关联关系。

- ***KubeSchedulerProfile*** 是提供给用户进行配置的界面。
- ***Profile*** 是由 KubeSchedulerProfile 创建而来，用于执行具体的调度操作。每个调度器对应一个 Profile。

本节分析 KubeSchedulerProfile 对象的初始化过程，下一节分析 Profile 对象的创建和使用。

KubeSchedulerProfile 是提供给用户进行配置的界面。

```
type KubeSchedulerProfile struct {
	SchedulerName string
	Plugins *Plugins
	PluginConfig []PluginConfig
}
```

内部包含了三个成员，分别表示当前调度器的名称、当前调度器包含的插件列表，当前调度器包含的插件对应的配置信息。

在 `scheduler.New()` 内部初始化的时候会创建一个默认的值。

``` go
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	podInformer coreinformers.PodInformer,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {

	stopEverything := stopCh
	if stopEverything == nil {
		stopEverything = wait.NeverStop
	}

	options := defaultSchedulerOptions
	for _, opt := range opts {
		opt(&options)
	}
    ...
```

``` go
var defaultSchedulerOptions = schedulerOptions{
	profiles: []schedulerapi.KubeSchedulerProfile{
		// Profiles' default plugins are set from the algorithm provider.
		{SchedulerName: v1.DefaultSchedulerName},
	},
    ...
}

const DefaultSchedulerName = "default-scheduler"
```

可以看出这里仅仅是创建了一个名为 "default-scheduler" 的默认调度器对应的 KubeSchedulerProfile，对应的插件和插件配置信息都为空。

由于当前函数的可变参数 `opts` 在调用时包含了 `scheduler.WithProfiles(cc.ComponentConfig.Profiles...)`，因此，如果用户指定的配置文件中有 Profiles，会用它覆盖默认的 KubeSchedulerProfile，详见 [参数初始化过程分析](option.md)。

``` go
	configurator := &Configurator{
        ...
		profiles:                 append([]schedulerapi.KubeSchedulerProfile(nil), options.profiles...),
        ...
	}

	source := options.schedulerAlgorithmSource
	switch {
	case source.Provider != nil:
		// Create the config from a named algorithm provider.
		sc, err := configurator.createFromProvider(*source.Provider)
        ...
		sched = sc
```

接着将这个 KubeSchedulerProfile 存入 Configurator 对象中的 `profiles []schedulerapi.KubeSchedulerProfile` 字段中。

紧接着在 `configurator.createFromProvider()` 中会对这个 `profiles []schedulerapi.KubeSchedulerProfile` 对象进一步填充。

``` go
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	klog.V(2).Infof("Creating scheduler from algorithm provider '%v'", providerName)
	r := algorithmprovider.NewRegistry()
	defaultPlugins, exist := r[providerName]
    ...

	for i := range c.profiles {
		prof := &c.profiles[i]
		plugins := &schedulerapi.Plugins{}
		plugins.Append(defaultPlugins)
		plugins.Apply(prof.Plugins)
		prof.Plugins = plugins
	}
	return c.create()
}
```

这里首先获取默认的插件列表，将其存入 `defaultPlugins` 变量中，可参见 [Algorithm Provider](./algorithm-provider.md)。然后使用 `plugins.Append(defaultPlugins)` 将默认插件加入到 `profiles []schedulerapi.KubeSchedulerProfile` 中，最后再把前文提到的用户自定义的 Profiles 中指定的插件合并来。由于用户指定的插件的优先级最高，因此最后执行，这样便可以覆盖默认的插件。

至此，已经完成了 Configurator 中 `profiles []schedulerapi.KubeSchedulerProfile` 字段的初始化。接下来分析从 Profile 到 `profiles []schedulerapi.KubeSchedulerProfile` 的转换过程。
