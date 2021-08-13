# Pod

此package包括MirrorClient和[Pod]Manager两个接口。kubelet可以创建static pod，这些static pod由kubelet而非kubernetes controller来管理，但是需要告知kubernetes api server这些静态pod的存在。因此每一个static pod创建后，kubelet会在api server创建一个mirror pod，来代替这个static pod.

## MirrorClient

### MirrorClient接口

MirrorClient接口用于mirror pod的创建和删除

```go
type MirrorClient interface {
	// CreateMirrorPod creates a mirror pod in the API server for the given
	// pod or returns an error.  The mirror pod will have the same annotations
	// as the given pod as well as an extra annotation containing the hash of
	// the static pod.
	CreateMirrorPod(pod *v1.Pod) error
	// DeleteMirrorPod deletes the mirror pod with the given full name from
	// the API server or returns an error.
	DeleteMirrorPod(podFullName string, uid *types.UID) (bool, error)
}
```

### nodeGetter

nodeGetter通过节点名获取代表节点的数据结构

```go
// nodeGetter is a subset of NodeLister, simplified for testing.
type nodeGetter interface {
	// Get retrieves the Node for a given name.
	Get(name string) (*v1.Node, error)
}
```

### basicMirrorClient

basicMirrorClient实现了MirrorClient接口

```go
// basicMirrorClient is a functional MirrorClient.  Mirror pods are stored in
// the kubelet directly because they need to be in sync with the internal
// pods.
type basicMirrorClient struct {
	apiserverClient clientset.Interface
	nodeGetter      nodeGetter
	nodeName        string
}
```

#### 构造函数

```go
// NewBasicMirrorClient returns a new MirrorClient.
func NewBasicMirrorClient(apiserverClient clientset.Interface, nodeName string, nodeGetter nodeGetter) MirrorClient {
	return &basicMirrorClient{
		apiserverClient: apiserverClient,
		nodeName:        nodeName,
		nodeGetter:      nodeGetter,
	}
}
```

#### 方法

`func (mc *basicMirrorClient) CreateMirrorPod(pod *v1.Pod) error`

在api server中为pod创建一个static pod

`func (mc *basicMirrorClient) DeleteMirrorPod(podFullName string, uid *types.UID) (bool, error)`

从api server中删除一个static pod

`func (mc *basicMirrorClient) getNodeUID() (types.UID, error)`

获取当前节点的UID

`func IsStaticPod(pod *v1.Pod) bool`

判断给定的pod是否是static pod

## Manager

存储和管理pods，维护static pods和mirror pods之间的映射关系。

kubelet可以从三个地方获取pod的更新信息: 文件、http和API server。从非apiserver获取的pods被称为static pods，API server不知道static pods的存在。因此为了能够监控static pods的状态，kubelet对每一个static pod会在API server中创建一个mirror pod。

### Manager interface

```go
// Manager stores and manages access to pods, maintaining the mappings
// between static pods and mirror pods.
//
// The kubelet discovers pod updates from 3 sources: file, http, and
// apiserver. Pods from non-apiserver sources are called static pods, and API
// server is not aware of the existence of static pods. In order to monitor
// the status of such pods, the kubelet creates a mirror pod for each static
// pod via the API server.
//
// A mirror pod has the same pod full name (name and namespace) as its static
// counterpart (albeit different metadata such as UID, etc). By leveraging the
// fact that the kubelet reports the pod status using the pod full name, the
// status of the mirror pod always reflects the actual status of the static
// pod. When a static pod gets deleted, the associated orphaned mirror pod
// will also be removed.
type Manager interface {
	// GetPods returns the regular pods bound to the kubelet and their spec.
	GetPods() []*v1.Pod
	// GetPodByFullName returns the (non-mirror) pod that matches full name, as well as
	// whether the pod was found.
	GetPodByFullName(podFullName string) (*v1.Pod, bool)
	// GetPodByName provides the (non-mirror) pod that matches namespace and
	// name, as well as whether the pod was found.
	GetPodByName(namespace, name string) (*v1.Pod, bool)
	// GetPodByUID provides the (non-mirror) pod that matches pod UID, as well as
	// whether the pod is found.
	GetPodByUID(types.UID) (*v1.Pod, bool)
	// GetPodByMirrorPod returns the static pod for the given mirror pod and
	// whether it was known to the pod manager.
	GetPodByMirrorPod(*v1.Pod) (*v1.Pod, bool)
	// GetMirrorPodByPod returns the mirror pod for the given static pod and
	// whether it was known to the pod manager.
	GetMirrorPodByPod(*v1.Pod) (*v1.Pod, bool)
	// GetPodsAndMirrorPods returns the both regular and mirror pods.
	GetPodsAndMirrorPods() ([]*v1.Pod, []*v1.Pod)
	// SetPods replaces the internal pods with the new pods.
	// It is currently only used for testing.
	SetPods(pods []*v1.Pod)
	// AddPod adds the given pod to the manager.
	AddPod(pod *v1.Pod)
	// UpdatePod updates the given pod in the manager.
	UpdatePod(pod *v1.Pod)
	// DeletePod deletes the given pod from the manager.  For mirror pods,
	// this means deleting the mappings related to mirror pods.  For non-
	// mirror pods, this means deleting from indexes for all non-mirror pods.
	DeletePod(pod *v1.Pod)
	// GetOrphanedMirrorPodNames returns names of orphaned mirror pods
	GetOrphanedMirrorPodNames() []string
	// TranslatePodUID returns the actual UID of a pod. If the UID belongs to
	// a mirror pod, returns the UID of its static pod. Otherwise, returns the
	// original UID.
	//
	// All public-facing functions should perform this translation for UIDs
	// because user may provide a mirror pod UID, which is not recognized by
	// internal Kubelet functions.
	TranslatePodUID(uid types.UID) kubetypes.ResolvedPodUID
	// GetUIDTranslations returns the mappings of static pod UIDs to mirror pod
	// UIDs and mirror pod UIDs to static pod UIDs.
	GetUIDTranslations() (podToMirror map[kubetypes.ResolvedPodUID]kubetypes.MirrorPodUID, mirrorToPod map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID)
	// IsMirrorPodOf returns true if mirrorPod is a correct representation of
	// pod; false otherwise.
	IsMirrorPodOf(mirrorPod, pod *v1.Pod) bool

	MirrorClient
}
```

### basicManager

basicManager实现了Manager接口

```go
// basicManager is a functional Manager.
//
// All fields in basicManager are read-only and are updated calling SetPods,
// AddPod, UpdatePod, or DeletePod.
type basicManager struct {
	// Protects all internal maps.
	lock sync.RWMutex

	// Regular pods indexed by UID. 正常的pods
	podByUID map[kubetypes.ResolvedPodUID]*v1.Pod
	// Mirror pods indexed by UID. 
	mirrorPodByUID map[kubetypes.MirrorPodUID]*v1.Pod

	// Pods indexed by full name for easy access.
	podByFullName       map[string]*v1.Pod
	mirrorPodByFullName map[string]*v1.Pod

	// Mirror pod UID to pod UID map.
	translationByUID map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID

	// basicManager is keeping secretManager and configMapManager up-to-date.
	secretManager    secret.Manager
	configMapManager configmap.Manager

	// A mirror pod client to create/delete mirror pods.
	MirrorClient
}
```

#### 构造函数

```go
// NewBasicPodManager returns a functional Manager.
func NewBasicPodManager(client MirrorClient, secretManager secret.Manager, configMapManager configmap.Manager) Manager {
	pm := &basicManager{}
	pm.secretManager = secretManager
	pm.configMapManager = configMapManager
	pm.MirrorClient = client
	pm.SetPods(nil)
	return pm
}
```

#### 方法

* SetPods

  `func (pm *basicManager) SetPods(newPods []*v1.Pod)`

  根据参数newPods设置basicManager中的pods

* AddPod

  `func (pm *basicManager) AddPod(pod *v1.Pod)`

  将pod添加到basicManager中

* UpdatePod

  `func (pm *basicManager) UpdatePod(pod *v1.Pod)`

  更新basicManager中的pod

* DeletePod

  `func (pm *basicManager) DeletePod(pod *v1.Pod)`

  将pod从basicManager中删除

* GetPods

  `func (pm *basicManager) GetPods() []*v1Pod`

  获取basicManager中所以的正常pods

* GetPodsAndMirrorPods

  `func (pm *basicManager) GetPodsAndMirrorPods() ([]*v1.Pod, []*v1.Pod)`

  获取basicManager中所以的普通pods和mirror pods

* GetPodByUID

  `func (pm *basicManager) GetPodByUID(uid types.UID) (*v1.Pod, bool)`

  通过UID获取相应的普通pod

* GetPodByFullName

  `func (pm *basicManager) GetPodByFullName(podFullName string) (*v1.Pod, bool)`

  通过full name(name_namespace)获取相应的普通pod

* GetPodByName

  `func (pm *basicManager) GetPodByName(namespace, name string) (*v1.Pod, bool)`

  通过namespace和name获取相应的普通pod

* TranslatePodUID

  `func (pm *basicManager) TranslatePodUID(uid types.UID) kubetypes.ResolvedPodUID`

  如果uid属于一个mirror pod，则返回相应的static pod；如果uid为空或者属于普通pod,则直接构造ResolvedPodUID

* GetUIDTranslations

  `func (pm *basicMananger) GetUIDTranslations() (podToMirror map[kubetypes.ResolvedPodUID]kubetypes.MirrorPodUID, mirrorToPod map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID)`

  返回basicManager中pods到mirror pods的映射和static pods到pods的映射

* GetOrphanedMirrorPodName

  `func (pm *basicManager) GetOrphanedMirrorPodNames() []string`

  获取basicManager中无普通pods的static pods的名称

* IsMirrorPodOf

  `func (pm *basicManager) IsMirrorPodOf(mirrorPod, pod *v1.Pod) bool`

  判断mirrorPod是否是pod的镜像pod

* GetMirrorPodByPod

  `func (pm *basicManager) GetMirrorPodByPod(pod *v1.Pod) (*v1.Pod, bool)`

  获取pod相应的mirror pod

* GetPodByMirrorPod

  `func (pm *basicManager) GetPodByMirrorPod(mirrorPod *v1.Pod) (*v1.Pod, bool)`

  获取mirror pod相应的普通pod

