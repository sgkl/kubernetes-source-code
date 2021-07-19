# Configurator
Configurator持有创建Scheduler的所有信息，用来创建Scheduler

## Configurator
### 数据结构
```go
// Configurator defines I/O, caching, and other functionality needed to
// construct a new scheduler.
type Configurator struct {
	client     clientset.Interface     // api server client
	kubeConfig *restclient.Config      // 用来创建api server client的配置

	recorderFactory profile.RecorderFactory

	informerFactory informers.SharedInformerFactory  // sharedInformerFactory

	// Close this to stop all reflectors
	StopEverything <-chan struct{}

	schedulerCache internalcache.Cache    // 缓存

	componentConfigVersion string     // 当前组件的版本信息，例如： kubescheduler.config.k8s.io/v1beta2

	// Always check all predicates even if the middle of one predicate fails.
	alwaysCheckAllPredicates bool    // 如果为true，即便某个predicate失败了，还要检查后续的predicates

	// percentageOfNodesToScore specifies percentage of all nodes to score in each scheduling cycle.
	percentageOfNodesToScore int32   // 所以node中需要打分的node的百分比，如果为0，则由调度器自行决定

	podInitialBackoffSeconds int64   // 第一次backoff的时间

	podMaxBackoffSeconds int64       // backoff的最大时间

	profiles          []schedulerapi.KubeSchedulerProfile
	registry          frameworkruntime.Registry   // 保存所有plugin的构造工厂函数
	nodeInfoSnapshot  *internalcache.Snapshot     // snapshot
	extenders         []schedulerapi.Extender     // 调度扩展
	frameworkCapturer FrameworkCapturer 
	parallellism      int32
	// A "cluster event" -> "plugin names" map.
	clusterEventMap map[framework.ClusterEvent]sets.String
}

// FrameworkCapturer is used for registering a notify function in building framework.
type FrameworkCapturer func(schedulerapi.KubeSchedulerProfile)
```

### 方法

#### create

`create() (*Scheduler, error)`创建Scheduler，这个是用新框架创建的。

```go
// create a scheduler from a set of registered plugins.
func (c *Configurator) create() (*Scheduler, error) {
	var extenders []framework.Extender
	var ignoredExtendedResources []string
	if len(c.extenders) != 0 {
		var ignorableExtenders []framework.Extender
		for ii := range c.extenders {
			klog.V(2).InfoS("Creating extender", "extender", c.extenders[ii])
			extender, err := NewHTTPExtender(&c.extenders[ii])
			if err != nil {
				return nil, err
			}
			if !extender.IsIgnorable() {
				extenders = append(extenders, extender)
			} else {
				ignorableExtenders = append(ignorableExtenders, extender)
			}
			for _, r := range c.extenders[ii].ManagedResources {
				if r.IgnoredByScheduler {
					ignoredExtendedResources = append(ignoredExtendedResources, r.Name)
				}
			}
		}
		// place ignorable extenders to the tail of extenders
		extenders = append(extenders, ignorableExtenders...)
	}

	// If there are any extended resources found from the Extenders, append them to the pluginConfig for each profile.
	// This should only have an effect on ComponentConfig, where it is possible to configure Extenders and
	// plugin args (and in which case the extender ignored resources take precedence).
	// For earlier versions, using both policy and custom plugin config is disallowed, so this should be the only
	// plugin config for this plugin.
  // 如果有可忽略的ignoredExtendedResources，则查找PluginConfig，将其添加到名为noderesources.FitName的PluginConfig中。
  // 后续做什么用还未知
	if len(ignoredExtendedResources) > 0 {
		for i := range c.profiles {
			prof := &c.profiles[i]
			var found = false
			for k := range prof.PluginConfig {
				if prof.PluginConfig[k].Name == noderesources.FitName {
					// Update the existing args
					pc := &prof.PluginConfig[k]
					args, ok := pc.Args.(*schedulerapi.NodeResourcesFitArgs)
					if !ok {
						return nil, fmt.Errorf("want args to be of type NodeResourcesFitArgs, got %T", pc.Args)
					}
					args.IgnoredResources = ignoredExtendedResources
					found = true
					break
				}
			}
			if !found {
				return nil, fmt.Errorf("can't find NodeResourcesFitArgs in plugin config")
			}
		}
	}

	// The nominator will be passed all the way to framework instantiation.
	nominator := internalqueue.NewPodNominator(c.informerFactory.Core().V1().Pods().Lister())
	profiles, err := profile.NewMap(c.profiles, c.registry, c.recorderFactory,
		frameworkruntime.WithComponentConfigVersion(c.componentConfigVersion),
		frameworkruntime.WithClientSet(c.client),
		frameworkruntime.WithKubeConfig(c.kubeConfig),
		frameworkruntime.WithInformerFactory(c.informerFactory),
		frameworkruntime.WithSnapshotSharedLister(c.nodeInfoSnapshot),
		frameworkruntime.WithRunAllFilters(c.alwaysCheckAllPredicates),
		frameworkruntime.WithPodNominator(nominator),
		frameworkruntime.WithCaptureProfile(frameworkruntime.CaptureProfile(c.frameworkCapturer)),
		frameworkruntime.WithClusterEventMap(c.clusterEventMap),
		frameworkruntime.WithParallelism(int(c.parallellism)),
		frameworkruntime.WithExtenders(extenders),
	)
	if err != nil {
		return nil, fmt.Errorf("initializing profiles: %v", err)
	}
	if len(profiles) == 0 {
		return nil, errors.New("at least one profile is required")
	}
	// Profiles are required to have equivalent queue sort plugins.
	lessFn := profiles[c.profiles[0].SchedulerName].QueueSortFunc()
	podQueue := internalqueue.NewSchedulingQueue(
		lessFn,
		c.informerFactory,
		internalqueue.WithPodInitialBackoffDuration(time.Duration(c.podInitialBackoffSeconds)*time.Second),
		internalqueue.WithPodMaxBackoffDuration(time.Duration(c.podMaxBackoffSeconds)*time.Second),
		internalqueue.WithPodNominator(nominator),
		internalqueue.WithClusterEventMap(c.clusterEventMap),
	)

	// Setup cache debugger.
	debugger := cachedebugger.New(
		c.informerFactory.Core().V1().Nodes().Lister(),
		c.informerFactory.Core().V1().Pods().Lister(),
		c.schedulerCache,
		podQueue,
	)
	debugger.ListenForSignal(c.StopEverything)

	algo := NewGenericScheduler(
		c.schedulerCache,
		c.nodeInfoSnapshot,
		c.percentageOfNodesToScore,
	)

	return &Scheduler{
		SchedulerCache:  c.schedulerCache,
		Algorithm:       algo,
		Extenders:       extenders,
		Profiles:        profiles,
		NextPod:         internalqueue.MakeNextPodFunc(podQueue),
		Error:           MakeDefaultErrorFunc(c.client, c.informerFactory.Core().V1().Pods().Lister(), podQueue, c.schedulerCache),
		StopEverything:  c.StopEverything,
		SchedulingQueue: podQueue,
	}, nil
}
```

#### createFromPolicy

createFromPolicy通过传统的policy文件构造Scheduler。这个函数存在的意义是为了兼容之前的版本。

```go
// createFromPolicy creates a scheduler from the legacy policy file.
func (c *Configurator) createFromPolicy(policy schedulerapi.Policy) (*Scheduler, error) {
	lr := frameworkplugins.NewLegacyRegistry()
	args := &frameworkplugins.ConfigProducerArgs{}

	klog.V(2).InfoS("Creating scheduler from configuration", "policy", policy)

	// validate the policy configuration
	if err := validation.ValidatePolicy(policy); err != nil {
		return nil, err
	}

	// If profiles is already set, it means the user is using both CC and policy config, error out
	// since these configs are no longer merged and they should not be used simultaneously.
	if c.profiles != nil {
		return nil, fmt.Errorf("profiles and policy config both set, this should not happen")
	}

	predicateKeys := sets.NewString()
	if policy.Predicates == nil {
		predicateKeys = lr.DefaultPredicates
	} else {
		for _, predicate := range policy.Predicates {
			klog.V(2).InfoS("Registering predicate", "predicate", predicate.Name)
			predicateName, err := lr.ProcessPredicatePolicy(predicate, args)
			if err != nil {
				return nil, err
			}
			predicateKeys.Insert(predicateName)
		}
	}

	priorityKeys := make(map[string]int64)
	if policy.Priorities == nil {
		klog.V(2).InfoS("Using default priorities")
		priorityKeys = lr.DefaultPriorities
	} else {
		for _, priority := range policy.Priorities {
			if priority.Name == frameworkplugins.EqualPriority {
				klog.V(2).InfoS("Skip registering priority", "priority", priority.Name)
				continue
			}
			klog.V(2).InfoS("Registering priority", "priority", priority.Name)
			priorityName, err := lr.ProcessPriorityPolicy(priority, args)
			if err != nil {
				return nil, err
			}
			priorityKeys[priorityName] = priority.Weight
		}
	}

	// HardPodAffinitySymmetricWeight in the policy config takes precedence over
	// CLI configuration.
	if policy.HardPodAffinitySymmetricWeight != 0 {
		args.InterPodAffinityArgs = &schedulerapi.InterPodAffinityArgs{
			HardPodAffinityWeight: policy.HardPodAffinitySymmetricWeight,
		}
	}

	// When AlwaysCheckAllPredicates is set to true, scheduler checks all the configured
	// predicates even after one or more of them fails.
	if policy.AlwaysCheckAllPredicates {
		c.alwaysCheckAllPredicates = policy.AlwaysCheckAllPredicates
	}

	klog.V(2).InfoS("Creating scheduler", "predicates", predicateKeys, "priorities", priorityKeys)

	// Combine all framework configurations. If this results in any duplication, framework
	// instantiation should fail.

	// "PrioritySort", "DefaultPreemption" and "DefaultBinder" were neither predicates nor priorities
	// before. We add them by default.
	plugins := schedulerapi.Plugins{
		QueueSort: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{{Name: queuesort.Name}},
		},
		PostFilter: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{{Name: defaultpreemption.Name}},
		},
		Bind: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{{Name: defaultbinder.Name}},
		},
	}
	var pluginConfig []schedulerapi.PluginConfig
	var err error
	if plugins, pluginConfig, err = lr.AppendPredicateConfigs(predicateKeys, args, plugins, pluginConfig); err != nil {
		return nil, err
	}
	if plugins, pluginConfig, err = lr.AppendPriorityConfigs(priorityKeys, args, plugins, pluginConfig); err != nil {
		return nil, err
	}
	if pluginConfig, err = dedupPluginConfigs(pluginConfig); err != nil {
		return nil, err
	}

	c.profiles = []schedulerapi.KubeSchedulerProfile{
		{
			SchedulerName: v1.DefaultSchedulerName,
			Plugins:       &plugins,
			PluginConfig:  pluginConfig,
		},
	}

	if err := defaultPluginConfigArgs(&c.profiles[0]); err != nil {
		return nil, err
	}

	return c.create()
}
```

