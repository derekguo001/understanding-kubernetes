# PrioritySort 调度插件 #

PrioritySort 调度插件是调度器默认使用的对 Pod 队列进行排序的插件，顾名思义，它使用 Pod 的优先级来对 Pod 进行排序。代码位于 `./pkg/scheduler/framework/plugins/queuesort/priority_sort.go`。

``` go
// Less is the function used by the activeQ heap algorithm to sort pods.
// It sorts pods based on their priority. When priorities are equal, it uses
// PodQueueInfo.timestamp.
func (pl *PrioritySort) Less(pInfo1, pInfo2 *framework.QueuedPodInfo) bool {
	p1 := corev1helpers.PodPriority(pInfo1.Pod)
	p2 := corev1helpers.PodPriority(pInfo2.Pod)
	return (p1 > p2) || (p1 == p2 && pInfo1.Timestamp.Before(pInfo2.Timestamp))
}
```

核心实现是一个 `Less()` 函数。如果两个 Pod 的优先级一样，则按照 Pod 被加到调度队列中的时间顺序进行排序，否则就按照优先级排序。返回优先级的函数如下：

``` go
// PodPriority returns priority of the given pod.
func PodPriority(pod *v1.Pod) int32 {
	if pod.Spec.Priority != nil {
		return *pod.Spec.Priority
	}
	// When priority of a running pod is nil, it means it was created at a time
	// that there was no global default priority class and the priority class
	// name of the pod was empty. So, we resolve to the static default priority.
	return 0
}
```

如果没有指定优先级，则认为其优先级为0。
