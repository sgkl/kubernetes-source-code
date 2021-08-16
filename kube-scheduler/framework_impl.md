# Framework实现

`pkg/scheduler/framework/runtime/framework.go`中的frameworkImpl实现了Framework接口。

## frameworkImpl

### 数据结构

```go
// frameworkImpl is the component responsible for initializing and running scheduler
// plugins.
type frameworkImpl struct {
   registry             Registry   // 保存所有plugin的构造函数
   snapshotSharedLister framework.SharedLister  // 通过snapshot获取node信息
   waitingPods          *waitingPodsMap
   scorePluginWeight    map[string]int
   queueSortPlugins     []framework.QueueSortPlugin
   preFilterPlugins     []framework.PreFilterPlugin
   filterPlugins        []framework.FilterPlugin
   postFilterPlugins    []framework.PostFilterPlugin
   preScorePlugins      []framework.PreScorePlugin
   scorePlugins         []framework.ScorePlugin
   reservePlugins       []framework.ReservePlugin
   preBindPlugins       []framework.PreBindPlugin
   bindPlugins          []framework.BindPlugin
   postBindPlugins      []framework.PostBindPlugin
   permitPlugins        []framework.PermitPlugin

   clientSet       clientset.Interface
   kubeConfig      *restclient.Config
   eventRecorder   events.EventRecorder   // 记录事件
   informerFactory informers.SharedInformerFactory  // informers

   metricsRecorder *metricsRecorder
   profileName     string

   extenders []framework.Extender  // 扩展
   framework.PodNominator

   parallelizer parallelize.Parallelizer

   // Indicates that RunFilterPlugins should accumulate all failed statuses and not return
   // after the first failure.
   runAllFilters bool
}

// NodeInfoLister interface represents anything that can list/get NodeInfo objects from node name.
type NodeInfoLister interface {
	// Returns the list of NodeInfos.
	List() ([]*NodeInfo, error)
	// Returns the list of NodeInfos of nodes with pods with affinity terms.
	HavePodsWithAffinityList() ([]*NodeInfo, error)
	// Returns the list of NodeInfos of nodes with pods with required anti-affinity terms.
	HavePodsWithRequiredAntiAffinityList() ([]*NodeInfo, error)
	// Returns the NodeInfo of the given node name.
	Get(nodeName string) (*NodeInfo, error)
}

// SharedLister groups scheduler-specific listers.
type SharedLister interface {
	NodeInfos() NodeInfoLister
}
```

### 构造frameworkImpl

构造frameworkImpl时通过创建frameworkOptions传入所有参数

#### frameworkOptions

```go
type frameworkOptions struct {
  // 例如：kubescheduler.config.k8s.io/v1beta2
	componentConfigVersion string 
	clientSet              clientset.Interface
	kubeConfig             *restclient.Config
	eventRecorder          events.EventRecorder
	informerFactory        informers.SharedInformerFactory
	snapshotSharedLister   framework.SharedLister
	metricsRecorder        *metricsRecorder
	podNominator           framework.PodNominator
	extenders              []framework.Extender
	runAllFilters          bool
	captureProfile         CaptureProfile
	clusterEventMap        map[framework.ClusterEvent]sets.String
	parallelizer           parallelize.Parallelizer
}

// CaptureProfile is a callback to capture a finalized profile.
type CaptureProfile func(config.KubeSchedulerProfile)
```

##### 设置相关值的函数

* `func WithComponentConfigVersion(componentConfigVersion string) Option` 设置componentConfigVersion
* `func WithClientSet(clientSet clients.Interface) Option` 设置clientSet
* `func WithKubeConfig(KubeConfig *restclient.Config) Option`设置kubeConfig
* `func WithEventRecorder(recorder events.EventRecorder) Option` 设置eventRecorder
* `func WithInformerFactory(informerFactory informers.SharedInformerFactory) Option` 设置informerFactory
* `func WithSnapshotSharedLister(snapshotSharedLister framework.SharedLister) Option` 设置snapshotSharedLister
* `func WithRunAllFilters(runAllFilters bool) Option` 设置runAllFilters
* `func WithPodNominator(nominator framework.PodNominator) Option` 设置PodNominator
* `func WithExtenders(extenders []framework.Extender) Option` 设置extenders
* `func WithParallelism(parallelism int) Option`设置parallelism
* `func WithCaptureProfile(c CaptureProfile) Option`设置captureProfile
* `func WithClusterEventMap(m map[framework.ClusterEvent]sets.String) Option`设置clusterEventMap

##### 创建默认option

```go
func defaultFrameworkOptions() frameworkOptions {
	return frameworkOptions{
		metricsRecorder: newMetricsRecorder(1000, time.Second),
		clusterEventMap: make(map[framework.ClusterEvent]sets.String),
		parallelizer:    parallelize.NewParallelizer(parallelize.DefaultParallelism),
	}
}
```

#### KubeSchedulerProfile

构造frameworkImple时需要传入KubeSchedulerProfile作为参数

```go
// KubeSchedulerProfile is a scheduling profile.
type KubeSchedulerProfile struct {
	// SchedulerName is the name of the scheduler associated to this profile.
	// If SchedulerName matches with the pod's "spec.schedulerName", then the pod
	// is scheduled with this profile.
	SchedulerName string

	// Plugins specify the set of plugins that should be enabled or disabled.
	// Enabled plugins are the ones that should be enabled in addition to the
	// default plugins. Disabled plugins are any of the default plugins that
	// should be disabled.
	// When no enabled or disabled plugin is specified for an extension point,
	// default plugins for that extension point will be used if there is any.
	// If a QueueSort plugin is specified, the same QueueSort Plugin and
	// PluginConfig must be specified for all profiles.
	Plugins *Plugins

	// PluginConfig is an optional set of custom plugin arguments for each plugin.
	// Omitting config args for a plugin is equivalent to using the default config
	// for that plugin.
	PluginConfig []PluginConfig
}

// Plugins include multiple extension points. When specified, the list of plugins for
// a particular extension point are the only ones enabled. If an extension point is
// omitted from the config, then the default set of plugins is used for that extension point.
// Enabled plugins are called in the order specified here, after default plugins. If they need to
// be invoked before default plugins, default plugins must be disabled and re-enabled here in desired order.
type Plugins struct {
	// QueueSort is a list of plugins that should be invoked when sorting pods in the scheduling queue.
	QueueSort PluginSet

	// PreFilter is a list of plugins that should be invoked at "PreFilter" extension point of the scheduling framework.
	PreFilter PluginSet

	// Filter is a list of plugins that should be invoked when filtering out nodes that cannot run the Pod.
	Filter PluginSet

	// PostFilter is a list of plugins that are invoked after filtering phase, no matter whether filtering succeeds or not.
	PostFilter PluginSet

	// PreScore is a list of plugins that are invoked before scoring.
	PreScore PluginSet

	// Score is a list of plugins that should be invoked when ranking nodes that have passed the filtering phase.
	Score PluginSet

	// Reserve is a list of plugins invoked when reserving/unreserving resources
	// after a node is assigned to run the pod.
	Reserve PluginSet

	// Permit is a list of plugins that control binding of a Pod. These plugins can prevent or delay binding of a Pod.
	Permit PluginSet

	// PreBind is a list of plugins that should be invoked before a pod is bound.
	PreBind PluginSet

	// Bind is a list of plugins that should be invoked at "Bind" extension point of the scheduling framework.
	// The scheduler call these plugins in order. Scheduler skips the rest of these plugins as soon as one returns success.
	Bind PluginSet

	// PostBind is a list of plugins that should be invoked after a pod is successfully bound.
	PostBind PluginSet
}

// PluginSet specifies enabled and disabled plugins for an extension point.
// If an array is empty, missing, or nil, default plugins at that extension point will be used.
type PluginSet struct {
	// Enabled specifies plugins that should be enabled in addition to default plugins.
	// These are called after default plugins and in the same order specified here.
	Enabled []Plugin
	// Disabled specifies default plugins that should be disabled.
	// When all default plugins need to be disabled, an array containing only one "*" should be provided.
	Disabled []Plugin
}

// Plugin specifies a plugin name and its weight when applicable. Weight is used only for Score plugins.
type Plugin struct {
	// Name defines the name of plugin
	Name string
	// Weight defines the weight of plugin, only used for Score plugins.
	Weight int32
}
```

#### 构造函数

```go
// NewFramework initializes plugins given the configuration and the registry.
func NewFramework(r Registry, profile *config.KubeSchedulerProfile, opts ...Option) (framework.Framework, error) {
	options := defaultFrameworkOptions()  // 调用默认的构造函数创建options
	for _, opt := range opts {            // 通过传入的参数设置option中的值
		opt(&options)
	}

	f := &frameworkImpl{                  // 根据options创建framworkImpl
		registry:             r,
		snapshotSharedLister: options.snapshotSharedLister,
		scorePluginWeight:    make(map[string]int),
		waitingPods:          newWaitingPodsMap(),
		clientSet:            options.clientSet,
		kubeConfig:           options.kubeConfig,
		eventRecorder:        options.eventRecorder,
		informerFactory:      options.informerFactory,
		metricsRecorder:      options.metricsRecorder,
		runAllFilters:        options.runAllFilters,
		extenders:            options.extenders,
		PodNominator:         options.podNominator,
		parallelizer:         options.parallelizer,
	}

	if profile == nil {
		return f, nil
	}

	f.profileName = profile.SchedulerName
	if profile.Plugins == nil {
		return f, nil
	}

	var totalPriority int64
	for _, e := range profile.Plugins.Score.Enabled {
		// a weight of zero is not permitted, plugins can be disabled explicitly
		// when configured.
		f.scorePluginWeight[e.Name] = int(e.Weight)
		if f.scorePluginWeight[e.Name] == 0 {
			f.scorePluginWeight[e.Name] = 1
		}

		// Checks totalPriority against MaxTotalScore to avoid overflow
		if int64(f.scorePluginWeight[e.Name])*framework.MaxNodeScore > framework.MaxTotalScore-totalPriority {
			return nil, fmt.Errorf("total score of Score plugins could overflow")
		}
		totalPriority += int64(f.scorePluginWeight[e.Name]) * framework.MaxNodeScore
	}

	// get needed plugins from config
	pg := f.pluginsNeeded(profile.Plugins)

	pluginConfig := make(map[string]runtime.Object, len(profile.PluginConfig))
	for i := range profile.PluginConfig {
		name := profile.PluginConfig[i].Name
		if _, ok := pluginConfig[name]; ok {
			return nil, fmt.Errorf("repeated config for plugin %s", name)
		}
		pluginConfig[name] = profile.PluginConfig[i].Args
	}
	outputProfile := config.KubeSchedulerProfile{
		SchedulerName: f.profileName,
		Plugins:       profile.Plugins,
		PluginConfig:  make([]config.PluginConfig, 0, len(pg)),
	}

	pluginsMap := make(map[string]framework.Plugin)
	for name, factory := range r {
		// initialize only needed plugins.
		if _, ok := pg[name]; !ok {
			continue
		}

		args := pluginConfig[name]
		if args != nil {
			outputProfile.PluginConfig = append(outputProfile.PluginConfig, config.PluginConfig{
				Name: name,
				Args: args,
			})
		}
		p, err := factory(args, f)
		if err != nil {
			return nil, fmt.Errorf("initializing plugin %q: %w", name, err)
		}
		pluginsMap[name] = p

		// Update ClusterEventMap in place.
		fillEventToPluginMap(p, options.clusterEventMap)
	}

	for _, e := range f.getExtensionPoints(profile.Plugins) {
		if err := updatePluginList(e.slicePtr, *e.plugins, pluginsMap); err != nil {
			return nil, err
		}
	}

	// Verifying the score weights again since Plugin.Name() could return a different
	// value from the one used in the configuration.
	for _, scorePlugin := range f.scorePlugins {
		if f.scorePluginWeight[scorePlugin.Name()] == 0 {
			return nil, fmt.Errorf("score plugin %q is not configured with weight", scorePlugin.Name())
		}
	}

	if len(f.queueSortPlugins) == 0 {
		return nil, fmt.Errorf("no queue sort plugin is enabled")
	}
	if len(f.queueSortPlugins) > 1 {
		return nil, fmt.Errorf("only one queue sort plugin can be enabled")
	}
	if len(f.bindPlugins) == 0 {
		return nil, fmt.Errorf("at least one bind plugin is needed")
	}

	if options.captureProfile != nil {
		if len(outputProfile.PluginConfig) != 0 {
			sort.Slice(outputProfile.PluginConfig, func(i, j int) bool {
				return outputProfile.PluginConfig[i].Name < outputProfile.PluginConfig[j].Name
			})
		} else {
			outputProfile.PluginConfig = nil
		}
		options.captureProfile(outputProfile)
	}

	return f, nil
}
```

