# Profile #

Profile 是在 Kubernetes Scheduler 代码中完成初始化后，在调度过程中使用的一个对象。

``` go
// Profile is a scheduling profile.
type Profile struct {
	framework.Framework
	Recorder events.EventRecorder
}

// Map holds profiles indexed by scheduler name.
type Map map[string]*Profile
```

Profile 结构体中包含了一个 framework.Framework 对象，也就是调度框架对象。

如果有多个调度器，那么每一个调度器的名称会作为 key，对应的包含了调度器框架的 Profile 对象会作为 value。这样便将多个调度器和对应的 Profile 整合到了一个 Map 对象中。在 Kubernetes Scheduler 执行调度决策的过程中，会从 Pod 中获取调度器名称，然后根据名称再从这里的 Map 对象中找到对应的 Profile 对象，再使用 Profile 对象执行具体的调度过程。

## 初始化 ##

下面分析 Profile 的创建过程。

在 [KubeSchedulerProfile](kube-scheduler-profile.md) 中提到了会在 `createFromProvider()` 中完成对 KubeSchedulerProfile 对象的初始化过程。

``` go
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	klog.V(2).Infof("Creating scheduler from algorithm provider '%v'", providerName)
	r := algorithmprovider.NewRegistry()
	defaultPlugins, exist := r[providerName]
	if !exist {
		return nil, fmt.Errorf("algorithm provider %q is not registered", providerName)
	}

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

在这个函数的末尾有一个 `create()` 调用，在其中会完成从 KubeSchedulerProfile 到 Profile 的转换。

``` go
// create a scheduler from a set of registered plugins.
func (c *Configurator) create() (*Scheduler, error) {
    ...
	profiles, err := profile.NewMap(c.profiles, c.buildFramework, c.recorderFactory)
	if err != nil {
		return nil, fmt.Errorf("initializing profiles: %v", err)
	}
	if len(profiles) == 0 {
		return nil, errors.New("at least one profile is required")
	}
    ...

	return &Scheduler{
		SchedulerCache:  c.schedulerCache,
		Algorithm:       algo,
		Profiles:        profiles,
		NextPod:         internalqueue.MakeNextPodFunc(podQueue),
		Error:           MakeDefaultErrorFunc(c.client, podQueue, c.schedulerCache),
		StopEverything:  c.StopEverything,
		VolumeBinder:    c.volumeBinder,
		SchedulingQueue: podQueue,
	}, nil
}
```

这里会使用 `profile.NewMap()` 来创建 Profile 对象，然后将其存入 Scheduler 的 `Profiles profile.Map` 字段中。

现在回头来看创建过程，`profile.NewMap()` 的第二个参数是一个回调函数：

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

作用只是简单地创建一个 framework.Framework 对象，即调度框架。下面分析 `profile.NewMap()`。

``` go
func NewMap(cfgs []config.KubeSchedulerProfile, frameworkFact FrameworkFactory, recorderFact RecorderFactory) (Map, error) {
	m := make(Map)
	v := cfgValidator{m: m}

	for _, cfg := range cfgs {
		if err := v.validate(cfg); err != nil {
			return nil, err
		}
		p, err := NewProfile(cfg, frameworkFact, recorderFact)
		if err != nil {
			return nil, fmt.Errorf("creating profile for scheduler name %s: %v", cfg.SchedulerName, err)
		}
		m[cfg.SchedulerName] = p
	}
	return m, nil
}

func NewProfile(cfg config.KubeSchedulerProfile, frameworkFact FrameworkFactory, recorderFact RecorderFactory) (*Profile, error) {
	f, err := frameworkFact(cfg)
	if err != nil {
		return nil, err
	}
	r := recorderFact(cfg.SchedulerName)
	return &Profile{
		Framework: f,
		Recorder:  r,
	}, nil
}
```

`profile.NewMap()` 会遍历 Configurator 中的 `profiles []schedulerapi.KubeSchedulerProfile` 对象。首先会验证其有效性，如果通过，则会调用 `NewProfile()` 创建 Profile 对象，最后将这些 Profile 对象按照调度器名称作为 key 的方式存入 Map，最后返回这个 Map。

而在 `NewProfile()` 中则是简单地调用传入的回调函数来创建 framework.Framework 对象并返回，这里的回调函数其实就是 `buildFramework()`。

至此，可以知道 Profile 对象是如何创建的，这里主要是如何构造一个由调度器名称作为 key 的 Map 对象。核心的逻辑其实是如何创建 framework.Framework，可参见 [调度框架](./framework.md)。

## 使用 ##

调度的主流程在 `scheduleOne()` 中。

``` go
func (sched *Scheduler) scheduleOne(ctx context.Context) {
	podInfo := sched.NextPod()
	// pod could be nil when schedulerQueue is closed
	if podInfo == nil || podInfo.Pod == nil {
		return
	}
	pod := podInfo.Pod
	prof, err := sched.profileForPod(pod)

    ...
}
```

``` go
func (sched *Scheduler) profileForPod(pod *v1.Pod) (*profile.Profile, error) {
	prof, ok := sched.Profiles[pod.Spec.SchedulerName]
	if !ok {
		return nil, fmt.Errorf("profile not found for scheduler name %q", pod.Spec.SchedulerName)
	}
	return prof, nil
}
```

可以看出在调度时会从 Pod 中获取调度器名称，然后根据名称再从这里的 Map 对象中找到对应的 profile 对象，再使用 profile 以及它内部的 framework.Framework 对象来执行具体的调度过程。
