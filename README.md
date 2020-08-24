# 深入理解 Kubernetes — 通过阅读源代码理解 Kubernetes 的实现原理

# 目录

## Kubernetes API Server

## Kubernetes Scheduler

## Kubernetes Controller Manager

- [Namespace](kube-controller-manager/namespace.md)
- Deployment
- StatefulSet
- DaemonSet

## Kubelet

- [抢占](kubelet/preemption.md)

## Kube Proxy

## Kubectl

## Volume

- 概述

- Volume Plugin 框架和 Volume Plugin Manager

  - Volume 相关的数据结构
  - Volume Plugin 框架
  - [Volume Plugin Manager](volume/plugin-manager.md)

- Volume in Kube-Controller-Manager

  - AttachDetach Controller

    - 概述

  - PersistentVolume Controller

    - 概述
    - PV Controller
    - [PVC Controller](kube-controller-manager/volume/persistentvolume/pvc.md)

  - Expand Controller

- Volume in Kube-Scheduler

- Volume Manager in Kubelet

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

- CSI
