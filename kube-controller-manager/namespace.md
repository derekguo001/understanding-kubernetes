# Namespace Controller #

## 概述 ##

Namespace Controller 的主要作用是在用户删除一个 Namespace 的时候自动删除这个 Namespace 中的其它资源，然后再删除这个 Namespace 对象本身。

一个 Namespace 有两种状态：

- **Active**

  正常使用状态。新创建的 Namespace 即为 Active 状态。

- **Terminating**

  删除状态，此时不能再在其中创建其它资源对象。

Namespace 的属性里有两个字段与删除操作相关：

- **Spec.Finalizers**
- **ObjectMeta.DeletionTimestamp**

当对一个 Active 状态的 Namespace 执行 Delete 操作后，Namespace 的 ObjectMeta.DeletionTimestamp 会被设置为当时的服务器端的时间。Kubernetes Namespace Controller 会 watch 到已经设置了 ObjectMeta.DeletionTimestamp 属性的这个 Namespace，且对应的值不为空，它就会将这个 Namespace 的状态设置为 Terminating，然后对其中的资源逐个进行删除操作。

Kubernetes Namespace Controller 是如何知道这个 Namespace 中包含哪些对象呢？这个就与 Spec.Finalizers 字段有关系了，这个字段的值是一个列表。当新建一个 Namespace 对象时，Kubernetes 会自动将 **kubernetes** 这个 value 添加到 Spec.Finalizers 字段中，用于表示这个 Namespace 中的资源包含有 Kubernetes 资源，同时，在创建 Namespace Controller 的时候需要指定一个回调函数作为参数，这个回调函数的作用是在删除时找出当前 Namespace 下有哪些资源。在最终执行删除操作的时候会先将其中的这些子资源删除，删除了这些子资源之后，Kubernetes Namespace Controller会将 Spec.Finalizers 中的 **kubernetes** value 删除，这时，如果 Spec.Finalizers 中没有其它 value 的话，Namespace Controller 就会认为其中的所有子资源已经正确清除掉了，这时就可以删除 Namespace 本身了。

如果有另外的一个 Namespace Controller 为 Spec.Finalizers 添加了其它 value，那么这些 Controller 需要在执行删除 Namespace 对象的时候先删除由它定义的第三方的资源，然后清除 Namespace Controller 的 Spec.Finalizers 中对应的 value。只有当 Spec.Finalizers 的 value 为空，Kubernetes Namespace Controller 才会执行删除 Namespace 对象本身的操作。

## 参考 ##

- [Share a Cluster with Namespaces](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/)
- [Namespace Design Doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/namespaces.md#finalizers)
