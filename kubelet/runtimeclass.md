# runtimeclass

## Manager

管理runtimeclass的Manager

```go
// Manager caches RuntimeClass API objects, and provides accessors to the Kubelet.
type Manager struct {
	informerFactory informers.SharedInformerFactory
	lister          nodev1.RuntimeClassLister
}
```

### 构造函数

```go
// NewManager returns a new RuntimeClass Manager. Run must be called before the manager can be used.
func NewManager(client clientset.Interface) *Manager {
	const resyncPeriod = 0

	factory := informers.NewSharedInformerFactory(client, resyncPeriod)
	lister := factory.Node().V1().RuntimeClasses().Lister()

	return &Manager{
		informerFactory: factory,
		lister:          lister,
	}
}
```

### 方法

* Start

  Start启动informerFactory

  ```go
  // Start starts syncing the RuntimeClass cache with the apiserver.
  func (m *Manager) Start(stopCh <-chan struct{}) {
  	m.informerFactory.Start(stopCh)
  }
  ```

* WaitForCacheSync

  暴露informerFactory的WaitForCacheSync方法，测试用

  ```go
  // WaitForCacheSync exposes the WaitForCacheSync method on the informer factory for testing
  // purposes.
  func (m *Manager) WaitForCacheSync(stopCh <-chan struct{}) {
  	m.informerFactory.WaitForCacheSync(stopCh)
  }
  ```

* LookupRuntimeHandler

  `func (m *Manager) LookupRuntimeHandler(runtimeClassName *string) (string, error)`

  返回名为runtimeClassName的RuntimeHandler