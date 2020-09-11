# 关于 mount 和 umount 操作

在执行 mount/umount 操作的时候，会涉及到很多挂载点，容易混淆。

1. 局部挂载点

   即与 Pod 关联的 Volume 挂载点，这个就是某个 Pod 关联某个 PVC 时，Pod 对应的 Volume 在节点上的路径。具体分为两种情况：

   - 当存储作为文件系统提供给 Pod 使用时，格式为 `pods/{podUid}/volume/{escapeQualifiedPluginName}/{volumeName}`。
     例如 FC SAN 的局部挂载点为 `/var/lib/kubelet/pods/50fbcc62-0cb2-4b19-975b-542ae1cedbca/volumes/kubernetes.io~fc/pv1`。

   - 当存储作为块设备提供给 Pod 使用时，格式为 `pods/{podUid}/volumeDevices/{escapeQualifiedPluginName}/{volumeName}`。

2. 全局挂载点

   由于一个存储设备同时只能挂载到一个目录中，但在 Kubernetes 中一个 Volume 却可以同时被一个节点上的多个 Pod 使用，为了解决这个问题，可以先将存储设备挂载到节点上一个全局挂载点，然后再将这个全局挂载点以 bind 的模式挂载到各个 Pod 的 Volume 对应的目录。因此对于底层需要执行 mount 操作的存储类型，在执行 mount 操作时实际上内部执行了两次 mount。

   当存储作为块设备映射到 Pod 中使用时，全局挂载点是一个目录。执行局部挂载点的 mount 操作时，会在全局挂载点目录中创建对应的信息，详见下面的 demo。

## demo

下面用几个例子以更直观的方式 demo 一下 mount 相关的操作。

### 当存储作为文件系统使用 ###

首先，第一个例子是将存储设备 /dev/sda 作为文件系统挂载到多个目录上，模拟一个 PV 作为文件系统同时挂载到多个 Pod 中。

``` bash
# mkdir /mnt/tmp{1,2,3}
# mount /dev/sda /mnt/tmp1
# mount --bind /mnt/tmp1 /mnt/tmp2
# mount --bind /mnt/tmp1 /mnt/tmp3
# mount | grep sda
/dev/sda on /mnt/tmp1 type ext4 (rw,relatime,data=ordered)
/dev/sda on /mnt/tmp2 type ext4 (rw,relatime,data=ordered)
/dev/sda on /mnt/tmp3 type ext4 (rw,relatime,data=ordered)
```

接下来一个例子是使用 iSCSI Volume Plugin 将存储设备作为文件系统使用的情况。

``` bash
kubectl create -f - <<EOF
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
EOF

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
EOF

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
spec:
  volumes:
    - name: pvc1
      persistentVolumeClaim:
        claimName: pvc1
  containers:
    - name: busybox
      image: busybox
      command:
        - sleep
        - "36000000"
      volumeMounts:
        - mountPath: "/pod/data"
          name: pvc1
EOF
```

iSCSI 存储设备为 `/dev/sda`。完成挂载后它会先挂载到全局挂载点，然后再将全局挂载点以 bind 模式挂载到 Pod 上的局部挂载点。

`/var/lib/kubelet/plugins/kubernetes.io/iscsi/iface-default/178.104.163.35:3260-iqn.2020-09.com.test:lv1-lun-0` 为全局挂载点，`/var/lib/kubelet/pods/b70c2bbd-8d97-4770-8e28-d8ed21f6c719/volumes/kubernetes.io~iscsi/pv1` 为局部挂载点。

``` bash
# mount | grep /dev/sda
/dev/sda on /var/lib/kubelet/plugins/kubernetes.io/iscsi/iface-default/178.104.163.35:3260-iqn.2020-09.com.test:lv1-lun-0 type ext4 (rw,relatime,data=ordered)
/dev/sda on /var/lib/kubelet/pods/b70c2bbd-8d97-4770-8e28-d8ed21f6c719/volumes/kubernetes.io~iscsi/pv1 type ext4 (rw,relatime,data=ordered)
# tree /var/lib/kubelet/plugins/kubernetes.io/iscsi
/var/lib/kubelet/plugins/kubernetes.io/iscsi
├── iface-default
│   └── 178.104.163.35:3260-iqn.2020-09.com.test:lv1-lun-0
│       └── lost+found
└── volumeDevices
    └── iface-default

5 directories, 0 files
#
```

### 当存储作为块设备使用 ###

下面一个例子是 iSCSI Volume Plugin，将存储设备作为块设备使用，并且两个 Pod 同时引用同一个 PVC 的情况。

创建资源的命令如下：

``` bash
kubectl create -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  volumeMode: Block
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  iscsi:
     targetPortal: 178.104.163.35:3260
     iqn: iqn.2020-09.com.test:lv1
     lun: 0
     readOnly: false
EOF

kubectl apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc1
spec:
  volumeMode: Block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
spec:
  volumes:
    - name: pvc1
      persistentVolumeClaim:
        claimName: pvc1
  containers:
    - name: busybox
      image: busybox
      command:
        - sleep
        - "36000000"
      volumeDevices:
        - devicePath: /dev/xvda
          name: pvc1
EOF

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
spec:
  volumes:
    - name: pvc1
      persistentVolumeClaim:
        claimName: pvc1
  containers:
    - name: busybox
      image: busybox
      command:
        - sleep
        - "36000000"
      volumeDevices:
        - devicePath: /dev/xvda
          name: pvc1
EOF
```

底层的 iSCSI 磁盘 attach 到节点上之后的路径为 `/dev/sda`。

下面是全局挂载点这个目录的结构，具体内容与各个 Volume Plugin 自己的实现相关。

``` bash
# tree /var/lib/kubelet/plugins/kubernetes.io/iscsi/
/var/lib/kubelet/plugins/kubernetes.io/iscsi/
├── iface-default
└── volumeDevices
    └── iface-default
        └── 178.104.163.35:3260-iqn.2020-09.com.test:lv1-lun-0
            ├── 34ce8ee5-f9b6-4851-8014-ed9ceab8a094
            ├── e40386c1-7d83-447e-88a6-d8278c9e7971
            └── iscsi.json

4 directories, 3 files
```

其中两个以 Pod UID 命名的文件都是 block 设备，是将 `/dev/sda` 按照 bind 模式 mount 到上面之后的结果，在 mount 之前这两个文件是普通文件。

``` bash
# file /var/lib/kubelet/plugins/kubernetes.io/iscsi/volumeDevices/iface-default/178.104.163.35\:3260-iqn.2020-09.com.test\:lv1-lun-0/34ce8ee5-f9b6-4851-8014-ed9ceab8a094
/var/lib/kubelet/plugins/kubernetes.io/iscsi/volumeDevices/iface-default/178.104.163.35:3260-iqn.2020-09.com.test:lv1-lun-0/34ce8ee5-f9b6-4851-8014-ed9ceab8a094: block special
# blkid /var/lib/kubelet/plugins/kubernetes.io/iscsi/volumeDevices/iface-default/178.104.163.35\:3260-iqn.2020-09.com.test\:lv1-lun-0/34ce8ee5-f9b6-4851-8014-ed9ceab8a094
/var/lib/kubelet/plugins/kubernetes.io/iscsi/volumeDevices/iface-default/178.104.163.35:3260-iqn.2020-09.com.test:lv1-lun-0/34ce8ee5-f9b6-4851-8014-ed9ceab8a094: UUID="12affb26-7606-4da4-bd3f-5c21f6e36c0c" TYPE="ext4"
# blkid /dev/sda
/dev/sda: UUID="12affb26-7606-4da4-bd3f-5c21f6e36c0c" TYPE="ext4"
#
```

其中还有一个 json 文件，保存有当前 iSCSI 存储的元数据信息。

``` bash
# cat /var/lib/kubelet/plugins/kubernetes.io/iscsi/volumeDevices/iface-default/178.104.163.35\:3260-iqn.2020-09.com.test\:lv1-lun-0/iscsi.json
{"VolName":"pv1","Portals":["178.104.163.35:3260"],"Iqn":"iqn.2020-09.com.test:lv1","Lun":"0","InitIface":"default","Iface":"default","InitiatorName":"","MetricsProvider":null}
#
```

在 Pod 对应的 Volume 路径上，有一个指向 iSCSI 设备的符号链接文件，通过这个文件将设备映射到 Pod 内部。

``` bash
# ls -l /var/lib/kubelet/pods/34ce8ee5-f9b6-4851-8014-ed9ceab8a094/volumeDevices/kubernetes.io~iscsi/pv1
lrwxrwxrwx 1 root root 8 9月  10 16:20 /var/lib/kubelet/pods/34ce8ee5-f9b6-4851-8014-ed9ceab8a094/volumeDevices/kubernetes.io~iscsi/pv1 -> /dev/sda
#
```

## mount 和 umount 相关的 interface

   ``` go
   type DeviceMounter interface {
       GetDeviceMountPath(spec *Spec) (string, error)
       MountDevice(spec *Spec, devicePath string, deviceMountPath string) error
   }
   ```

   当存储作为文件系统提供给 Pod 使用时，`DeviceMounter` interface 可以用来执行全局挂载操作，其中的 `GetDeviceMountPath()` 返回的就是文件系统的全局挂载点。

   当存储作为块设备提供给 Pod 使用时，`BlockVolume` interface 的 `GetGlobalMapPath()` 返回的就是块设备的全局挂载点。

   全局挂载点的具体位置与 Volume Plugin 的实现有关，当存储无论作为文件系统还是作为块设备提供给 Pod 使用时，全局路径都为 `plugins/kubernetes.io/{PluginName}/{DefaultKubeletVolumeDevicesDirName}/{volumePluginDependentPath}`，也就是说最后一个字段由 Volume Plugin 自己决定。例如：FC SAN 的全局挂载点为 `/var/lib/kubelet/plugins/kubernetes.io/fc/201290b11c051614-203290b11c051614-lun-0`。

下表是各种 interface 和其对应的函数与各种 umount 操作的对应关系，可参考 [Volume 相关的数据结构](volume-interface.md) 和 [Volume Plugin 框架](plugin.md)

|存储提供方式|挂载点类型|实现函数所属的interface|实现函数|格式(都位于`/var/lib/kubelet`目录下)|
|---|---|---|---|---|
|文件系统|局部挂载点|Volume|`GetPath()`|`pods/{podUid}/volume/{escapeQualifiedPluginName}/{volumeName}`|
|块设备|局部挂载点|BlockVolume|`GetPodDeviceMapPath()`|`pods/{podUid}/volumeDevices/{escapeQualifiedPluginName}/{volumeName}`|
|文件系统|全局挂载点|DeviceMounter|`GetDeviceMountPath()`|`plugins/kubernetes.io/{PluginName}/{DefaultKubeletVolumeDevicesDirName}/{volumePluginDependentPath}`|
|块设备|全局挂载点|BlockVolume|`GetGlobalMapPath()`|`plugins/kubernetes.io/{PluginName}/{DefaultKubeletVolumeDevicesDirName}/{volumePluginDependentPath}`|
