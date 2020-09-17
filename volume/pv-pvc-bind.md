# 关于 PV 和 PVC 的绑定

## 概述

用户创建了 PV 和 PVC 之后，二者会尝试绑定，这种绑定关系是一对一的。

绑定完成后，PV 和 PVC 对象都会增加一些信息来引用对方。

``` bash
$ kubectl get pv pv1 -o yaml
apiVersion: v1
kind: PersistentVolume
...
spec:
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: pvc1
    namespace: default
    resourceVersion: "1390698"
    uid: a72fe5c9-a5d6-4214-8047-3250624e96f1
  ...
status:
  phase: Bound
```

PV 中会增加一个 `spec.claimRef` 字段，里面包含所绑定的 PVC 的信息。这个信息也可以由用户自己指定，例如用户在创建 PV 对象时指定 `spec.claimRef` 中 PVC 的 name 和 namespace 信息，这种实际上是由用户完成 PV 到 PVC 的绑定，当创建完成后，会自动到集群中找到指定名称的 PVC 对象，然后将其 uid 更新到 `spec.claimRef.uid` 字段中。

``` bash
$ kubectl get pvc pvc1 -o yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
...
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeMode: Filesystem
  volumeName: pv1
```

PVC 对象是通过其中的 `spec.volumeName` 来反向引用它绑定的 PV 对象的，由于 PV 是全局对象，因此这里不需要 namespace 字段。

## 手动指定绑定对象

用户在创建 PV 对象的时候，可以明确地指定和哪个 PVC 对象绑定，例如：

例如用户按照以下格式创建 PV 对象：

``` yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  iscsi:
     targetPortal: 178.104.163.35:3260
     iqn: iqn.2020-09.com.test:lv1
     lun: 0
     fsType: 'ext4'
     readOnly: false
  claimRef:
    name: pvc1
    namespace: default
EOF
```

在指定 pvc 名称的同时必须指定对应的 namespace，因为 PV 是全局的，但是 PVC 却是必须属于某个 namespace 的。

同样，用户在创建 PVC 对象的时候，也可以明确地指定和哪个 PV 对象绑定，例如：

``` yaml
kubectl apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc1
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: pv1
EOF
```

在手动绑定时，可以只指定一方，例如创建 PV 时指定 PVC，但是创建 PVC 的时候不指定 PV；或者创建 PV 的时候不指定 PVC，但创建 PVC 的时候指定 PV。

也可以同时指定对方，这种情况下需要双方的名称都匹配才可以正确地绑定。

## 延迟绑定

绑定也可以延迟进行，这就是所谓的 **延迟绑定**，需要对应的 StorageClass 对象的绑定模式为 `WaitForFirstConsumer`。

``` bash
const (
	// VolumeBindingImmediate indicates that PersistentVolumeClaims should be
	// immediately provisioned and bound.  This is the default mode.
	VolumeBindingImmediate VolumeBindingMode = "Immediate"

	// VolumeBindingWaitForFirstConsumer indicates that PersistentVolumeClaims
	// should not be provisioned and bound until the first Pod is created that
	// references the PeristentVolumeClaim.  The volume provisioning and
	// binding will occur during Pod scheduing.
	VolumeBindingWaitForFirstConsumer VolumeBindingMode = "WaitForFirstConsumer"
)
```
