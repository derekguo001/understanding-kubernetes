# Volume Plugin Manager #

## 对象解析 ##

为了对所有的 Volume Plugin 进行统一的管理，在 Kubernetes 有一个名为 VolumePluginMgr 对象。

``` go
// VolumePluginMgr tracks registered plugins.
type VolumePluginMgr struct {
	mutex         sync.Mutex
	plugins       map[string]VolumePlugin
	prober        DynamicPluginProber
	probedPlugins map[string]VolumePlugin
	Host          VolumeHost
}
```

主要字段说明：

- `plugins` 字段用于保存所有静态载入的 volume plugin 对象，也就是说创建 Volume Plugin Manager 对象时会将对应的值作为一个参数传入，然后保存到这个字段中。
- `prober` 和 `probedPlugins` 用于保存 Flex 动态发现的 plugin 对象，目前推荐使用 CSI 来扩展 Kubernetes ，Flex 已不推荐使用，我们不会对其深入分析。
- `Host VolumeHost` 字段用于保存需要访问 Kubelet 资源的对象。

Volume Plugin Manager 主要包含以下一些函数：

- `InitPlugins()` 用于初始化操作，会调用每一个 plugin 的 `Init()` 函数对 plugin 分别执行初始化操作。
- `FindPluginBySpec()`、`FindPluginByName()` 等一系列 `Find Plugin` 类型的函数，这一步主要用于找到对应的 plugin 对象，接下来会调用 plugin 进行各种操作。

## 使用方式 ##

Volume Plugin Manager 在 Kubernetes 源代码中主要用于两个地方：

1. 各种 Volume 相关的 Volume Controller 中，用于根据 PV 和 PVC 对象来执行创建/删除底层的存储对象，以及执行 attach/detach 等操作。
2. Kubelet 中，用于执行 mount/umount 等操作。

Volume Plugin Manager 在各种 Controller 中和在 Kubelet 中的使用方式都是类似的，通常的使用方式如下：
1. 针对各种 in-tree 类型的 Volume Plugin，分别调用每个 Plugin 的 `ProbeVolumePlugins()`，这个函数会返回当前的 plugin 对象，例如下面是 local volume 对应的函数。
   ``` go
   func ProbeVolumePlugins() []volume.VolumePlugin {
       return []volume.VolumePlugin{&localVolumePlugin{}}
   }
   ```
2. 将上一步中所有的 plugin 搜集到一个列表中。
3. 创建 Volume Plugin Manager 对象。
4. Volume Plugin Manager 执行自身的 InitPlugins()，会将第 2 步中的 plugin 列表作为参数传入，然后 Volume Plugin Manager 会将这个列表的值存入自身的 `plugins` 字段，然后调用每一个 plugin 的 `Init()` 函数对每个 Plugin 进行初始化。
5. 以上是初始化的步骤，初始化完成后，会进行周期性的循环中，当检测到需要执行对应的操作时，会遍历所有的 volume 列表，针对每个 volume 使用 `FindPluginBySpec()` 等一系列 `Find Plugin` 函数找到当前 volume 对应的 plugin 对象。接下来就是使用这些具体的 plugin 对象执行各自的操作了，具体的操作依赖于 Volume Plugin Manager 的上下文环境，例如在各种 Volume Controller 中或者在 Kubelet 中。
