# desiredStateOfWorldPopulator #

desiredStateOfWorldPopulator 会周期性地根据 Kube API Server 的内容来同步 desiredStateOfWorld 中的信息，同步周期为一分钟。代码位于 `pkg/controller/volume/attachdetach/populator`。

执行 `Run()` 进行启动。

``` go
func (dswp *desiredStateOfWorldPopulator) Run(stopCh <-chan struct{}) {
	wait.Until(dswp.populatorLoopFunc(), dswp.loopSleepDuration, stopCh)
}

func (dswp *desiredStateOfWorldPopulator) populatorLoopFunc() func() {
	return func() {
		dswp.findAndRemoveDeletedPods()

		// findAndAddActivePods is called periodically, independently of the main
		// populator loop.
		if time.Since(dswp.timeOfLastListPods) < dswp.listPodsRetryDuration {
			klog.V(5).Infof(
				"Skipping findAndAddActivePods(). Not permitted until %v (listPodsRetryDuration %v).",
				dswp.timeOfLastListPods.Add(dswp.listPodsRetryDuration),
				dswp.listPodsRetryDuration)

			return
		}
		dswp.findAndAddActivePods()
	}
}
```

这里主要执行 `findAndRemoveDeletedPods()` 和 `findAndAddActivePods()`。

其中涉及到了两个时间，`dswp.loopSleepDuration` 和 `dswp.listPodsRetryDuration`，他们分别对应于：

``` go
	DesiredStateOfWorldPopulatorLoopSleepPeriod:       1 * time.Minute,
	DesiredStateOfWorldPopulatorListPodsRetryDuration: 3 * time.Minute,
```

第一个表示执行 `populatorLoopFunc()` 的间隔时间，第二个表示执行 `dswp.findAndAddActivePods()` 的间隔时间。

下面对这里涉及的两个主要的函数进行分析。
