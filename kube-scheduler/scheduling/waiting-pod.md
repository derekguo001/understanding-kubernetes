# 延时绑定 #

当由于某种原因，需要将某些 Pod 执行延时绑定，在目前的调度框架中是允许这种操作的。在本节中，对延时绑定进行代码层面的实现分析，下一节会分析在整个调度框架中如何是如何实现延时绑定的。

延时绑定的代码位于 `pkg/scheduler/framework/runtime/waiting_pods_map.go` 中。

当一个 Pod 需要延时绑定时，会有一个对应的 `waitingPod` 对象：

``` go
type waitingPod struct {
	pod            *v1.Pod
	pendingPlugins map[string]*time.Timer
	s              chan *framework.Status
	mu             sync.RWMutex
}
```

其中的

- `pendingPlugins map[string]*time.Timer` key 为插件名称，value 是一个定时器。因为针对某一个 Pod 可能有多个插件同时需要对其进行延时绑定，因此需要此字段。
- `s chan *framework.Status` 用于存储最终状态。如果处于延时绑定状态中的 Pod 被批准允许进行立即调度，那么此状态为 `framework.NewStatus(framework.Success, "")`，如果超过了预定的时间，那么 Pod 会被拒绝执行调度操作，此时状态为 `framework.NewStatus(framework.Unschedulable, msg)`。其它组件可以对监听此 channel，根据不同的最终状态进行相应的动作。

下面是创建一个需要执行延时绑定的 Pod 对象的代码。

``` go
func newWaitingPod(pod *v1.Pod, pluginsMaxWaitTime map[string]time.Duration) *waitingPod {
	wp := &waitingPod{
		pod: pod,
		s: make(chan *framework.Status, 1),
	}

	wp.pendingPlugins = make(map[string]*time.Timer, len(pluginsMaxWaitTime))
    ...
	for k, v := range pluginsMaxWaitTime {
		plugin, waitTime := k, v
		wp.pendingPlugins[plugin] = time.AfterFunc(waitTime, func() {
			msg := fmt.Sprintf("rejected due to timeout after waiting %v at plugin %v",
				waitTime, plugin)
			wp.Reject(msg)
		})
	}

	return wp
}
```

第二个参数 `pluginsMaxWaitTime map[string]time.Duration` 的 key 为插件名称，value 表示此插件需要将当前 Pod 延时多久。创建时，会创建一个定时执行的匿名函数，当超时后会执行 `wp.Reject(msg)` 禁止当前 Pod  进行调度。

在等待超时之前，可以主动停止 Pod 的等待状态，让其主动进行下一步的调度工作，通过 `Allow()` 来实现。

``` go
func (w *waitingPod) Allow(pluginName string) {
	w.mu.Lock()
	defer w.mu.Unlock()
	if timer, exist := w.pendingPlugins[pluginName]; exist {
		timer.Stop()
		delete(w.pendingPlugins, pluginName)
	}

	// Only signal success status after all plugins have allowed
	if len(w.pendingPlugins) != 0 {
		return
	}

	// The select clause works as a non-blocking send.
	// If there is no receiver, it's a no-op (default case).
	select {
	case w.s <- framework.NewStatus(framework.Success, ""):
	default:
	}
}
```

它会遍历 `w.pendingPlugins`，停止其中的每一个定时器，并删除对应的项，然后将成功状态写入 `w.s` 中。

在等待超时后，会执行默认的 `Reject()` 来禁止当前 Pod 的调度，也可以在等待超时之前，主动调用此函数来禁止调度。

``` go
func (w *waitingPod) Reject(msg string) {
	w.mu.RLock()
	defer w.mu.RUnlock()
	for _, timer := range w.pendingPlugins {
		timer.Stop()
	}

	// The select clause works as a non-blocking send.
	// If there is no receiver, it's a no-op (default case).
	select {
	case w.s <- framework.NewStatus(framework.Unschedulable, msg):
	default:
	}
}
```

执行过程为停止掉所有的定时器，然后将不可调度的信息写入 `w.s` 中。

在调度框架中可能有多个 Pod 同时执行延时绑定，因此还需要一个结构。

``` go
type waitingPodsMap struct {
	pods map[types.UID]*waitingPod
	mu   sync.RWMutex
}

func newWaitingPodsMap() *waitingPodsMap {
	return &waitingPodsMap{
		pods: make(map[types.UID]*waitingPod),
	}
}
```

其中的 `pods map[types.UID]*waitingPod` key 为 Pod 的 UID，value 为上文提到的 `waitingPod` 对象。

在 `NewFramework()` 中创建调度框架时，会调用 `newWaitingPodsMap()` 创建一个 waitingPodsMap 对象。

``` go
	f := &frameworkImpl{
		registry:              r,
        ...
		waitingPods:           newWaitingPodsMap(),
        ...
	}
```

延时绑定的使用会在 [Permit 扩展点](./permit.md) 和 [WaitOnPermit 扩展点](./wait-on-permit.md) 中分析。
