# ConfigMap

管理ConfigMap

## Manager

```go
// Manager interface provides methods for Kubelet to manage ConfigMap.
type Manager interface {
	// Get configmap by configmap namespace and name.
	GetConfigMap(namespace, name string) (*v1.ConfigMap, error)

	// WARNING: Register/UnregisterPod functions should be efficient,
	// i.e. should not block on network operations.

	// RegisterPod registers all configmaps from a given pod.
	RegisterPod(pod *v1.Pod)

	// UnregisterPod unregisters configmaps from a given pod that are not
	// used by any other registered pod.
	UnregisterPod(pod *v1.Pod)
}
```

## simpleConfigMapManager

simpleConfigMapManager实现了Manager接口，但内部不缓存configMap，每次get时都需要从API server获取configMap

```go
// simpleConfigMapManager implements ConfigMap Manager interface with
// simple operations to apiserver.
type simpleConfigMapManager struct {
	kubeClient clientset.Interface
}
```

### 构造函数

```go
// NewSimpleConfigMapManager creates a new ConfigMapManager instance.
func NewSimpleConfigMapManager(kubeClient clientset.Interface) Manager {
	return &simpleConfigMapManager{kubeClient: kubeClient}
}
```

### 方法

* GetConfigMap

  获取给定namespace和name的configMap

  ```go
  func (s *simpleConfigMapManager) GetConfigMap(namespace, name string) (*v1.ConfigMap, error) {
  	return s.kubeClient.CoreV1().ConfigMaps(namespace).Get(context.TODO(), name, metav1.GetOptions{})
  }
  ```

* RegisterPod

  ```go
  func (s *simpleConfigMapManager) RegisterPod(pod *v1.Pod) {
  }
  ```

  

* UnregisterPod

  ```go
  func (s *simpleConfigMapManager) UnregisterPod(pod *v1.Pod) {
  }
  ```

## configMapManager

configMapManager实现了Manager接口，内部缓存ConfigMap对象

```go
// configMapManager keeps a cache of all configmaps necessary
// for registered pods. Different implementation of the store
// may result in different semantics for freshness of configmaps
// (e.g. ttl-based implementation vs watch-based implementation).
type configMapManager struct {
	manager manager.Manager
}
```

### 方法

* GetConfigMap

  从缓存中获取给定namespace和name的ConfigMap

  ```go
  func (c *configMapManager) GetConfigMap(namespace, name string) (*v1.ConfigMap, error) {
  	object, err := c.manager.GetObject(namespace, name)
  	if err != nil {
  		return nil, err
  	}
  	if configmap, ok := object.(*v1.ConfigMap); ok {
  		return configmap, nil
  	}
  	return nil, fmt.Errorf("unexpected object type: %v", object)
  }
  ```

* RegisterPod

  注册需要监控ConfigMap的pod

  ```go
  func (c *configMapManager) RegisterPod(pod *v1.Pod) {
  	c.manager.RegisterPod(pod)
  }
  ```

* UnregisterPod

  解注册需要监控ConfigMap的pod

### 构造函数

#### NewCachingConfigMapManager

```go
// NewCachingConfigMapManager creates a manager that keeps a cache of all configmaps
// necessary for registered pods.
// It implement the following logic:
// - whenever a pod is create or updated, the cached versions of all configmaps
//   are invalidated
// - every GetObject() call tries to fetch the value from local cache; if it is
//   not there, invalidated or too old, we fetch it from apiserver and refresh the
//   value in cache; otherwise it is just fetched from cache
func NewCachingConfigMapManager(kubeClient clientset.Interface, getTTL manager.GetObjectTTLFunc) Manager {
	getConfigMap := func(namespace, name string, opts metav1.GetOptions) (runtime.Object, error) {
		return kubeClient.CoreV1().ConfigMaps(namespace).Get(context.TODO(), name, opts)
	}
	configMapStore := manager.NewObjectStore(getConfigMap, clock.RealClock{}, getTTL, defaultTTL)
	return &configMapManager{
		manager: manager.NewCacheBasedManager(configMapStore, getConfigMapNames),
	}
}
```

#### NewWatchingConfigMapManager

```go
// NewWatchingConfigMapManager creates a manager that keeps a cache of all configmaps
// necessary for registered pods.
// It implements the following logic:
// - whenever a pod is created or updated, we start individual watches for all
//   referenced objects that aren't referenced from other registered pods
// - every GetObject() returns a value from local cache propagated via watches
func NewWatchingConfigMapManager(kubeClient clientset.Interface, resyncInterval time.Duration) Manager {
	listConfigMap := func(namespace string, opts metav1.ListOptions) (runtime.Object, error) {
		return kubeClient.CoreV1().ConfigMaps(namespace).List(context.TODO(), opts)
	}
	watchConfigMap := func(namespace string, opts metav1.ListOptions) (watch.Interface, error) {
		return kubeClient.CoreV1().ConfigMaps(namespace).Watch(context.TODO(), opts)
	}
	newConfigMap := func() runtime.Object {
		return &v1.ConfigMap{}
	}
	isImmutable := func(object runtime.Object) bool {
		if configMap, ok := object.(*v1.ConfigMap); ok {
			return configMap.Immutable != nil && *configMap.Immutable
		}
		return false
	}
	gr := corev1.Resource("configmap")
	return &configMapManager{
		manager: manager.NewWatchBasedManager(listConfigMap, watchConfigMap, newConfigMap, isImmutable, gr, resyncInterval, getConfigMapNames),
	}
}
```



