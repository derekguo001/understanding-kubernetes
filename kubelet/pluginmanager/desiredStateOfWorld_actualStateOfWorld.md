# desiredStateOfWorld 与 actualStateOfWorld #

本文来分析 desiredStateOfWorld 与 actualStateOfWorld 这两个缓存对象。

## desiredStateOfWorld ##

``` go
type desiredStateOfWorld struct {
	socketFileToInfo map[string]PluginInfo
	sync.RWMutex
}
```

核心对象是一个 map，key为 socket 文件路径，value 则是对应的 PluginInfo 对象。

实现了 DesiredStateOfWorld interface。

``` go
type DesiredStateOfWorld interface {
	AddOrUpdatePlugin(socketPath string) error
	RemovePlugin(socketPath string)
	GetPluginsToRegister() []PluginInfo
	PluginExists(socketPath string) bool
}
```

其中各个函数的实现也非常简单。例如有一个插件需要添加到 desiredStateOfWorld 的时候，就会执行其中的 `AddOrUpdatePlugin()` 函数，这个函数的实际操作就是将当前插件对应的 socket 文件路径和对应的 PluginInfo 对象添加到刚才提到的内部的 map 中，并更新时间戳。删除插件时的操作相反。

代码如下：

``` go
func (dsw *desiredStateOfWorld) AddOrUpdatePlugin(socketPath string) error {
	dsw.Lock()
	defer dsw.Unlock()

	if socketPath == "" {
		return fmt.Errorf("socket path is empty")
	}
	if _, ok := dsw.socketFileToInfo[socketPath]; ok {
		klog.V(2).Infof("Plugin (Path %s) exists in actual state cache, timestamp will be updated", socketPath)
	}

    ...
	dsw.socketFileToInfo[socketPath] = PluginInfo{
		SocketPath: socketPath,
		Timestamp:  time.Now(),
	}
	return nil
}

func (dsw *desiredStateOfWorld) RemovePlugin(socketPath string) {
	dsw.Lock()
	defer dsw.Unlock()

	delete(dsw.socketFileToInfo, socketPath)
}
```

## actualStateOfWorld ##

actualStateOfWorld 的实现与 desiredStateOfWorld 几乎一模一样。

``` go
type actualStateOfWorld struct {
	socketFileToInfo map[string]PluginInfo
	sync.RWMutex
}
```

``` go
type ActualStateOfWorld interface {
	GetRegisteredPlugins() []PluginInfo
	AddPlugin(pluginInfo PluginInfo) error
	RemovePlugin(socketPath string)
	PluginExistsWithCorrectTimestamp(pluginInfo PluginInfo) bool
}
```
