# 调度插件对象和扩展点对象 #

## 调度插件 ##

在 Scheduler 框架中，针对不同扩展点的插件，都定义了相应的 interface。例如：

``` go
type PreScorePlugin interface {
	Plugin
	PreScore(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) *Status
}
```

这是针对 PreScore 扩展点定义的 interface，所有实现 PreScore 扩展点功能的插件都需要实现这个 interface。即需要实现 `PreScore()` 函数。

其它所有扩展点也都定义了相应的 interface，在这里不再一一列举。

另外从上面代码可以看出这些所有的扩展点的 interface 都实现了上面的 Plugin interface。

``` go
type Plugin interface {
	Name() string
}
```

这是所有的调度插件都需要实现的接口，即返回当前插件的名称。

这里的 `调度插件` 是指已经实例化后的插件本身，是内存中运行的实体。而非调度插件名称组成的列表对象，后者只是一些字符串集合。

## 扩展点 ##

在 Kubernetes 调度框架中有很多扩展点，分别代表了调度过程中的不同阶段，每个扩展点分别实现不同的功能。

在代码层面，扩展点有着具体的结构体。

``` go
type extensionPoint struct {
	// the set of plugins to be configured at this extension point.
	plugins *config.PluginSet
	// a pointer to the slice storing plugins implementations that will run at this
	// extension point.
	slicePtr interface{}
}
```

其中的 `plugins *config.PluginSet` 表示在当前扩展点用户配置的启用或者禁用的调度插件名称的列表。下面是 `config.PluginSet` 的结构：

``` go
type PluginSet struct {
	Enabled []Plugin
	Disabled []Plugin
}

type Plugin struct {
	Name string
	// Weight defines the weight of plugin, only used for Score plugins.
	Weight int32
}
```

可以看出这里仅仅是调度器插件名称的列表(如果当前插件实现了 Score 扩展点，其中的 `Weight` 还用来表示当前插件的权重)。

扩展点对象的 `slicePtr interface{}` 成员代表当前扩展点的插件对应的插件对象本身，这里的插件对象是指 [调度插件](#调度插件) 中所描述的插件实体，是实例化后的插件本身，而不仅仅是一个名字。在执行调度操作的时候，调度框架会针对每一个扩展点，分别调用这里的 `slicePtr interface{}` 字段中指定的所有的调度插件，执行具体的调度操作。

现在，我们知道扩展点对象表示在每个扩展点的插件命令以及对应的可执行插件对象本身。在下一节，我们将会看这些对象如果集成到调度框架中。
