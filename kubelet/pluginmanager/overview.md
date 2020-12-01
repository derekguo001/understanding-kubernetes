# Kubelet 中的插件管理 #


Kubelet 自身实现了一个插件管理的组件用于扩展 Kubelet 自身的功能，代码位于 `pkg/kubelet/pluginmanager`。

Kubelet 中的插件目前包含两类：

- ***CSI Plugin***。即各种 CSI 插件，确切地说是指各种 CSI 插件中于运行于各个 Node 上与 Kubelet 交互的部分。
- ***DevicePlugin***。即设备插件，包括管理各种硬件设备的插件。

Kubelet 中的插件都需要实现一个共同的 interface，位于 `pkg/kubelet/pluginmanager/cache/types.go`。

```
type PluginHandler interface {
	ValidatePlugin(pluginName string, endpoint string, versions []string) error
	RegisterPlugin(pluginName, endpoint string, versions []string) error
	DeRegisterPlugin(pluginName string)
}
```

这几个函数分别用于实现插件的有效性验证、注册和删除任务。每一类插件实现此 interface 的对象在内部被称为 handler。

Kubelet 中的插件通过 socket 文件与 Kubelet 之间进行通信，同时 Kubelet 也会 watch 对应的 socket 文件，根据 socket 文件存在与否来决定这个插件是否是新注册的或者已经删除，从而执行相应的注册和删除的动作。

Kubelet 插件管理器名为 pluginManager，实现了 PluginManager interface。

``` go
type PluginManager interface {
	Run(sourcesReady config.SourcesReady, stopCh <-chan struct{})
	AddHandler(pluginType string, pluginHandler cache.PluginHandler)
}
```

其中的 `AddHandler()` 用于注册回调函数，参数 pluginType 为插件类型，PluginHandler 为当前类型插件对应的 handler。

pluginManager 结构体定义如下：

``` go
type pluginManager struct {
	desiredStateOfWorldPopulator *pluginwatcher.Watcher
	reconciler reconciler.Reconciler
	actualStateOfWorld cache.ActualStateOfWorld
	desiredStateOfWorld cache.DesiredStateOfWorld
}
```

desiredStateOfWorld 和 actualStateOfWorld 是两个缓存对象，用于存储所有插件期望的和实际的状态。具体而言，如果 Kubelet 发现了一个新的插件，则会将其添加到 desiredStateOfWorld 中，随后会执行注册操作，然后将这个插件的信息再存储到 actualStateOfWorld 中，表明它已经成功注册，可以供 Kubelet 使用；如果之前注册的某个插件在底层已经删除，则会将其从 desiredStateOfWorld 中删除，然后执行删除插件的操作，最后将其从 actualStateOfWorld 中删除。

reconciler 则会比较这两个缓存对象之间的差异，然后执行相应的动作，使所有插件的期望状态和实际状态保持一致。

desiredStateOfWorldPopulator 的作用则是生成 desiredStateOfWorld 的内容。

下面分别针对这几个对象进行详细的分析。
