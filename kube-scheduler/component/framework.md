# Kubernetes 调度框架 #

在 [调度插件对象和扩展点对象](plugin-and-extensionpoint.md) 一节中，我们对调度插件本身和调度扩展点对象进行了一些说明，本节我们将详细分析它们是如何集成到调度框架中的。

在 Kubernetes 调度框架(Scheduler Framework)中有多个扩展点，而每个扩展点可以启用多个插件(Sort 扩展点除外，同时只能启用一个 Sort 扩展点的插件)。下面是调度框架的定义：

``` go
type framework struct {
	queueSortPlugins      []QueueSortPlugin
	preFilterPlugins      []PreFilterPlugin
	filterPlugins         []FilterPlugin
	preScorePlugins       []PreScorePlugin
	scorePlugins          []ScorePlugin
	reservePlugins        []ReservePlugin
	preBindPlugins        []PreBindPlugin
	bindPlugins           []BindPlugin
	postBindPlugins       []PostBindPlugin
	unreservePlugins      []UnreservePlugin
	permitPlugins         []PermitPlugin

	registry              Registry
	pluginNameToWeightMap map[string]int
    ...
}
```

其中前面的一部分是针对不同扩展点的启用的插件列表，每个扩展点对应一个插件数组。

现在来考虑如何将这么多插件进行实例化并存储到 Scheduler Framework 结构体中。

在配置的时候我们定义了默认启用的插件名称以及每个插件的配置参数，详见 [Algorithm Provider](algorithm-provider.md)。在 [KubeSchedulerProfile](kube-scheduler-profile.md) 一节中，又分析了在初始化的时候如何将用户自定义的插件与默认的插件列表合并。接下来分析如何将这些配置的插件名称的列表和参数实例化成插件实例对象并存储到 Scheduler Framework 结构体中。

在 [Profile](./profile.md) 中我们分析了每一种调度器包含一个 `Scheduler Framework` 对象，下面来看每一个 `Scheduler Framework` 的初始化过程。

``` go
func NewFramework(r Registry, plugins *config.Plugins, args []config.PluginConfig, opts ...Option) (Framework, error) {
```

先来分析一下调用的参数，

- `r Registry` 参数是在 [framework.Registry](./framework-registry.md) 中用 `frameworkplugins.NewInTreeRegistry()` 返回的对象，即所有 in-tree 类型的调度插件名称和创建对应插件的回调函数。
- `plugins *config.Plugins` 参数是在 [KubeSchedulerProfile](./kube-scheduler-profile.md) 中分析的合并了默认启用的插件列表合并了用户自定义的插件列表后返回的值，这个插件列表只是插件名称列表，而不是对应的插件对象。
- `args []config.PluginConfig` 参数是用户定义的插件参数信息。
- `opts ...Option`，由回调函数组成的可变参数，用于覆盖默认的值，使用这些回调函数来把调用 `NewFramework()` 时指定的参数覆盖默认的 `options` 中的字段：

  ``` go
  func (c *Configurator) buildFramework(p schedulerapi.KubeSchedulerProfile) (framework.Framework, error) {
      return framework.NewFramework(
          c.registry,
          p.Plugins,
          p.PluginConfig,
          framework.WithClientSet(c.client),
          framework.WithInformerFactory(c.informerFactory),
          framework.WithSnapshotSharedLister(c.nodeInfoSnapshot),
          framework.WithRunAllFilters(c.alwaysCheckAllPredicates),
          framework.WithVolumeBinder(c.volumeBinder),
      )
  }
  ```

接着开始执行 `Scheduler Framework` 的初始化过程。

``` go
	options := defaultFrameworkOptions
	for _, opt := range opts {
		opt(&options)
	}
```

先使用默认的 opts 可变参数来覆盖默认的 `options` 中的字段。例如用 `framework.WithClientSet(c.client)` 调用时，会用 `c.client` 的值覆盖 `options` 中的 client。

初始化完成 `options` 后，创建 `Scheduler Framework` 对象。

```
	f := &framework{
		registry:              r,
		snapshotSharedLister:  options.snapshotSharedLister,
		pluginNameToWeightMap: make(map[string]int),
		waitingPods:           newWaitingPodsMap(),
		clientSet:             options.clientSet,
		informerFactory:       options.informerFactory,
		volumeBinder:          options.volumeBinder,
		metricsRecorder:       options.metricsRecorder,
		runAllFilters:         options.runAllFilters,
	}
	if plugins == nil {
		return f, nil
	}
```

其中大部分字段使用 `options` 中的值。另外，其中的 `registry` 成员是在 [framework.Registry](./framework-registry.md) 中用 `frameworkplugins.NewInTreeRegistry()` 返回的对象，即所有 in-tree 类型的调度插件名称和创建对应插件的回调函数。

``` go
	// get needed plugins from config
	pg := f.pluginsNeeded(plugins)
```

这个函数调用是将 `plugins` 按照 `Scheduler Framework` 中不同的扩展点进行整理，形成一个 map，key 为插件名称，value 为 `config.Plugin` 类型，这是一种封装了插件名称和插件权重值的配置信息。

``` go
	pluginConfig := make(map[string]*runtime.Unknown, 0)
	for i := range args {
		name := args[i].Name
		if _, ok := pluginConfig[name]; ok {
			return nil, fmt.Errorf("repeated config for plugin %s", name)
		}
		pluginConfig[name] = &args[i].Args
	}
```

上面这段代码是将插件的参数进行整理，形成一个以插件名称为 key 的 map 对象。

``` go
	pluginsMap := make(map[string]Plugin)
	var totalPriority int64
	for name, factory := range r {
		// initialize only needed plugins.
		if _, ok := pg[name]; !ok {
			continue
		}

		p, err := factory(pluginConfig[name], f)
		if err != nil {
			return nil, fmt.Errorf("error initializing plugin %q: %v", name, err)
		}
		pluginsMap[name] = p

		// a weight of zero is not permitted, plugins can be disabled explicitly
		// when configured.
		f.pluginNameToWeightMap[name] = int(pg[name].Weight)
		if f.pluginNameToWeightMap[name] == 0 {
			f.pluginNameToWeightMap[name] = 1
		}
		// Checks totalPriority against MaxTotalScore to avoid overflow
		if int64(f.pluginNameToWeightMap[name])*MaxNodeScore > MaxTotalScore-totalPriority {
			return nil, fmt.Errorf("total score of Score plugins could overflow")
		}
		totalPriority += int64(f.pluginNameToWeightMap[name]) * MaxNodeScore
	}
```

接着创建具体的插件对象，这里的 factory 是每个插件的创建插件对象的回调函数。执行时会将此插件的配置信息和当前插件所述的 Framework 作为参数。创建完成之后将结果存入一个名为 pluginsMap 的 map 中，key 为插件名称，value 是实例化后的插件对象。检测插件的权重值，如果没有指定，则默认为 1；最后进行权重的有效性验证，所有插件的权重*100加起来不能超过 `math.MaxInt64`。

``` go
	for _, e := range f.getExtensionPoints(plugins) {
		if err := updatePluginList(e.slicePtr, e.plugins, pluginsMap); err != nil {
			return nil, err
		}
	}
```

这段代码是将刚才创建的实例化后的插件对象的 map 中的内容存入 Framework 对应的成员中。下面对这段代码进行展开分析。

``` go
func (f *framework) getExtensionPoints(plugins *config.Plugins) []extensionPoint {
	return []extensionPoint{
		{plugins.PreFilter, &f.preFilterPlugins},
		{plugins.Filter, &f.filterPlugins},
		{plugins.Reserve, &f.reservePlugins},
		{plugins.PreScore, &f.preScorePlugins},
		{plugins.Score, &f.scorePlugins},
		{plugins.PreBind, &f.preBindPlugins},
		{plugins.Bind, &f.bindPlugins},
		{plugins.PostBind, &f.postBindPlugins},
		{plugins.Unreserve, &f.unreservePlugins},
		{plugins.Permit, &f.permitPlugins},
		{plugins.QueueSort, &f.queueSortPlugins},
	}
}
```

`getExtensionPoints()` 返回 Framework 中所有的扩展点对象，每个扩展点对象的 plugins 成员是当前扩展点所启用的插件名称的列表，每个扩展点对象的 slicePtr 成员是 Framework 中当前扩展点的插件实体对象数组的指针，后续会通过修改这个指针所指向的内容来给每个扩展点对象赋值。

``` go
func updatePluginList(pluginList interface{}, pluginSet *config.PluginSet, pluginsMap map[string]Plugin) error {
	if pluginSet == nil {
		return nil
	}

	plugins := reflect.ValueOf(pluginList).Elem()
	pluginType := plugins.Type().Elem()
	set := sets.NewString()
	for _, ep := range pluginSet.Enabled {
		pg, ok := pluginsMap[ep.Name]
		if !ok {
			return fmt.Errorf("%s %q does not exist", pluginType.Name(), ep.Name)
		}

		if !reflect.TypeOf(pg).Implements(pluginType) {
			return fmt.Errorf("plugin %q does not extend %s plugin", ep.Name, pluginType.Name())
		}

		if set.Has(ep.Name) {
			return fmt.Errorf("plugin %q already registered as %q", ep.Name, pluginType.Name())
		}

		set.Insert(ep.Name)

		newPlugins := reflect.Append(plugins, reflect.ValueOf(pg))
		plugins.Set(newPlugins)
	}
	return nil
}
```

`updatePluginList()` 对每个扩展点分别进行赋值。它的第一个参数是刚才提到的 Framework 中当前扩展点的插件实体对象数组的指针，第二个参数是当前扩展点所启用的插件名称的列表，第三个参数是刚才已经创建好的名为 pluginsMap 的 map 中，key 为插件名称，value 是实例化后的插件对象。`updatePluginList()` 中具体执行时，会遍历每个已启用的插件名称列表，然后从 pluginsMap 中找到对应的实例化后的插件对象，然后校验其是否实现了当前扩展点的 interface。接着进行重复性检测，最后将此实例化后的插件对象追加到 Framework 中扩展点的插件实体对象数组中。

以上就是创建 Framework 的核心实现。

``` go
	// Verifying the score weights again since Plugin.Name() could return a different
	// value from the one used in the configuration.
	for _, scorePlugin := range f.scorePlugins {
		if f.pluginNameToWeightMap[scorePlugin.Name()] == 0 {
			return nil, fmt.Errorf("score plugin %q is not configured with weight", scorePlugin.Name())
		}
	}

	if len(f.queueSortPlugins) == 0 {
		return nil, fmt.Errorf("no queue sort plugin is enabled")
	}
	if len(f.queueSortPlugins) > 1 {
		return nil, fmt.Errorf("only one queue sort plugin can be enabled")
	}
	if len(f.bindPlugins) == 0 {
		return nil, fmt.Errorf("at least one bind plugin is needed")
	}

	return f, nil
}
```

这是最后的一些检测，包括 Sort 扩展点必须启用且只能启用一个插件；必须至少启用一个 Bind 扩展点插件等等。

Framework 初始化完成后，会作为 value 存入以调度器名称为 key 的 Profile 中，相关内容可参考 [Profile](./profile.md)。

在后续章节中，会对调度的过程进行详细分析。
