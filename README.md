# Kubernetes源代码解析

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

- [Kubernetes 中的 Volume 概述](volume/overview.md)

- Volume Plugin 框架和 Volume Plugin Manager

  - Volume 相关的数据结构
  - Volume Plugin 框架
  - [Volume Plugin Manager](volume/plugin-manager.md)
  - [Volume 操作的真正执行者 OperationExecutor](volume/operationexecutor.md)

- Volume in Kube-Controller-Manager

  - AttachDetach Controller

    - 概述

  - [PersistentVolume Controller](kube-controller-manager/volume/overview.md)

    - PV Controller
    - [PVC Controller](kube-controller-manager/volume/persistentvolume/pvc.md)

  - Expand Controller

- Volume in Kube-Scheduler

- [Volume in Kubelet](kubelet/volume/overview.md)

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
