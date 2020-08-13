## 抢占

### 概述

当某个 node 上的 pod 无法创建时，在开启抢占功能的情况下，会尝试删除优先级低的 pod ，为原先的 pod 预留出资源，被抢占的 pod 删除后，会被调度器再次调度到其它节点。

本节内容只关注 kubelet 中抢占功能的实现，调度模块中的抢占功能请参考相应章节。

抢占功能的实现代码位于 `pkg/kubelet/preemption` 中，入口函数是 `HandleAdmissionFailure()`。

``` go
// HandleAdmissionFailure gracefully handles admission rejection, and, in some cases,
// to allow admission of the pod despite its previous failure.
func (c *CriticalPodAdmissionHandler) HandleAdmissionFailure(admitPod *v1.Pod, failureReasons []lifecycle.PredicateFailureReason) ([]lifecycle.PredicateFailureReason, error) {
	if !kubetypes.IsCriticalPod(admitPod) {
		return failureReasons, nil
	}
	// InsufficientResourceError is not a reason to reject a critical pod.
	// Instead of rejecting, we free up resources to admit it, if no other reasons for rejection exist.
	nonResourceReasons := []lifecycle.PredicateFailureReason{}
	resourceReasons := []*admissionRequirement{}
	for _, reason := range failureReasons {
		if r, ok := reason.(*lifecycle.InsufficientResourceError); ok {
			resourceReasons = append(resourceReasons, &admissionRequirement{
				resourceName: r.ResourceName,
				quantity:     r.GetInsufficientAmount(),
			})
		} else {
			nonResourceReasons = append(nonResourceReasons, reason)
		}
	}
	if len(nonResourceReasons) > 0 {
		// Return only reasons that are not resource related, since critical pods cannot fail admission for resource reasons.
		return nonResourceReasons, nil
	}
	err := c.evictPodsToFreeRequests(admitPod, admissionRequirementList(resourceReasons))
	// if no error is returned, preemption succeeded and the pod is safe to admit.
	return nil, err
}
```

这个函数的包括三个部分：

1. 判断发起抢占的 pod 是否是 Critical Pod，如果不是的话则直接返回，不会进行抢占操作。
1. 判断发起抢占的 pod 创建失败是否全是资源不足导致的，如果不是的话，则返回，不进行抢占。
1. 发起抢占操作。

下面针对这几个步骤分别进行分析。

### 一、预判断

首先，先来看第一步，即判断一个创建失败的 pod 是不是 Critical Pod，实现函数是 `IsCriticalPod()`。

``` go
// IsCriticalPod returns true if pod's priority is greater than or equal to SystemCriticalPriority.
func IsCriticalPod(pod *v1.Pod) bool {
	if IsStaticPod(pod) {
		return true
	}
	if IsMirrorPod(pod) {
		return true
	}
	if pod.Spec.Priority != nil && IsCriticalPodBasedOnPriority(*pod.Spec.Priority) {
		return true
	}
	return false
}

// IsCriticalPodBasedOnPriority checks if the given pod is a critical pod based on priority resolved from pod Spec.
func IsCriticalPodBasedOnPriority(priority int32) bool {
	return priority >= scheduling.SystemCriticalPriority
}
```


``` go
	HighestUserDefinablePriority = int32(1000000000)
	SystemCriticalPriority = 2 * HighestUserDefinablePriority
```

可以看出如果 pod 是 3 种情况之一，则认为它是 Critical Pod，可以进行抢占操作。

1. pod 为 static pod。
1. pod 为 static pod 对应的 mirror pod。
1. pod 优先级 >= 2,000,000,000。

### 二、是否有其它原因导致 pod 创建失败

代码如下：

``` go
	// InsufficientResourceError is not a reason to reject a critical pod.
	// Instead of rejecting, we free up resources to admit it, if no other reasons for rejection exist.
	nonResourceReasons := []lifecycle.PredicateFailureReason{}
	resourceReasons := []*admissionRequirement{}
	for _, reason := range failureReasons {
		if r, ok := reason.(*lifecycle.InsufficientResourceError); ok {
			resourceReasons = append(resourceReasons, &admissionRequirement{
				resourceName: r.ResourceName,
				quantity:     r.GetInsufficientAmount(),
			})
		} else {
			nonResourceReasons = append(nonResourceReasons, reason)
		}
	}
	if len(nonResourceReasons) > 0 {
		// Return only reasons that are not resource related, since critical pods cannot fail admission for resource reasons.
		return nonResourceReasons, nil
	}
```

判断发起抢占的 pod 创建失败是否全是资源不足导致的，如果不是的话，则返回，不进行抢占。原因是即使抢占成功，还有其它原因导致创建失败，抢占也就失去了意义。

### 三、抢占算法

接下来看分析如何真正进行抢占操作，实现函数是 `evictPodsToFreeRequests()`。

``` go
// evictPodsToFreeRequests takes a list of insufficient resources, and attempts to free them by evicting pods
// based on requests.  For example, if the only insufficient resource is 200Mb of memory, this function could
// evict a pod with request=250Mb.
func (c *CriticalPodAdmissionHandler) evictPodsToFreeRequests(admitPod *v1.Pod, insufficientResources admissionRequirementList) error {
	podsToPreempt, err := getPodsToPreempt(admitPod, c.getPodsFunc(), insufficientResources)
    ...

	klog.Infof("preemption: attempting to evict pods %v, in order to free up resources: %s", podsToPreempt, insufficientResources.toString())
	for _, pod := range podsToPreempt {
		status := v1.PodStatus{
			Phase:   v1.PodFailed,
			Message: message,
			Reason:  events.PreemptContainer,
		}
        ...
		err := c.killPodFunc(pod, status, nil)
        ...

		klog.Infof("preemption: pod %s evicted successfully", format.Pod(pod))
	}
	return nil
}
```

这个函数分成两个部分，找出被抢占的 pod 列表，然后使用预注册的函数将其逐个 kill 掉。我们重点关注前者，即找到哪些 pod 可以被抢占，实现函数如下。

``` go
// getPodsToPreempt returns a list of pods that could be preempted to free requests >= requirements
func getPodsToPreempt(pod *v1.Pod, pods []*v1.Pod, requirements admissionRequirementList) ([]*v1.Pod, error) {
	bestEffortPods, burstablePods, guaranteedPods := sortPodsByQOS(pod, pods)

	// make sure that pods exist to reclaim the requirements
	unableToMeetRequirements := requirements.subtract(append(append(bestEffortPods, burstablePods...), guaranteedPods...)...)
	if len(unableToMeetRequirements) > 0 {
		return nil, fmt.Errorf("no set of running pods found to reclaim resources: %v", unableToMeetRequirements.toString())
	}
	// find the guaranteed pods we would need to evict if we already evicted ALL burstable and besteffort pods.
	guaranteedToEvict, err := getPodsToPreemptByDistance(guaranteedPods, requirements.subtract(append(bestEffortPods, burstablePods...)...))
	if err != nil {
		return nil, err
	}
	// Find the burstable pods we would need to evict if we already evicted ALL besteffort pods, and the required guaranteed pods.
	burstableToEvict, err := getPodsToPreemptByDistance(burstablePods, requirements.subtract(append(bestEffortPods, guaranteedToEvict...)...))
	if err != nil {
		return nil, err
	}
	// Find the besteffort pods we would need to evict if we already evicted the required guaranteed and burstable pods.
	bestEffortToEvict, err := getPodsToPreemptByDistance(bestEffortPods, requirements.subtract(append(burstableToEvict, guaranteedToEvict...)...))
	if err != nil {
		return nil, err
	}
	return append(append(bestEffortToEvict, burstableToEvict...), guaranteedToEvict...), nil
}
```

这个函数主要的逻辑也是两个部分：将所有被抢占 pod 的列表按照 QoS 分成三类，然后分别找出这三类 pod 分别进行过滤。

#### 按 QoS 分类

按 QoS 将 pod 进行分类函数为 `sortPodsByQOS()`

``` go
// sortPodsByQOS returns lists containing besteffort, burstable, and guaranteed pods that
// can be preempted by preemptor pod.
func sortPodsByQOS(preemptor *v1.Pod, pods []*v1.Pod) (bestEffort, burstable, guaranteed []*v1.Pod) {
	for _, pod := range pods {
		if kubetypes.Preemptable(preemptor, pod) {
			switch v1qos.GetPodQOS(pod) {
			case v1.PodQOSBestEffort:
				bestEffort = append(bestEffort, pod)
			case v1.PodQOSBurstable:
				burstable = append(burstable, pod)
			case v1.PodQOSGuaranteed:
				guaranteed = append(guaranteed, pod)
			default:
			}
		}
	}

	return
}
```

``` go
// Preemptable returns true if preemptor pod can preempt preemptee pod
// if preemptee is not critical or if preemptor's priority is greater than preemptee's priority
func Preemptable(preemptor, preemptee *v1.Pod) bool {
	if IsCriticalPod(preemptor) && !IsCriticalPod(preemptee) {
		return true
	}
	if (preemptor != nil && preemptor.Spec.Priority != nil) &&
		(preemptee != nil && preemptee.Spec.Priority != nil) {
		return *(preemptor.Spec.Priority) > *(preemptee.Spec.Priority)
	}

	return false
}
```

先判断当前 pod 能否被抢占，判断的依据是当前 pod 不能为 Critical Pod，也就是说不能是 static pod、mirror pod 以及优先级 < 2,000,000,000 的 pod。另外，当前 pod 的优先级必须小于主动发起抢占的 pod 的优先级。

如果当前 pod 可以被抢占，则按照其 QoS 属性分为 bestEffort、burstable 和 guaranteed 三类。

然后计算如果将这三类所有的 pod 全部进行驱逐的话能否释放足够的资源，如果不能的话则直接返回。

#### 按照 QoS 计算所有的被抢占 pod 列表

最后，分别统计这三类 pod ，计算每一类可以驱逐的 pod 列表，代码如下。

``` go
	// find the guaranteed pods we would need to evict if we already evicted ALL burstable and besteffort pods.
	guaranteedToEvict, err := getPodsToPreemptByDistance(guaranteedPods, requirements.subtract(append(bestEffortPods, burstablePods...)...))
	if err != nil {
		return nil, err
	}
	// Find the burstable pods we would need to evict if we already evicted ALL besteffort pods, and the required guaranteed pods.
	burstableToEvict, err := getPodsToPreemptByDistance(burstablePods, requirements.subtract(append(bestEffortPods, guaranteedToEvict...)...))
	if err != nil {
		return nil, err
	}
	// Find the besteffort pods we would need to evict if we already evicted the required guaranteed and burstable pods.
	bestEffortToEvict, err := getPodsToPreemptByDistance(bestEffortPods, requirements.subtract(append(burstableToEvict, guaranteedToEvict...)...))
	if err != nil {
		return nil, err
	}
	return append(append(bestEffortToEvict, burstableToEvict...), guaranteedToEvict...), nil
```

这里的逻辑是这样的，先计算需要驱逐的 guaranteed 类型的 pod 列表，假如所有的 bestEffort 和 burstable 类型的 pod 全部驱逐之后，还不能释放足够的资源，在这情况下还需要驱逐哪些 guaranteed 类型的 pod 才能满足要求。将这些 pod 存入 guaranteedToEvict 中。

然后计算需要驱逐的 burstable 类型的 pod 列表，计算过程是假如所有的 bestEffort 类型的 pod 全部被驱逐，并且将刚才计算出来的需要驱逐的 guaranteed 类型的 pod 全部驱逐，还不能释放足够的资源，这种情况下还需要驱逐哪些 burstable 类型的 pod，将其存入 burstableToEvict 。

最后计算需要驱逐的 bestEffort 类型的 pod 列表，计算过程是假如刚才计算出来的所有的 burstable 和 guaranteed 类型的 pod 全部被驱逐(即 burstableToEvict + guaranteedToEvict 中的所有 pod)，还不能释放足够的资源，这种情况下还需要驱逐哪些 bestEffort 类型的 pod，将其存入 bestEffortToEvict 。

然后将这三个列表 append 到一起返回，顺序是 `(bestEffortToEvict, burstableToEvict, guaranteedToEvict)`。

这个算法有一些绕，需要仔细分析背后的逻辑。并且这三个步骤和最后的 append 到一起的 pod 列表的顺序都非常重要，不能颠倒。有一个核心逻辑，即这三类 QoS 按照重要性来分是 `guaranteed > burstable > bestEffort`，会优先抢占 QoS 重要性低的 pod。

举一些例子会更容易理解。

1. 假如驱逐一部分 bestEffort 类型的 pod 就可以满足要求，那么 bestEffortToEvict 不为空，burstableToEvict 和 guaranteedToEvict 为空。
1. 假如驱逐所有的 bestEffort 类型的 pod 和一部分 burstable 类型的 pod 才可以满足要求，那么 bestEffortToEvict 包含所有的 bestEffort 类型的 pod，burstableToEvict 包含一部分需要驱逐的 burstable 类型的 pod，guaranteedToEvict 为空。

上面这个算法中用到了两个核心的函数 `requirements.subtract()` 和 `getPodsToPreemptByDistance()`

- `requirements.subtract()` 用来计算总共所需资源减去参数中 pod 列表所占资源后还剩哪些资源，具体逻辑不在这里展开。

- `getPodsToPreemptByDistance()` 第一个参数是 pod 列表，第二个参数是需要释放的资源量。这个函数的作用是计算需要驱逐哪些 pod 才可以释放第二个参数指定的资源。这个函数的代码如下。

``` go
// getPodsToPreemptByDistance finds the pods that have pod requests >= admission requirements.
// Chooses pods that minimize "distance" to the requirements.
// If more than one pod exists that fulfills the remaining requirements,
// it chooses the pod that has the "smaller resource request"
// This method, by repeatedly choosing the pod that fulfills as much of the requirements as possible,
// attempts to minimize the number of pods returned.
func getPodsToPreemptByDistance(pods []*v1.Pod, requirements admissionRequirementList) ([]*v1.Pod, error) {
	podsToEvict := []*v1.Pod{}
	// evict pods by shortest distance from remaining requirements, updating requirements every round.
	for len(requirements) > 0 {
		if len(pods) == 0 {
			return nil, fmt.Errorf("no set of running pods found to reclaim resources: %v", requirements.toString())
		}
		// all distances must be less than len(requirements), because the max distance for a single requirement is 1
		bestDistance := float64(len(requirements) + 1)
		bestPodIndex := 0
		// Find the pod with the smallest distance from requirements
		// Or, in the case of two equidistant pods, find the pod with "smaller" resource requests.
		for i, pod := range pods {
			dist := requirements.distance(pod)
			if dist < bestDistance || (bestDistance == dist && smallerResourceRequest(pod, pods[bestPodIndex])) {
				bestDistance = dist
				bestPodIndex = i
			}
		}
		// subtract the pod from requirements, and transfer the pod from input-pods to pods-to-evicted
		requirements = requirements.subtract(pods[bestPodIndex])
		podsToEvict = append(podsToEvict, pods[bestPodIndex])
		pods[bestPodIndex] = pods[len(pods)-1]
		pods = pods[:len(pods)-1]
	}
	return podsToEvict, nil
}
```

当需求的资源量大于零的时候，这个函数会遍历所有的 pod 列表，每次找出一个占用这种资源最小的 pod，然后将这个 pod append 到最终返回列表里，用候选 pod 列表中的最后一个元素覆盖刚才找到的 pod，然后删除最后一个元素，通过这种方式将找到的 pod 从候选 pod 列表中删除，然后进行下一次循环。最终找到所有的需要驱逐的 pod 列表并返回。

以上是 kubelet 中抢占功能的主要实现逻辑，其中最重要的是如何筛选出被抢占的 pod 列表。
