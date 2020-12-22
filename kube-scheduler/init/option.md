# kubernetes Scheduler 启动过程中的参数初始化过程分析 #

## 概述 ##

kubernetes Scheduler 中所有的启动参数可参考 https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/ 。

参数大致可以分为两类：

- ***调度策略相关的参数***，例如启用那些调度插件，以及给某些调度插件配置一些参数
- ***通用参数***，这是作为一个普通的 Server 所包含的一些参数，例如端口号等等

我们这里重点关注第一种，即调度策略相关的参数，目前版本中推荐将这些参数配置到一个配置文件中，使用 `--config File` 来指定配置文件名称。

所有的配置都存储于 `Options` 结构体中。

``` go
type Options struct {
	// The default values. These are overridden if ConfigFile is set or by values in InsecureServing.
	ComponentConfig kubeschedulerconfig.KubeSchedulerConfiguration

	SecureServing           *apiserveroptions.SecureServingOptionsWithLoopback
	CombinedInsecureServing *CombinedInsecureServingOptions
	Authentication          *apiserveroptions.DelegatingAuthenticationOptions
	Authorization           *apiserveroptions.DelegatingAuthorizationOptions
	Deprecated              *DeprecatedOptions

	// ConfigFile is the location of the scheduler server's configuration file.
	ConfigFile string

	// WriteConfigTo is the path where the default configuration will be written.
	WriteConfigTo string

	Master string

	ShowHiddenMetricsForVersion string
}
```

第一个成员 `ComponentConfig kubeschedulerconfig.KubeSchedulerConfiguration` 即为我们提到的***调度策略相关的参数***。而其中的 `Deprecated *DeprecatedOptions` 是旧的调度策略相关的参数，它们已经标记为过期并且会在将来的版本中移除掉。剩下的其它参数可以认为是***通用参数***。
