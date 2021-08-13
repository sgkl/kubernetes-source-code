# secret

管理kubernetes secrets

## Manager

```go
// Manager manages Kubernetes secrets. This includes retrieving
// secrets or registering/unregistering them via Pods.
type Manager interface {
	// Get secret by secret namespace and name.
	GetSecret(namespace, name string) (*v1.Secret, error)

	// WARNING: Register/UnregisterPod functions should be efficient,
	// i.e. should not block on network operations.

	// RegisterPod registers all secrets from a given pod.
	RegisterPod(pod *v1.Pod)

	// UnregisterPod unregisters secrets from a given pod that are not
	// used by any other registered pod.
	UnregisterPod(pod *v1.Pod)
}
```

## simpleSecretManager

simpleSecretManager实现了Manager接口，内部不缓存secret

```go
// simpleSecretManager implements SecretManager interfaces with
// simple operations to apiserver.
type simpleSecretManager struct {
	kubeClient clientset.Interface
}
```

### 构造函数

```go
// NewSimpleSecretManager creates a new SecretManager instance.
func NewSimpleSecretManager(kubeClient clientset.Interface) Manager {
	return &simpleSecretManager{kubeClient: kubeClient}
}
```

### 方法

* GetSecret

  ```go
  func (s *simpleSecretManager) GetSecret(namespace, name string) (*v1.Secret, error) {
  	return s.kubeClient.CoreV1().Secrets(namespace).Get(context.TODO(), name, metav1.GetOptions{})
  }
  ```

* RegisterPod

  因内部无缓存，所以RegisterPod内部不进行任何操作

  ```go
  func (s *simpleSecretManager) RegisterPod(pod *v1.Pod) {
  }
  ```

  

* UnregisterPod

  因内部无缓存，所以RegisterPod内部不进行任何操作

  ```go
  func (s *simpleSecretManager) UnregisterPod(pod *v1.Pod) {
  }
  ```

  

## secretManager

secretManager实现了manager接口，内部缓存secret。通过manager.Manager接口实现。

```go
// secretManager keeps a store with secrets necessary
// for registered pods. Different implementations of the store
// may result in different semantics for freshness of secrets
// (e.g. ttl-based implementation vs watch-based implementation).
type secretManager struct {
	manager manager.Manager
}
```

### 方法

* getSecret

  `func (s *secretManager) GetSecret(namespace, name string) (*v1.Secret, error)`

  给定namespace和name，获取secret

* RegisterPod

  `func (s *secretManager) RegisterPod(pod *v1.Pod)`

  注册pod

* UnregisterPod

  `func (s *secretManager) UnregisterPod(pod *v1.Pod)`

  解注册pod

### 构造函数

#### NewCachingSecretManager

It implements the following logic:

- whenever a pod is created or updated, the cached versions of all secrets are invalidated
- every GetObject() call tries to fetch the value from local cache; if it is not there, invalidated or too old, we fetch it from apiserver and refresh the value in cache; otherwise it is just fetched from cache

创建一个SecretManager，此SecretManager有如下特征:

1. 如果一个pod被创建或被更新的时候，所以缓存的secrets都失效
2. `GetObject()`的每一次调用都会尝试从本地缓存中获取值，如果本地缓存中没有这个值则失效较旧的值，从API server获取并更新本地缓存

```go
// NewCachingSecretManager creates a manager that keeps a cache of all secrets
// necessary for registered pods.
// It implements the following logic:
// - whenever a pod is created or updated, the cached versions of all secrets
//   are invalidated
// - every GetObject() call tries to fetch the value from local cache; if it is
//   not there, invalidated or too old, we fetch it from apiserver and refresh the
//   value in cache; otherwise it is just fetched from cache
func NewCachingSecretManager(kubeClient clientset.Interface, getTTL manager.GetObjectTTLFunc) Manager {
	getSecret := func(namespace, name string, opts metav1.GetOptions) (runtime.Object, error) {
		return kubeClient.CoreV1().Secrets(namespace).Get(context.TODO(), name, opts)
	}
	secretStore := manager.NewObjectStore(getSecret, clock.RealClock{}, getTTL, defaultTTL)
	return &secretManager{
		manager: manager.NewCacheBasedManager(secretStore, getSecretNames),
	}
}
```

####  NewWatchingSecretManager

It implements the following logic:
- whenever a pod is created or updated, we start individual watches for all referenced objects that aren't referenced from other registered pods
- every GetObject() returns a value from local cache propagated via watches

```go
// NewWatchingSecretManager creates a manager that keeps a cache of all secrets
// necessary for registered pods.
// It implements the following logic:
// - whenever a pod is created or updated, we start individual watches for all
//   referenced objects that aren't referenced from other registered pods
// - every GetObject() returns a value from local cache propagated via watches
func NewWatchingSecretManager(kubeClient clientset.Interface, resyncInterval time.Duration) Manager {
	listSecret := func(namespace string, opts metav1.ListOptions) (runtime.Object, error) {
		return kubeClient.CoreV1().Secrets(namespace).List(context.TODO(), opts)
	}
	watchSecret := func(namespace string, opts metav1.ListOptions) (watch.Interface, error) {
		return kubeClient.CoreV1().Secrets(namespace).Watch(context.TODO(), opts)
	}
	newSecret := func() runtime.Object {
		return &v1.Secret{}
	}
	isImmutable := func(object runtime.Object) bool {
		if secret, ok := object.(*v1.Secret); ok {
			return secret.Immutable != nil && *secret.Immutable
		}
		return false
	}
	gr := corev1.Resource("secret")
	return &secretManager{
		manager: manager.NewWatchBasedManager(listSecret, watchSecret, newSecret, isImmutable, gr, resyncInterval, getSecretNames),
	}
}
```

