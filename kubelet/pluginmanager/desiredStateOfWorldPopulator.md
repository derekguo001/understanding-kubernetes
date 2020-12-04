# desiredStateOfWorldPopulator #

desiredStateOfWorldPopulator 的作用则是生成 desiredStateOfWorld 的内容，它在代码实现层面实际上是一个 Watcher 对象。

``` go
func NewPluginManager(
	sockDir string,
	recorder record.EventRecorder) PluginManager {
    ...
	pm := &pluginManager{
		desiredStateOfWorldPopulator: pluginwatcher.NewWatcher(
			sockDir,
			dsw,
		),
        ...
	}
	return pm
}
```

在返回 pluginManager 对象的时候会创建 desiredStateOfWorldPopulator 对象。其中的 sockDir 为 '/var/lib/kubelet/plugins_registry'。

Watcher 对象定义如下：

``` go
type Watcher struct {
	path                string
	fs                  utilfs.Filesystem
	fsWatcher           *fsnotify.Watcher
	desiredStateOfWorld cache.DesiredStateOfWorld
}
```

下面是创建 Watcher 的代码：

``` go
func NewWatcher(sockDir string, desiredStateOfWorld cache.DesiredStateOfWorld) *Watcher {
	return &Watcher{
		path:                sockDir,
		fs:                  &utilfs.DefaultFs{},
		desiredStateOfWorld: desiredStateOfWorld,
	}
}
```

Watcher 启动的代码如下：

``` go
func (w *Watcher) Start(stopCh <-chan struct{}) error {
	klog.V(2).Infof("Plugin Watcher Start at %s", w.path)

    ...

	fsWatcher, err := fsnotify.NewWatcher()
    ...
	w.fsWatcher = fsWatcher

	if err := w.traversePluginDir(w.path); err != nil {
        ...
	}
    ...
```

它会调用 `traversePluginDir()` 来遍历 `/var/lib/kubelet/plugins_registry'，而 `traversePluginDir()` 则会递归地将它以及下面的子文件夹作为 watch 的对象，当有 socket 文件创建时会生成 Create 事件。并调用 `handleCreateEvent()` 进行插件的注册。`traversePluginDir()` 代码如下：

``` go
func (w *Watcher) traversePluginDir(dir string) error {
	err := w.fsWatcher.Add(dir)
    ...
	return w.fs.Walk(dir, func(path string, info os.FileInfo, err error) error {
        ...
		switch mode := info.Mode(); {
		case mode.IsDir():
			if err := w.fsWatcher.Add(path); err != nil {
                ...
			}
		case mode&os.ModeSocket != 0:
			event := fsnotify.Event{
				Name: path,
				Op:   fsnotify.Create,
			}
			if err := w.handleCreateEvent(event); err != nil {
                ...
			}
            ...
		}

		return nil
	})
}
```

执行完 `traversePluginDir()` 后，会监听 Watcher 的事件，代码如下：

``` go
func (w *Watcher) Start(stopCh <-chan struct{}) error {

	if err := w.traversePluginDir(w.path); err != nil {
        ...
    }

	go func(fsWatcher *fsnotify.Watcher) {
		for {
			select {
			case event := <-fsWatcher.Events:
				//TODO: Handle errors by taking corrective measures
				if event.Op&fsnotify.Create == fsnotify.Create {
					err := w.handleCreateEvent(event)
                    ...
				} else if event.Op&fsnotify.Remove == fsnotify.Remove {
					w.handleDeleteEvent(event)
				}
				continue
                ...
			case <-stopCh:
				w.fsWatcher.Close()
				return
			}
		}
	}(fsWatcher)

	return nil
}
```

当有文件创建时调用 `handleCreateEvent()` 来处理，当有文件删除时，调用 `handleDeleteEvent()` 进行处理。

``` go
func (w *Watcher) handleCreateEvent(event fsnotify.Event) error {
    ...
	if strings.HasPrefix(fi.Name(), ".") {
		klog.V(5).Infof("Ignoring file (starts with '.'): %s", fi.Name())
		return nil
	}

	if !fi.IsDir() {
		isSocket, err := util.IsUnixDomainSocket(util.NormalizePath(event.Name))
		if err != nil {
			return fmt.Errorf("failed to determine if file: %s is a unix domain socket: %v", event.Name, err)
		}
		if !isSocket {
			klog.V(5).Infof("Ignoring non socket file %s", fi.Name())
			return nil
		}

		return w.handlePluginRegistration(event.Name)
	}

	return w.traversePluginDir(event.Name)
}
```

可以看出 `handleCreateEvent()` 会忽略 `.` 开头的文件，然后忽略非 socket 文件，针对 socket 文件，则会调用 `handlePluginRegistration()` 进行注册。而如果当前函数处理的对象是一个子目录时，又会重新调用前面的 `traversePluginDir()` 来处理这个子目录。

`handlePluginRegistration()` 和 `handleDeleteEvent()` 的实现非常简单，就是分别调用 desiredStateOfWorld 的 `AddOrUpdatePlugin()` 和 `RemovePlugin()` 函数来注册和删除当前 socket 文件对应的插件。

``` go
func (w *Watcher) handlePluginRegistration(socketPath string) error {
    ...
	klog.V(2).Infof("Adding socket path or updating timestamp %s to desired state cache", socketPath)
	err := w.desiredStateOfWorld.AddOrUpdatePlugin(socketPath)
    ...
	return nil
}

func (w *Watcher) handleDeleteEvent(event fsnotify.Event) {
	klog.V(6).Infof("Handling delete event: %v", event)

	socketPath := event.Name
	klog.V(2).Infof("Removing socket path %s from desired state cache", socketPath)
	w.desiredStateOfWorld.RemovePlugin(socketPath)
}
```

注意：在 `handleDeleteEvent()` 中有一个 bug，它没有检测当前事件对应文件的类型就将其直接将其当做 socket 文件了，进而传递给 desiredStateOfWorld 的 `RemovePlugin()` 函数。
