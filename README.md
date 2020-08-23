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

- Volume Plugins

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
