# NodePorts 调度插件 #

NodePorts 调度插件用来检查 Pod 请求的端口在节点上是否可用，它实现了 `PreFilter` 和 `Filter` 扩展点。

## PreFilter ##

``` go
func (pl *NodePorts) PreFilter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod) *framework.Status {
	s := getContainerPorts(pod)
	cycleState.Write(preFilterStateKey, preFilterState(s))
	return nil
}

func getContainerPorts(pods ...*v1.Pod) []*v1.ContainerPort {
	ports := []*v1.ContainerPort{}
	for _, pod := range pods {
		for j := range pod.Spec.Containers {
			container := &pod.Spec.Containers[j]
			for k := range container.Ports {
				ports = append(ports, &container.Ports[k])
			}
		}
	}
	return ports
}
```

PreFilter 扩展点会搜集当前 Pod 所使用的所有 Port，让它们存入到调度器缓存中。

## Filter ##

``` go
func (pl *NodePorts) Filter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
	wantPorts, err := getPreFilterState(cycleState)
	if err != nil {
		return framework.AsStatus(err)
	}

	fits := fitsPorts(wantPorts, nodeInfo)
	if !fits {
		return framework.NewStatus(framework.Unschedulable, ErrReason)
	}

	return nil
}
```

Filter 扩展点首先会使用 `getPreFilterState()` 取回在 PreFilter 阶段存入调度器缓存中的数据，即当前 Pod 所使用的所有 Port 的列表。然后使用 `fitsPorts()` 来确定和当前节点上已使用的 Port 是否冲突。

``` go
func fitsPorts(wantPorts []*v1.ContainerPort, nodeInfo *framework.NodeInfo) bool {
	existingPorts := nodeInfo.UsedPorts
	for _, cp := range wantPorts {
		if existingPorts.CheckConflict(cp.HostIP, string(cp.Protocol), cp.HostPort) {
			return false
		}
	}
	return true
}
```

这里比较的是 ContainerPort.HostIP、ContainerPort.Protocol 和 ContainerPort.HostPort 三元组，因为不同的三元组组合是可以共存的。

``` go
func (h HostPortInfo) CheckConflict(ip, protocol string, port int32) bool {
	if port <= 0 {
		return false
	}

	h.sanitize(&ip, &protocol)

	pp := NewProtocolPort(protocol, port)

	// If ip is 0.0.0.0 check all IP's (protocol, port) pair
	if ip == DefaultBindAllHostIP {
		for _, m := range h {
			if _, ok := m[*pp]; ok {
				return true
			}
		}
		return false
	}

	// If ip isn't 0.0.0.0, only check IP and 0.0.0.0's (protocol, port) pair
	for _, key := range []string{DefaultBindAllHostIP, ip} {
		if m, ok := h[key]; ok {
			if _, ok2 := m[*pp]; ok2 {
				return true
			}
		}
	}

	return false
}
```

其中的 `0.0.0.0` 地址需要做特殊处理，相当于 "所有的IP"，因此如果 ContainerPort.HostIP 为 `0.0.0.0` 的话，则需要匹配所有地址对应的协议和端口号组合。
