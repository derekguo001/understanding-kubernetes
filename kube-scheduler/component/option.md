# Kubernetes Scheduler 启动过程中的参数初始化过程分析 #

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

## 启动过程分析 ##

下面详细分析启动过程中这些参数是如何解析的，与 kubernetes Scheduler 提供的一些默认配置是如何交互和融合的。

``` go
func NewSchedulerCommand(registryOptions ...Option) *cobra.Command {
	opts, err := options.NewOptions()
	if err != nil {
		klog.Fatalf("unable to initialize command options: %v", err)
	}
```

启动后，先创建 Options 对象。

``` go
func NewOptions() (*Options, error) {
	cfg, err := newDefaultComponentConfig()
	if err != nil {
		return nil, err
	}

	hhost, hport, err := splitHostIntPort(cfg.HealthzBindAddress)
	if err != nil {
		return nil, err
	}

	o := &Options{
		ComponentConfig: *cfg,
		SecureServing:   apiserveroptions.NewSecureServingOptions().WithLoopback(),
        ...
		Deprecated: &DeprecatedOptions{
			UseLegacyPolicyConfig:          false,
			PolicyConfigMapNamespace:       metav1.NamespaceSystem,
			SchedulerName:                  corev1.DefaultSchedulerName,
			HardPodAffinitySymmetricWeight: interpodaffinity.DefaultHardPodAffinityWeight,
		},
	}
    ...

	return o, nil
}
```

这里也默认创建了 KubeSchedulerConfiguration 对象，即***调度策略相关的参数***，另外还创建了默认的 DeprecatedOptions 对象，其它的都是通用参数对象。

``` go
	fs := cmd.Flags()
	namedFlagSets := opts.Flags()
	verflag.AddFlags(namedFlagSets.FlagSet("global"))
	globalflag.AddGlobalFlags(namedFlagSets.FlagSet("global"), cmd.Name())
	for _, f := range namedFlagSets.FlagSets {
		fs.AddFlagSet(f)
	}
    ...
	return cmd
}
```

创建完 Options 对象后，会为其添加命令行参数，代码中的 `opts.Flags()`。


``` go
func (o *Options) Flags() (nfs cliflag.NamedFlagSets) {
	fs := nfs.FlagSet("misc")
	fs.StringVar(&o.ConfigFile, "config", o.ConfigFile, "The path to the configuration file. Flags override values in this file.")
	fs.StringVar(&o.WriteConfigTo, "write-config-to", o.WriteConfigTo, "If set, write the configuration values to this file and exit.")
	fs.StringVar(&o.Master, "master", o.Master, "The address of the Kubernetes API server (overrides any value in kubeconfig)")

	o.SecureServing.AddFlags(nfs.FlagSet("secure serving"))
	o.CombinedInsecureServing.AddFlags(nfs.FlagSet("insecure serving"))
	o.Authentication.AddFlags(nfs.FlagSet("authentication"))
	o.Authorization.AddFlags(nfs.FlagSet("authorization"))
	o.Deprecated.AddFlags(nfs.FlagSet("deprecated"), &o.ComponentConfig)

    ...
	return nfs
}
```

其中非常重要的是 `--config`，我们会通过这个参数指定配置文件。其它通用参数分别通过具体的标准对象来添加。

``` go
	cmd := &cobra.Command{
		Use: "kube-scheduler",
        ...
		Run: func(cmd *cobra.Command, args []string) {
			if err := runCommand(cmd, args, opts, registryOptions...); err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
		},
	}
```

创建完 Options 和 Kubernetes Scheduler 的 cmd 对象后，会通过 `runCommand()` 来启动，同时把 Options 作为第三个参数传入。

``` go
func runCommand(cmd *cobra.Command, args []string, opts *options.Options, registryOptions ...Option) error {
	verflag.PrintAndExitIfRequested()
	utilflag.PrintFlags(cmd.Flags())
```

在 `runCommand()` 中首先 dump 所有的参数的解析结果。


``` go
	if errs := opts.Validate(); len(errs) > 0 {
		return utilerrors.NewAggregate(errs)
	}

	if len(opts.WriteConfigTo) > 0 {
		c := &schedulerserverconfig.Config{}
		if err := opts.ApplyTo(c); err != nil {
			return err
		}
		if err := options.WriteConfigFile(opts.WriteConfigTo, &c.ComponentConfig); err != nil {
			return err
		}
		klog.Infof("Wrote configuration to: %s\n", opts.WriteConfigTo)
		return nil
	}
```

然后使用 `opts.Validate()` 对所有参数进行有效性验证，接着判断如果需要将配置写入文件，则执行写入操作并退出。

``` go
	c, err := opts.Config()
	if err != nil {
		return err
	}

	// Get the completed config
	cc := c.Complete()
```

接着使用 `opts.Config()` 从 Options 对象创建 schedulerappconfig.Config 对象。

``` go
func (o *Options) Config() (*schedulerappconfig.Config, error) {
    ...
	c := &schedulerappconfig.Config{}
	if err := o.ApplyTo(c); err != nil {
		return nil, err
	}
    ...

	c.Client = client
	c.InformerFactory = informers.NewSharedInformerFactory(client, 0)
	c.PodInformer = scheduler.NewPodInformer(client, 0)
    ...
	return c, nil
}
```

其中使用 `o.ApplyTo(c)` 执行 Options 对象到 schedulerappconfig.Config 对象的转换。

``` go
func (o *Options) ApplyTo(c *schedulerappconfig.Config) error {
	if len(o.ConfigFile) == 0 {
		c.ComponentConfig = o.ComponentConfig
        ...
		if err := o.Deprecated.ApplyTo(&c.ComponentConfig); err != nil {
			return err
		}
		if err := o.CombinedInsecureServing.ApplyTo(c, &c.ComponentConfig); err != nil {
			return err
		}
	} else {
		cfg, err := loadConfigFromFile(o.ConfigFile)
		if err != nil {
			return err
		}
		if err := validation.ValidateKubeSchedulerConfiguration(cfg).ToAggregate(); err != nil {
			return err
		}
        ...
		c.ComponentConfig = *cfg
        ...
	}

    ...
	return nil
}

```

在 `ApplyTo()` 中，如果设置了 `--config` 参数，则会解析对应的配置文件并验证其有效性，如果一切正常，则会将其存入 schedulerappconfig.Config 中的 ComponentConfig 字段。这个字段保存了前面提到的***调度策略相关的参数***。注意，只有没有设置 `--config` 参数的情况下才会使用旧的参数，即 `Deprecated`，因此如果同时在命令行进行了设置，则后面这些旧的参数是***不生效***的。

``` go
    ...
	return Run(ctx, cc, registryOptions...)
}
```

一切准备就绪后，将封装了 schedulerappconfig.Config 对象的 CompletedConfig 对象作为第二个参数传递给 `Run()` 进行执行，在其中会创建真正的 Scheduler 对象并将其运行起来。

``` go
func Run(ctx context.Context, cc schedulerserverconfig.CompletedConfig, outOfTreeRegistryOptions ...Option) error {
	klog.V(1).Infof("Starting Kubernetes Scheduler version %+v", version.Get())
    ...

	sched, err := scheduler.New(cc.Client,
		cc.InformerFactory,
		cc.PodInformer,
		recorderFactory,
		ctx.Done(),
		scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
		scheduler.WithAlgorithmSource(cc.ComponentConfig.AlgorithmSource),
		scheduler.WithPreemptionDisabled(cc.ComponentConfig.DisablePreemption),
		scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
		scheduler.WithBindTimeoutSeconds(cc.ComponentConfig.BindTimeoutSeconds),
		scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
		scheduler.WithPodMaxBackoffSeconds(cc.ComponentConfig.PodMaxBackoffSeconds),
		scheduler.WithPodInitialBackoffSeconds(cc.ComponentConfig.PodInitialBackoffSeconds),
		scheduler.WithExtenders(cc.ComponentConfig.Extenders...),
	)

    ...
	sched.Run(ctx)
	return fmt.Errorf("finished without leader elect")
}
```

下面是 `scheduler.New()` 的代码片段。

``` go
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	podInformer coreinformers.PodInformer,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {

    ...
	options := defaultSchedulerOptions
	for _, opt := range opts {
		opt(&options)
	}
```

这里需要注意的是调用 `scheduler.New()` 时使用 的 `With...` 等相关的参数，以 `scheduler.WithProfiles()` 为例。

``` go
func WithProfiles(p ...schedulerapi.KubeSchedulerProfile) Option {
	return func(o *schedulerOptions) {
		o.profiles = p
	}
}
```

这些其实都是一个个的回调函数，这些回调函数会作为可变参数 `opts ...Option` 传入。

在 `scheduler.New()` 内部执行时，会首先使用 `options := defaultSchedulerOptions` 来设置一些默认值。

``` go
var defaultSchedulerOptions = schedulerOptions{
	profiles: []schedulerapi.KubeSchedulerProfile{
		// Profiles' default plugins are set from the algorithm provider.
		{SchedulerName: v1.DefaultSchedulerName},
	},
	schedulerAlgorithmSource: schedulerapi.SchedulerAlgorithmSource{
		Provider: defaultAlgorithmSourceProviderName(),
	},
	disablePreemption:        false,
	percentageOfNodesToScore: schedulerapi.DefaultPercentageOfNodesToScore,
	bindTimeoutSeconds:       BindTimeoutSeconds,
	podInitialBackoffSeconds: int64(internalqueue.DefaultPodInitialBackoffDuration.Seconds()),
	podMaxBackoffSeconds:     int64(internalqueue.DefaultPodMaxBackoffDuration.Seconds()),
}
```

然后使用可变参数 `opts ...Option` 作为回调函数来覆盖这些默认值，或者填充其中的其它相关字段。

因此，调用时通过将一系列的 `With...` 回调函数作为可变参数传入，在 `scheduler.New()` 内部执行时使用 cc.ComponentConfig 中的不同字段来覆盖一些默认值，或者填充其中的其它相关字段。这样便将用户通过命令行传递的所有选项对 Kubernetes Scheduler 自身的默认的值进行了覆盖和融合。`scheduler.New()` 内定义的 `options` 变量即为融合后的***调度策略相关的参数***。
