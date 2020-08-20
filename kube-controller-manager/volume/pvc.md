# PVC Controller #

PVC Controller 的代码位于 `pkg/controller/volume/persistentvolume/` 下面的 `pv_controller_base.go` 和 `pv_controller.go` 中。

下面是处理 PVC 对象的函数调用栈

``` go
updateClaim()
    syncClaim()
        syncUnboundClaim()
        syncBoundClaim()
```

根据当前 PVC 对象是否与 PV 绑定，分别进行处理。判断是否绑定的依据是根据 PVC 对象是否包含 `pv.kubernetes.io/bind-completed` 这个 Annotation，注意对应的值是什么没有关系。如果已绑定，则使用 `syncUnboundClaim()` 处理，否则使用 `syncBoundClaim()` 处理。

## 处理未与 PV 绑定的 PVC ##

`syncUnboundClaim()` 处理和 PV 之间没有绑定关系的 PVC 对象。用于新建的 PVC，这时 PVC 为 Pending 状态，又细分为两类：

- **没有明确指定需要绑定 PV 的 PVC**。这时会尝试查找一个已创建的 PV 并将其绑定，如果找不到，则会尝试使用 PVC 中的 StorageClass 来动态创建一个 PV 然后与其绑定。另外对于延迟绑定的 PVC 在这个地方也不会做任何操作。

  其中，查找已存在的 PV 的函数为 `findByClaim()`，找到后进行绑定的函数为 `bind()`。

- **有明确指定需要绑定 PV 的 PVC**。会尝试查找指定的 PV，如果没找到，则还将 PVC 设置为 Pending 状态，以便下一次处理；如果找到了对应的 PV 且 PV 可以与当前 PVC 进行绑定，则尝试进行绑定。如果找到的 PV 已经与当前 PVC 绑定(PV 状态为 BOUND，PVC 状态为 Pending)，则仍然执行绑定操作，用于为 PV 设置 `volume.Spec.ClaimRef` 字段的值。如果找到的 PV 已经和其它 PVC 绑定，则还将 PVC 设置为 Pending 状态，在下一次进行处理。


## 处理已经与 PV 绑定的 PVC ##

`syncBoundClaim()` 处理和 PV 之间已经有了绑定关系的 PVC 对象。分为几种情况：

- PVC 绑定的 PV 字段为空(`claim.Spec.VolumeName` 为空)，将 PVC 设置为 Lost 状态，将对应的 PV 字段置空。
- 找不到 PVC 绑定的 PV 对象，将 PVC 设置为 Lost 状态。
- PVC 绑定的 PV 对象的 `volume.Spec.ClaimRef` 为空，表明此前是绑定状态，此时重新进行绑定。
- PVC 绑定的 PV 对象的 `volume.Spec.ClaimRef` 为当前 PVC，此时重新进行绑定，正常状况下具体执行绑定操作时什么都不会做。
- PVC 绑定的 PV 对象的 `volume.Spec.ClaimRef` 是其它 PVC，表明同一个 PV 对象与两个 PVC 发生了绑定操作，即发生了错误，这时将当前 PVC 设置为 Lost 状态，将对应的 PV 字段置空。
