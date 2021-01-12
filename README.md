# Kubernetes源代码解析

# 目录

## Kubernetes API Server

## Kubernetes Scheduler

- Kubernetes Scheduler 中的组件和启动过程

  - [参数初始化过程分析](kube-scheduler/component/option.md)
  - [Algorithm Provider](./kube-scheduler/component/algorithm-provider.md)
  - [Schedule Algorithm](./kube-scheduler/component/schedule-algorithm.md)
  - [KubeSchedulerProfile](./kube-scheduler/component/kube-scheduler-profile.md)
  - [Profile](./kube-scheduler/component/profile.md)
  - [调度插件对象和扩展点对象](./kube-scheduler/component/plugin-and-extensionpoint.md)
  - [framework.Registry](./kube-scheduler/component/framework-registry.md)
  - [Kubernetes 调度框架](./kube-scheduler/component/framework.md)

- 调度过程分析

  - [PreFilter 扩展点](./kube-scheduler/scheduling/pre-filter.md)
  - [Filter 扩展点](./kube-scheduler/scheduling/filter.md)
  - [PreScore 扩展点](./kube-scheduler/scheduling/pre-score.md)
  - [Score 整体流程](./kube-scheduler/scheduling/score/overview.md)
    - [Score 和 NormalizeScore 扩展点](./kube-scheduler/scheduling/score/score-and-normalizescore.md)
  - [Reserve 扩展点](./kube-scheduler/scheduling/reserve.md)
  - [延时绑定](./kube-scheduler/scheduling/waiting-pod.md)
    - [Permit 扩展点](./kube-scheduler/scheduling/permit.md)
    - [WaitOnPermit 扩展点](./kube-scheduler/scheduling/wait-on-permit.md)
  - [PreBind 扩展点](./kube-scheduler/scheduling/pre-bind.md)

- 调度算法详解

  - [PrioritySort](kube-scheduler/scheduler-plugins/priority-sort.md)
  - [DefaultBinder](kube-scheduler/scheduler-plugins/default-binder.md)

  - [NodeName](kube-scheduler/scheduler-plugins/node-name.md)
  - [NodePorts](kube-scheduler/scheduler-plugins/node-ports.md)
  - [NodeLabel](kube-scheduler/scheduler-plugins/node-label.md)
  - [NodeUnschedulable](kube-scheduler/scheduler-plugins/node-unschedulable.md)
  - [SelectorSpread](kube-scheduler/scheduler-plugins/selector-spread.md)

## Kubernetes Controller Manager

- [Namespace](kube-controller-manager/namespace.md)
- Deployment
- StatefulSet
- DaemonSet

## Kubelet

- [Kubelet 中的插件管理](kubelet/pluginmanager/overview.md)

  - [desiredStateOfWorld 与 actualStateOfWorld](kubelet/pluginmanager/desiredStateOfWorld_actualStateOfWorld.md)
  - [desiredStateOfWorldPopulator](kubelet/pluginmanager/desiredStateOfWorldPopulator.md)
  - [reconciler](kubelet/pluginmanager/reconciler.md)

- [抢占](kubelet/preemption.md)

## Kube Proxy

## Kubectl

## Volume

- [Kubernetes 中的 Volume 概述](volume/overview.md)

  - [关于 PV 和 PVC 的绑定](volume/pv-pvc-bind.md)
  - [关于 attach 和 detach 操作](volume/attach-detach.md)
  - [关于 mount 和 umount 操作](volume/mount-umount.md)
  - 关于 Annotation

- Volume Plugin 框架和 Volume Plugin Manager

  - [Volume 相关的数据结构](volume/volume-interface.md)
  - [Volume Plugin 框架](volume/plugin.md)
  - [Volume Plugin Manager](volume/plugin-manager.md)
  - [Volume 操作的真正执行者 OperationExecutor](volume/operationexecutor.md)

- Volume in Kube-Controller-Manager

  - [AttachDetach Controller](kube-controller-manager/volume/attachdetach/overview.md)

    - [actualStateOfWorld](kube-controller-manager/volume/attachdetach/actualstateofworld.md)
    - desiredStateOfWorld
    - [desiredStateOfWorldPopulator](kube-controller-manager/volume/attachdetach/desiredstateofworldpopulator.md)
    - reconciler
    - AttachDetach Controller 核心实现

  - [PersistentVolume Controller](kube-controller-manager/volume/persistentvolume/overview.md)

    - [PV Controller](kube-controller-manager/volume/persistentvolume/pv-controller.md)
    - [PVC Controller](kube-controller-manager/volume/persistentvolume/pvc-controller.md)

  - Expand Controller

  - PVProtectionController 和 PVCProtectionController

- [Volume in Kube-Scheduler](kube-scheduler/volume/overview.md)

  - [Volume 相关操作在调度过程中的执行时机](kube-scheduler/volume/scheduler-volume-binder.md)
  - 预选阶段 Volume 相关操作
  - PVC 和 PV 的预绑定
  - PVC 和 PV 的绑定

- [Volume in Kubelet](kubelet/volume/overview.md)

  - actualStateOfWorld
  - desiredStateOfWorld
  - desiredStateOfWorldPopulator
  - reconciler

- CSI

- 各种 Volume Plugins 详解

  - 概述
  - hostpath
  - localvolume
  - configmap
  - secret
  - nfs
  - emptydir
  - projected
  - iscsi

