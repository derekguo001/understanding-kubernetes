# reconciler #

reconciler 对象则会比较 desiredStateOfWorld 和 actualStateOfWorld 这两个缓存对象之间的差异，然后执行相应的动作，使所有插件的期望状态和实际状态保持一致。定义如下：

``` go
type reconciler struct {
	operationExecutor   operationexecutor.OperationExecutor
	loopSleepDuration   time.Duration
	desiredStateOfWorld cache.DesiredStateOfWorld
	actualStateOfWorld  cache.ActualStateOfWorld
	handlers            map[string]cache.PluginHandler
	sync.RWMutex
}
```

它实现了 Reconciler interface：

``` go
type Reconciler interface {
	Run(stopCh <-chan struct{})
	AddHandler(pluginType string, pluginHandler cache.PluginHandler)
}
```

reconciler 结构中的 handlers 即在 [Kubelet 中的插件管理](./overview.md) 中提到的每一类插件对应的 handler，它实现了一些当前插件的回调函数。

reconciler 中还有一个 operationExecutor 对象，这个是实际操作的执行者。

下面来分析 reconciler 的执行过程。核心函数是 `reconcile()`。

``` go
func (rc *reconciler) reconcile() {
    ...
	for _, registeredPlugin := range rc.actualStateOfWorld.GetRegisteredPlugins() {
		unregisterPlugin := false
		if !rc.desiredStateOfWorld.PluginExists(registeredPlugin.SocketPath) {
			unregisterPlugin = true
		} else {
			for _, dswPlugin := range rc.desiredStateOfWorld.GetPluginsToRegister() {
				if dswPlugin.SocketPath == registeredPlugin.SocketPath && dswPlugin.Timestamp != registeredPlugin.Timestamp {
                    ...
					unregisterPlugin = true
					break
				}
			}
		}

		if unregisterPlugin {
			klog.V(5).Infof(registeredPlugin.GenerateMsgDetailed("Starting operationExecutor.UnregisterPlugin", ""))
			err := rc.operationExecutor.UnregisterPlugin(registeredPlugin, rc.actualStateOfWorld)
            ...
			}
            ...
		}
	}
```

这是整个函数的前半段，它会遍历 actualStateOfWorld，如果一个插件在 actualStateOfWorld 中存在但是在 desiredStateOfWorld 中不存在，或者两者的时间戳不一致，则认为此插件需要删掉，会调用 `operationExecutor.UnregisterPlugin()` 执行插件的删除操作。

`reconcile()` 函数的下半部分代码如下：

``` go
	for _, pluginToRegister := range rc.desiredStateOfWorld.GetPluginsToRegister() {
		if !rc.actualStateOfWorld.PluginExistsWithCorrectTimestamp(pluginToRegister) {
			err := rc.operationExecutor.RegisterPlugin(pluginToRegister.SocketPath, pluginToRegister.Timestamp, rc.getHandlers(), rc.actualStateOfWorld)
            ...
		}
	}
}
```

它会遍历 desiredStateOfWorld，如果一个插件在  actualStateOfWorld 中存在但是在 actualStateOfWorld 中不存在，或者两者的时间戳不一致，则认为此插件是新增加的，需要执行注册步骤，会调用 `operationExecutor.RegisterPlugin()` 执行插件的注册操作。

可以看出插件的注册和删除都是通过调用 operationExecutor 来实现的，接下来分析这个对象。

## operationExecutor ##

``` go
type operationExecutor struct {
	pendingOperations goroutinemap.GoRoutineMap
	operationGenerator OperationGenerator
}
```

其中包含了一个 operationGenerator 对象，它会生成实际的回调函数并由 operationExecutor 完成执行。

operationExecutor 实现了 ActualStateOfWorldUpdater interface。

``` go
type ActualStateOfWorldUpdater interface {
	AddPlugin(pluginInfo cache.PluginInfo) error
	RemovePlugin(socketPath string)
}
```

这个 interface 只是简单地抽象了插件的注册和删除的两个函数。下面是这两个函数的实现：

``` go
func (oe *operationExecutor) RegisterPlugin(
	socketPath string,
	timestamp time.Time,
	pluginHandlers map[string]cache.PluginHandler,
	actualStateOfWorld ActualStateOfWorldUpdater) error {
	generatedOperation :=
		oe.operationGenerator.GenerateRegisterPluginFunc(socketPath, timestamp, pluginHandlers, actualStateOfWorld)

	return oe.pendingOperations.Run(
		socketPath, generatedOperation)
}

func (oe *operationExecutor) UnregisterPlugin(
	pluginInfo cache.PluginInfo,
	actualStateOfWorld ActualStateOfWorldUpdater) error {
	generatedOperation :=
		oe.operationGenerator.GenerateUnregisterPluginFunc(pluginInfo, actualStateOfWorld)

	return oe.pendingOperations.Run(
		pluginInfo.SocketPath, generatedOperation)
}
```

可以看到 operationExecutor 会调用内部的 operationGenerator 对象来生成实际的回调函数，然后执行此函数。接下来进一步分析 operationGenerator 对象。

## operationGenerator ##

operationGenerator 实现了 OperationGenerator interface。

``` go
type OperationGenerator interface {
	GenerateRegisterPluginFunc(
		socketPath string,
		timestamp time.Time,
		pluginHandlers map[string]cache.PluginHandler,
		actualStateOfWorldUpdater ActualStateOfWorldUpdater) func() error

	GenerateUnregisterPluginFunc(
		pluginInfo cache.PluginInfo,
		actualStateOfWorldUpdater ActualStateOfWorldUpdater) func() error
}
```

这个 interface 会返回插件的注册和删除的两个函数。

先来看插件注册的函数时如何生成的。

### 插件的注册 ###

``` go
func (og *operationGenerator) GenerateRegisterPluginFunc(
	socketPath string,
	timestamp time.Time,
	pluginHandlers map[string]cache.PluginHandler,
	actualStateOfWorldUpdater ActualStateOfWorldUpdater) func() error {

	registerPluginFunc := func() error {
		client, conn, err := dial(socketPath, dialTimeoutDuration)
        ...

		infoResp, err := client.GetInfo(ctx, &registerapi.InfoRequest{})
        ...
```

首先，会创建一个到插件的 gRPC Client，然后执行 `client.GetInfo()` 来获取当前插件的一些信息。

``` go
		handler, ok := pluginHandlers[infoResp.Type]
        ...
		if infoResp.Endpoint == "" {
			infoResp.Endpoint = socketPath
		}
		if err := handler.ValidatePlugin(infoResp.Name, infoResp.Endpoint, infoResp.SupportedVersions); err != nil {
            ...
		}
```

接着，会从返回的插件信息中获取插件类型，然后根据插件类型得到此类插件对应的 handler 对象。接着执行 `handler.ValidatePlugin()` 以验证当前插件是有效的。

``` go
		err = actualStateOfWorldUpdater.AddPlugin(cache.PluginInfo{
			SocketPath: socketPath,
			Timestamp:  timestamp,
			Handler:    handler,
			Name:       infoResp.Name,
		})
        ...
		if err := handler.RegisterPlugin(infoResp.Name, infoResp.Endpoint, infoResp.SupportedVersions); err != nil {
			return og.notifyPlugin(client, false, fmt.Sprintf("RegisterPlugin error -- plugin registration failed with err: %v", err))
		}

		if err := og.notifyPlugin(client, true, ""); err != nil {
            ...
		}
		return nil
	}
	return registerPluginFunc
}
```

在验证完插件的有效性之后，会调用 `actualStateOfWorldUpdater.AddPlugin()` 来将此插件添加到 actualStateOfWorld 对象中，表明此插件已经实际完成注册。最后调用  `handler.RegisterPlugin()` 完成实际的注册，并通过 gRPC Client 的 `NotifyRegistrationStatus()` 函数将注册结果通知给插件。

注意在将此插件添加到 actualStateOfWorld 的时候还未执行真正的注册操作，也就意味着 actualStateOfWorld 中的对象并不一定全部都是注册成功的，只是表明 actualStateOfWorld 已经处理过此插件。

接下来分析插件删除的函数的生成过程。

### 插件的删除 ###

``` go
func (og *operationGenerator) GenerateUnregisterPluginFunc(
	pluginInfo cache.PluginInfo,
	actualStateOfWorldUpdater ActualStateOfWorldUpdater) func() error {

	unregisterPluginFunc := func() error {
        ...
		actualStateOfWorldUpdater.RemovePlugin(pluginInfo.SocketPath)

		pluginInfo.Handler.DeRegisterPlugin(pluginInfo.Name)

        ...
		return nil
	}
	return unregisterPluginFunc
}
```

先从 actualStateOfWorld 缓存中删除此插件，然后调用 `Handler.DeRegisterPlugin()` 完成插件的真正的删除操作。
