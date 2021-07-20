# cache

## nodeTree

nodeTree是一个保存每个zone中node名的树形数据结构。其中nodeTree.tree的键是zone的名字，值是node名字的数组。

### 数据结构

nodeTree的数据结构如下：

```go
// nodeTree is a tree-like data structure that holds node names in each zone. Zone names are
// keys to "NodeTree.tree" and values of "NodeTree.tree" are arrays of node names.
// NodeTree is NOT thread-safe, any concurrent updates/reads from it must be synchronized by the caller.
// It is used only by schedulerCache, and should stay as such.
type nodeTree struct {
   tree     map[string][]string // a map from zone (region-zone) to an array of nodes in the zone.
   zones    []string            // a list of all the zones in the tree (keys)
   numNodes int                 // 总的node的数量
}

```

可通过传入当前集群的所有node来构造nodeTree：

```go
// newNodeTree creates a NodeTree from nodes.
func newNodeTree(nodes []*v1.Node) *nodeTree {
   nt := &nodeTree{
      tree: make(map[string][]string),
   }
   for _, n := range nodes {
      nt.addNode(n)
   }
   return nt
}
```

### 方法

nodeTree的方法有：

addNode：将一个node添加到nodeTree中

```go
// addNode adds a node and its corresponding zone to the tree. If the zone already exists, the node
// is added to the array of nodes in that zone.
func (nt *nodeTree) addNode(n *v1.Node)
```

removeNode：将一个node从nodeTree中删除

```go
// removeNode removes a node from the NodeTree.
func (nt *nodeTree) removeNode(n *v1.Node) error
```

removeZone：将一个zone及其node从nodeTree中删除

```go
// removeZone removes a zone from tree.
// This function must be called while writer locks are hold.
func (nt *nodeTree) removeZone(zone string)
```

updateNode：更新nodeTree中的一个node

```go
// updateNode updates a node in the NodeTree.
func (nt *nodeTree) updateNode(old, new *v1.Node)
```

list：列出nodeTree中的所有node

```go
// list returns the list of names of the node. NodeTree iterates over zones and in each zone iterates
// over nodes in a round robin fashion.
func (nt *nodeTree) list() ([]string, error) {
   if len(nt.zones) == 0 {
      return nil, nil
   }
   nodesList := make([]string, 0, nt.numNodes)
   numExhaustedZones := 0
   nodeIndex := 0
   for len(nodesList) < nt.numNodes {
      if numExhaustedZones >= len(nt.zones) { // all zones are exhausted.
         return nodesList, errors.New("all zones exhausted before reaching count of nodes expected")
      }
      for zoneIndex := 0; zoneIndex < len(nt.zones); zoneIndex++ {
         na := nt.tree[nt.zones[zoneIndex]]
         if nodeIndex >= len(na) { // If the zone is exhausted, continue
            if nodeIndex == len(na) { // If it is the first time the zone is exhausted
               numExhaustedZones++
            }
            continue
         }
         nodesList = append(nodesList, na[nodeIndex])
      }
      nodeIndex++
   }
   return nodesList, nil
}
```

需要注意的是上面函数中，外层的for循环在正常情况下只执行一次。



## Snapshot

Snapshot是NodeInfo的一次快照。调度器在每个调度周期之前做执行一次快照然后在整个调度过程中使用它。Snapshot中的数据是通过cache接口进行更新的。

### 数据结构

```go
// Snapshot is a snapshot of cache NodeInfo and NodeTree order. The scheduler takes a
// snapshot at the beginning of each scheduling cycle and uses it for its operations in that cycle.
type Snapshot struct {
   // nodeInfoMap a map of node name to a snapshot of its NodeInfo.
   nodeInfoMap map[string]*framework.NodeInfo
   // nodeInfoList is the list of nodes as ordered in the cache's nodeTree.
   nodeInfoList []*framework.NodeInfo
   // havePodsWithAffinityNodeInfoList is the list of nodes with at least one pod declaring affinity terms.
   havePodsWithAffinityNodeInfoList []*framework.NodeInfo
   // havePodsWithRequiredAntiAffinityNodeInfoList is the list of nodes with at least one pod declaring
   // required anti-affinity terms.
   havePodsWithRequiredAntiAffinityNodeInfoList []*framework.NodeInfo
   generation                                   int64
}
```

其中：

* nodeInfoMap是node名到NodeInfo的映射
* nodeInfoList是nodeInfoMap中所有node的NodeInfo的列表
* generation

Snapshot实现了framework.SharedLister接口：

```go
var _ framework.SharedLister = &Snapshot{}

// SharedLister groups scheduler-specific listers.
type SharedLister interface {
	NodeInfos() NodeInfoLister
}
```

通过当前集群中所有的pod和node来构造Snapshot

```go
// NewSnapshot initializes a Snapshot struct and returns it.
func NewSnapshot(pods []*v1.Pod, nodes []*v1.Node) *Snapshot {
   nodeInfoMap := createNodeInfoMap(pods, nodes)
   nodeInfoList := make([]*framework.NodeInfo, 0, len(nodeInfoMap))
   havePodsWithAffinityNodeInfoList := make([]*framework.NodeInfo, 0, len(nodeInfoMap))
   havePodsWithRequiredAntiAffinityNodeInfoList := make([]*framework.NodeInfo, 0, len(nodeInfoMap))
   for _, v := range nodeInfoMap {
      nodeInfoList = append(nodeInfoList, v)
      if len(v.PodsWithAffinity) > 0 {
         havePodsWithAffinityNodeInfoList = append(havePodsWithAffinityNodeInfoList, v)
      }
      if len(v.PodsWithRequiredAntiAffinity) > 0 {
         havePodsWithRequiredAntiAffinityNodeInfoList = append(havePodsWithRequiredAntiAffinityNodeInfoList, v)
      }
   }

   s := NewEmptySnapshot()
   s.nodeInfoMap = nodeInfoMap
   s.nodeInfoList = nodeInfoList
   s.havePodsWithAffinityNodeInfoList = havePodsWithAffinityNodeInfoList
   s.havePodsWithRequiredAntiAffinityNodeInfoList = havePodsWithRequiredAntiAffinityNodeInfoList

   return s
}

// createNodeInfoMap obtains a list of pods and pivots that list into a map
// where the keys are node names and the values are the aggregated information
// for that node.
func createNodeInfoMap(pods []*v1.Pod, nodes []*v1.Node) map[string]*framework.NodeInfo {
	nodeNameToInfo := make(map[string]*framework.NodeInfo)
	for _, pod := range pods {
		nodeName := pod.Spec.NodeName
		if _, ok := nodeNameToInfo[nodeName]; !ok {
			nodeNameToInfo[nodeName] = framework.NewNodeInfo()
		}
		nodeNameToInfo[nodeName].AddPod(pod)
	}
	imageExistenceMap := createImageExistenceMap(nodes)

	for _, node := range nodes {
		if _, ok := nodeNameToInfo[node.Name]; !ok {
			nodeNameToInfo[node.Name] = framework.NewNodeInfo()
		}
		nodeInfo := nodeNameToInfo[node.Name]
		nodeInfo.SetNode(node)
		nodeInfo.ImageStates = getNodeImageStates(node, imageExistenceMap)
	}
	return nodeNameToInfo
}

// getNodeImageStates returns the given node's image states based on the given imageExistence map.
func getNodeImageStates(node *v1.Node, imageExistenceMap map[string]sets.String) map[string]*framework.ImageStateSummary {
	imageStates := make(map[string]*framework.ImageStateSummary)

	for _, image := range node.Status.Images {
		for _, name := range image.Names {
			imageStates[name] = &framework.ImageStateSummary{
				Size:     image.SizeBytes,
				NumNodes: len(imageExistenceMap[name]),
			}
		}
	}
	return imageStates
}

// createImageExistenceMap returns a map recording on which nodes the images exist, keyed by the images' names.
func createImageExistenceMap(nodes []*v1.Node) map[string]sets.String {
	imageExistenceMap := make(map[string]sets.String)
	for _, node := range nodes {
		for _, image := range node.Status.Images {
			for _, name := range image.Names {
				if _, ok := imageExistenceMap[name]; !ok {
					imageExistenceMap[name] = sets.NewString(node.Name)
				} else {
					imageExistenceMap[name].Insert(node.Name)
				}
			}
		}
	}
	return imageExistenceMap
}

// NewEmptySnapshot initializes a Snapshot struct and returns it.
func NewEmptySnapshot() *Snapshot {
	return &Snapshot{
		nodeInfoMap: make(map[string]*framework.NodeInfo),
	}
}
```

### 方法

```go
// NodeInfos returns a NodeInfoLister.
// 因为Snapshot实现了NodInfoLister接口
func (s *Snapshot) NodeInfos() framework.NodeInfoLister {
   return s
}

// NumNodes returns the number of nodes in the snapshot.
func (s *Snapshot) NumNodes() int {
   return len(s.nodeInfoList)
}

// List returns the list of nodes in the snapshot.
func (s *Snapshot) List() ([]*framework.NodeInfo, error) {
   return s.nodeInfoList, nil
}

// HavePodsWithAffinityList returns the list of nodes with at least one pod with inter-pod affinity
func (s *Snapshot) HavePodsWithAffinityList() ([]*framework.NodeInfo, error) {
   return s.havePodsWithAffinityNodeInfoList, nil
}

// HavePodsWithRequiredAntiAffinityList returns the list of nodes with at least one pod with
// required inter-pod anti-affinity
func (s *Snapshot) HavePodsWithRequiredAntiAffinityList() ([]*framework.NodeInfo, error) {
   return s.havePodsWithRequiredAntiAffinityNodeInfoList, nil
}

// Get returns the NodeInfo of the given node name.
func (s *Snapshot) Get(nodeName string) (*framework.NodeInfo, error) {
   if v, ok := s.nodeInfoMap[nodeName]; ok && v.Node() != nil {
      return v, nil
   }
   return nil, fmt.Errorf("nodeinfo not found for node name %q", nodeName)
}
```

Snapshot的方法都很浅显

## Cache接口

cache缓存调度过程中的pod的信息

```go
// Cache collects pods' information and provides node-level aggregated information.
// It's intended for generic scheduler to do efficient lookup.
// Cache's operations are pod centric. It does incremental updates based on pod events.
// Pod events are sent via network. We don't have guaranteed delivery of all events:
// We use Reflector to list and watch from remote.
// Reflector might be slow and do a relist, which would lead to missing events.
//
// State Machine of a pod's events in scheduler's cache:
//
//
//   +-------------------------------------------+  +----+
//   |                            Add            |  |    |
//   |                                           |  |    | Update
//   +      Assume                Add            v  v    |
//Initial +--------> Assumed +------------+---> Added <--+
//   ^                +   +               |       +
//   |                |   |               |       |
//   |                |   |           Add |       | Remove
//   |                |   |               |       |
//   |                |   |               +       |
//   +----------------+   +-----------> Expired   +----> Deleted
//         Forget             Expire
//
//
// Note that an assumed pod can expire, because if we haven't received Add event notifying us
// for a while, there might be some problems and we shouldn't keep the pod in cache anymore.
//
// Note that "Initial", "Expired", and "Deleted" pods do not actually exist in cache.
// Based on existing use cases, we are making the following assumptions:
// - No pod would be assumed twice
// - A pod could be added without going through scheduler. In this case, we will see Add but not Assume event.
// - If a pod wasn't added, it wouldn't be removed or updated.
// - Both "Expired" and "Deleted" are valid end states. In case of some problems, e.g. network issue,
//   a pod might have changed its state (e.g. added and deleted) without delivering notification to the cache.
type Cache interface {
   // NodeCount returns the number of nodes in the cache.
   // DO NOT use outside of tests.
   // 测试用，不要在测试外使用
   NodeCount() int

   // PodCount returns the number of pods in the cache (including those from deleted nodes).
   // DO NOT use outside of tests.
   // 测试用，不要在测试外使用
   PodCount() (int, error)

   // AssumePod assumes a pod scheduled and aggregates the pod's information into its node.
   // The implementation also decides the policy to expire pod before being confirmed (receiving Add event).
   // After expiration, its information would be subtracted.
   AssumePod(pod *v1.Pod) error

   // FinishBinding signals that cache for assumed pod can be expired
   FinishBinding(pod *v1.Pod) error

   // ForgetPod removes an assumed pod from cache.
   ForgetPod(pod *v1.Pod) error

   // AddPod either confirms a pod if it's assumed, or adds it back if it's expired.
   // If added back, the pod's information would be added again.
   AddPod(pod *v1.Pod) error

   // UpdatePod removes oldPod's information and adds newPod's information.
   UpdatePod(oldPod, newPod *v1.Pod) error

   // RemovePod removes a pod. The pod's information would be subtracted from assigned node.
   RemovePod(pod *v1.Pod) error

   // GetPod returns the pod from the cache with the same namespace and the
   // same name of the specified pod.
   GetPod(pod *v1.Pod) (*v1.Pod, error)

   // IsAssumedPod returns true if the pod is assumed and not expired.
   IsAssumedPod(pod *v1.Pod) (bool, error)

   // AddNode adds overall information about node.
   // It returns a clone of added NodeInfo object.
   AddNode(node *v1.Node) *framework.NodeInfo

   // UpdateNode updates overall information about node.
   // It returns a clone of updated NodeInfo object.
   UpdateNode(oldNode, newNode *v1.Node) *framework.NodeInfo

   // RemoveNode removes overall information about node.
   RemoveNode(node *v1.Node) error

   // UpdateSnapshot updates the passed infoSnapshot to the current contents of Cache.
   // The node info contains aggregated information of pods scheduled (including assumed to be)
   // on this node.
   // The snapshot only includes Nodes that are not deleted at the time this function is called.
   // nodeinfo.Node() is guaranteed to be not nil for all the nodes in the snapshot.
   UpdateSnapshot(nodeSnapshot *Snapshot) error

   // Dump produces a dump of the current cache.
   // 进行调试时使用，不要在其他情况下使用
   Dump() *Dump
}

// Dump is a dump of the cache state.
type Dump struct {
   AssumedPods sets.String
   Nodes       map[string]*framework.NodeInfo
}
```

## schedulerCache

schedulerCache实现了Cache接口，其数据结构如下：

```go
type schedulerCache struct {
   stop   <-chan struct{}
   ttl    time.Duration     // assumed pod的超时时间
   period time.Duration     // assumed pod超时的检查时间

   // This mutex guards all fields within this cache struct.
   mu sync.RWMutex
   // a set of assumed pod keys.
   // The key could further be used to get an entry in podStates.
   assumedPods sets.String        // assumed pods
   // a map from pod key to podState.
   podStates map[string]*podState          // 保存的除了assumedPods外还有已经被调度的pod
   nodes     map[string]*nodeInfoListItem  // nodeInfo的双向链表，最近更新的node在最前边
   // headNode points to the most recently updated NodeInfo in "nodes". It is the
   // head of the linked list.
   headNode *nodeInfoListItem              // 保存nodes的第一个元素
   nodeTree *nodeTree                      // 见上面
   // A map from image name to its imageState.
   imageStates map[string]*imageState      // nodes上的容器镜像的信息，键为镜像名，值为imageState
}
```

其中nodeInfoListItem、podState和imageState的定义如下：

```go
// nodeInfoListItem holds a NodeInfo pointer and acts as an item in a doubly
// linked list. When a NodeInfo is updated, it goes to the head of the list.
// The items closer to the head are the most recently updated items.
type nodeInfoListItem struct {
   info *framework.NodeInfo
   next *nodeInfoListItem
   prev *nodeInfoListItem
}

type podState struct {
	pod *v1.Pod
	// Used by assumedPod to determinate expiration.
	deadline *time.Time
	// Used to block cache from expiring assumedPod if binding still runs
	bindingFinished bool
}

type imageState struct {
	// Size of the image
	size int64
	// A set of node names for nodes having this image present
	nodes sets.String
}
```

### 方法

* `createImageStateSummary(state *imageState) *framework.ImageStateSummary` 由imageState创建一个*framework.ImageStateSummary

* `moveNodeInfoToHead(name string)`将名为name的node移动nodes链表的头部并修改headNode
* `removeNodeInfoFromList(name string)`将名为name的node从nodes链表中删除，如果被删除的节点是头节点，则同时修改headNode
* `Dump() *Dump`拷贝并返回所有的node和pod。此操作比较耗时，因此慎用。
* `NodeCount() int`返回Cache中node的数量（供测试用）
* `PodCount() (int, error)`返回Cache中pod的数量（供测试用）
* `IsAssumedPod(pod *v1.Pod) (bool, error)` pod是否assumed
* `GetPod(pod *v1.Pod) (*v1.Pod, error)` 返回Cache中与Pod缓存的对象

AssumePod   假设pod被调度到某个node上并在Cache中进行记录

```go
func (cache *schedulerCache) AssumePod(pod *v1.Pod) error {
	key, err := framework.GetPodKey(pod)
	if err != nil {
		return err
	}

	cache.mu.Lock()
	defer cache.mu.Unlock()
	if _, ok := cache.podStates[key]; ok {    // 同一pod不能被assume两次
		return fmt.Errorf("pod %v is in the cache, so can't be assumed", key)
	}

	cache.addPod(pod)
	ps := &podState{
		pod: pod,
	}
	cache.podStates[key] = ps
	cache.assumedPods.Insert(key)
	return nil
}

// Assumes that lock is already acquired.
func (cache *schedulerCache) addPod(pod *v1.Pod) {
	n, ok := cache.nodes[pod.Spec.NodeName]
	if !ok {
		n = newNodeInfoListItem(framework.NewNodeInfo())
		cache.nodes[pod.Spec.NodeName] = n
	}
	n.info.AddPod(pod)
	cache.moveNodeInfoToHead(pod.Spec.NodeName)
}
```

FinishBinding   调度器对pod的绑定操作完成，此函数更新Cache中pod的绑定状态

```go
func (cache *schedulerCache) FinishBinding(pod *v1.Pod) error {
   return cache.finishBinding(pod, time.Now())
}

// finishBinding exists to make tests determinitistic by injecting now as an argument
func (cache *schedulerCache) finishBinding(pod *v1.Pod, now time.Time) error {
   key, err := framework.GetPodKey(pod)
   if err != nil {
      return err
   }

   cache.mu.RLock()
   defer cache.mu.RUnlock()

   klog.V(5).Infof("Finished binding for pod %v. Can be expired.", key)
   currState, ok := cache.podStates[key]
   if ok && cache.assumedPods.Has(key) {
      dl := now.Add(cache.ttl)
      currState.bindingFinished = true
      currState.deadline = &dl
   }
   return nil
}
```

ForgetPod 在调度器对pod的调度失败后，通过ForgetPod从Cache中assumedPods中删除pod的信息

```go
func (cache *schedulerCache) ForgetPod(pod *v1.Pod) error {
   key, err := framework.GetPodKey(pod)
   if err != nil {
      return err
   }

   cache.mu.Lock()
   defer cache.mu.Unlock()

   currState, ok := cache.podStates[key]
   if ok && currState.pod.Spec.NodeName != pod.Spec.NodeName {
      return fmt.Errorf("pod %v was assumed on %v but assigned to %v", key, pod.Spec.NodeName, currState.pod.Spec.NodeName)
   }

   switch {
   // Only assumed pod can be forgotten.
   case ok && cache.assumedPods.Has(key):
      err := cache.removePod(pod)
      if err != nil {
         return err
      }
      delete(cache.assumedPods, key)
      delete(cache.podStates, key)
   default:
      return fmt.Errorf("pod %v wasn't assumed so cannot be forgotten", key)
   }
   return nil
}
```

AddPod  pod调度成功，更新Cache中的状态，如果pod在assumePods中，则将其删除，如果pod不在assumedPods中则将pod添加到其被调度到的node中。

```go
func (cache *schedulerCache) AddPod(pod *v1.Pod) error {
   key, err := framework.GetPodKey(pod)
   if err != nil {
      return err
   }

   cache.mu.Lock()
   defer cache.mu.Unlock()

   currState, ok := cache.podStates[key]
   switch {
   case ok && cache.assumedPods.Has(key):
      if currState.pod.Spec.NodeName != pod.Spec.NodeName {
         // The pod was added to a different node than it was assumed to.
         klog.Warningf("Pod %v was assumed to be on %v but got added to %v", key, pod.Spec.NodeName, currState.pod.Spec.NodeName)
         // Clean this up.
         if err = cache.removePod(currState.pod); err != nil {
            klog.Errorf("removing pod error: %v", err)
         }
         cache.addPod(pod)
      }
      delete(cache.assumedPods, key)
      cache.podStates[key].deadline = nil
      cache.podStates[key].pod = pod
   case !ok:
      // Pod was expired. We should add it back.
      cache.addPod(pod)
      ps := &podState{
         pod: pod,
      }
      cache.podStates[key] = ps
   default:
      return fmt.Errorf("pod %v was already in added state", key)
   }
   return nil
}
```

UpdatePod pod有更新，更新Cache中的数据

```go
func (cache *schedulerCache) UpdatePod(oldPod, newPod *v1.Pod) error {
   key, err := framework.GetPodKey(oldPod)
   if err != nil {
      return err
   }

   cache.mu.Lock()
   defer cache.mu.Unlock()

   currState, ok := cache.podStates[key]
   switch {
   // An assumed pod won't have Update/Remove event. It needs to have Add event
   // before Update event, in which case the state would change from Assumed to Added.
   case ok && !cache.assumedPods.Has(key):
      // Pod被调度后只能运行在被调度的node，不能再被调度到其他的node上去
      if currState.pod.Spec.NodeName != newPod.Spec.NodeName {
         klog.Errorf("Pod %v updated on a different node than previously added to.", key)
         klog.Fatalf("Schedulercache is corrupted and can badly affect scheduling decisions")
      }
      if err := cache.updatePod(oldPod, newPod); err != nil {
         return err
      }
      currState.pod = newPod
   default:
      return fmt.Errorf("pod %v is not added to scheduler cache, so cannot be updated", key)
   }
   return nil
}

// Assumes that lock is already acquired.
func (cache *schedulerCache) updatePod(oldPod, newPod *v1.Pod) error {
	if err := cache.removePod(oldPod); err != nil {
		return err
	}
	cache.addPod(newPod)
	return nil
}
```

RemovePod将pod从Cache中删除

```go
func (cache *schedulerCache) RemovePod(pod *v1.Pod) error {
   key, err := framework.GetPodKey(pod)
   if err != nil {
      return err
   }

   cache.mu.Lock()
   defer cache.mu.Unlock()

   currState, ok := cache.podStates[key]
   switch {
   // An assumed pod won't have Delete/Remove event. It needs to have Add event
   // before Remove event, in which case the state would change from Assumed to Added.
   case ok && !cache.assumedPods.Has(key):
      if currState.pod.Spec.NodeName != pod.Spec.NodeName {
         klog.Errorf("Pod %v was assumed to be on %v but got added to %v", key, pod.Spec.NodeName, currState.pod.Spec.NodeName)
         klog.Fatalf("Schedulercache is corrupted and can badly affect scheduling decisions")
      }
      err := cache.removePod(currState.pod)
      if err != nil {
         return err
      }
      delete(cache.podStates, key)
   default:
      return fmt.Errorf("pod %v is not found in scheduler cache, so cannot be removed from it", key)
   }
   return nil
}

// Assumes that lock is already acquired.
// Removes a pod from the cached node info. If the node information was already
// removed and there are no more pods left in the node, cleans up the node from
// the cache.
func (cache *schedulerCache) removePod(pod *v1.Pod) error {
	n, ok := cache.nodes[pod.Spec.NodeName]
	if !ok {
		klog.Errorf("node %v not found when trying to remove pod %v", pod.Spec.NodeName, pod.Name)
		return nil
	}
	if err := n.info.RemovePod(pod); err != nil {
		return err
	}
  // 删除node时，node上的pod可能还没有被删除（事件到达的先后顺序可能不一致）
  // 在node被删除时，如果node上的pod数量不为0，则Cache中的node不会被删除，仅把nodeInfoListItem里面的info的Node置为空
  // 因此在删除pod时如果pod所在的node上pod的数量为0，并且Node被为nil，表示此node已经被删除，在这里要把node从Cache中删除
	if len(n.info.Pods) == 0 && n.info.Node() == nil {
		cache.removeNodeInfoFromList(pod.Spec.NodeName)
	} else {
		cache.moveNodeInfoToHead(pod.Spec.NodeName)
	}
	return nil
}
```

AddNode 在Cache中添加Node

```go
func (cache *schedulerCache) AddNode(node *v1.Node) *framework.NodeInfo {
   cache.mu.Lock()
   defer cache.mu.Unlock()

   n, ok := cache.nodes[node.Name]
   if !ok {
      n = newNodeInfoListItem(framework.NewNodeInfo())
      cache.nodes[node.Name] = n
   } else {
      cache.removeNodeImageStates(n.info.Node())    // 因为imageStates里面保存的是所有node的image的信息，因此单个node变化时要更新imageStates
   }
   cache.moveNodeInfoToHead(node.Name)

   cache.nodeTree.addNode(node)
   cache.addNodeImageStates(node, n.info)
   n.info.SetNode(node)
   return n.info.Clone()
}
```

UpdateNode 在Cache中更新node

```go
func (cache *schedulerCache) UpdateNode(oldNode, newNode *v1.Node) *framework.NodeInfo {
   cache.mu.Lock()
   defer cache.mu.Unlock()

   n, ok := cache.nodes[newNode.Name]
   if !ok {
      n = newNodeInfoListItem(framework.NewNodeInfo())
      cache.nodes[newNode.Name] = n
      cache.nodeTree.addNode(newNode)
   } else {
      cache.removeNodeImageStates(n.info.Node())
   }
   cache.moveNodeInfoToHead(newNode.Name)

   cache.nodeTree.updateNode(oldNode, newNode)
   cache.addNodeImageStates(newNode, n.info)
   n.info.SetNode(newNode)
   return n.info.Clone()
}
```

RemoveNode 从Cache中删除node

```go
// RemoveNode removes a node from the cache's tree.
// The node might still have pods because their deletion events didn't arrive
// yet. Those pods are considered removed from the cache, being the node tree
// the source of truth.
// However, we keep a ghost node with the list of pods until all pod deletion
// events have arrived. A ghost node is skipped from snapshots.
func (cache *schedulerCache) RemoveNode(node *v1.Node) error {
   cache.mu.Lock()
   defer cache.mu.Unlock()

   n, ok := cache.nodes[node.Name]
   if !ok {
      return fmt.Errorf("node %v is not found", node.Name)
   }
   n.info.RemoveNode()
   // We remove NodeInfo for this node only if there aren't any pods on this node.
   // We can't do it unconditionally, because notifications about pods are delivered
   // in a different watch, and thus can potentially be observed later, even though
   // they happened before node removal.
   if len(n.info.Pods) == 0 {
      cache.removeNodeInfoFromList(node.Name)
   } else {
      cache.moveNodeInfoToHead(node.Name)
   }
   if err := cache.nodeTree.removeNode(node); err != nil {
      return err
   }
   cache.removeNodeImageStates(node)
   return nil
}
```

cleanupAssumedPods 被定时调用，如果assumedPods超时，则将其expire掉

```go
// cleanupAssumedPods exists for making test deterministic by taking time as input argument.
// It also reports metrics on the cache size for nodes, pods, and assumed pods.
func (cache *schedulerCache) cleanupAssumedPods(now time.Time) {
   cache.mu.Lock()
   defer cache.mu.Unlock()
   defer cache.updateMetrics()

   // The size of assumedPods should be small
   for key := range cache.assumedPods {
      ps, ok := cache.podStates[key]
      if !ok {
         klog.Fatal("Key found in assumed set but not in podStates. Potentially a logical error.")
      }
      if !ps.bindingFinished {
         klog.V(5).Infof("Couldn't expire cache for pod %v/%v. Binding is still in progress.",
            ps.pod.Namespace, ps.pod.Name)
         continue
      }
      if now.After(*ps.deadline) {
         klog.Warningf("Pod %s/%s expired", ps.pod.Namespace, ps.pod.Name)
         if err := cache.expirePod(key, ps); err != nil {
            klog.Errorf("ExpirePod failed for %s: %v", key, err)
         }
      }
   }
}

func (cache *schedulerCache) expirePod(key string, ps *podState) error {
   if err := cache.removePod(ps.pod); err != nil {
      return err
   }
   delete(cache.assumedPods, key)
   delete(cache.podStates, key)
   return nil
}
```

根据Cache中的值更新snapshot

```go
// UpdateSnapshot takes a snapshot of cached NodeInfo map. This is called at
// beginning of every scheduling cycle.
// The snapshot only includes Nodes that are not deleted at the time this function is called.
// nodeinfo.Node() is guaranteed to be not nil for all the nodes in the snapshot.
// This function tracks generation number of NodeInfo and updates only the
// entries of an existing snapshot that have changed after the snapshot was taken.
func (cache *schedulerCache) UpdateSnapshot(nodeSnapshot *Snapshot) error {
   cache.mu.Lock()
   defer cache.mu.Unlock()

   // Get the last generation of the snapshot.
   snapshotGeneration := nodeSnapshot.generation

   // NodeInfoList and HavePodsWithAffinityNodeInfoList must be re-created if a node was added
   // or removed from the cache.
   updateAllLists := false
   // HavePodsWithAffinityNodeInfoList must be re-created if a node changed its
   // status from having pods with affinity to NOT having pods with affinity or the other
   // way around.
   updateNodesHavePodsWithAffinity := false
   // HavePodsWithRequiredAntiAffinityNodeInfoList must be re-created if a node changed its
   // status from having pods with required anti-affinity to NOT having pods with required
   // anti-affinity or the other way around.
   updateNodesHavePodsWithRequiredAntiAffinity := false

   // Start from the head of the NodeInfo doubly linked list and update snapshot
   // of NodeInfos updated after the last snapshot.
   for node := cache.headNode; node != nil; node = node.next {
      if node.info.Generation <= snapshotGeneration {
         // all the nodes are updated before the existing snapshot. We are done.
         break
      }
      if np := node.info.Node(); np != nil {
         existing, ok := nodeSnapshot.nodeInfoMap[np.Name]
         if !ok {
            updateAllLists = true
            existing = &framework.NodeInfo{}
            nodeSnapshot.nodeInfoMap[np.Name] = existing
         }
         clone := node.info.Clone()
         // We track nodes that have pods with affinity, here we check if this node changed its
         // status from having pods with affinity to NOT having pods with affinity or the other
         // way around.
         if (len(existing.PodsWithAffinity) > 0) != (len(clone.PodsWithAffinity) > 0) {
            updateNodesHavePodsWithAffinity = true
         }
         if (len(existing.PodsWithRequiredAntiAffinity) > 0) != (len(clone.PodsWithRequiredAntiAffinity) > 0) {
            updateNodesHavePodsWithRequiredAntiAffinity = true
         }
         // We need to preserve the original pointer of the NodeInfo struct since it
         // is used in the NodeInfoList, which we may not update.
         *existing = *clone
      }
   }
   // Update the snapshot generation with the latest NodeInfo generation.
   if cache.headNode != nil {
      nodeSnapshot.generation = cache.headNode.info.Generation
   }

   // Comparing to pods in nodeTree.
   // Deleted nodes get removed from the tree, but they might remain in the nodes map
   // if they still have non-deleted Pods.
   if len(nodeSnapshot.nodeInfoMap) > cache.nodeTree.numNodes {
      cache.removeDeletedNodesFromSnapshot(nodeSnapshot)
      updateAllLists = true
   }

   if updateAllLists || updateNodesHavePodsWithAffinity || updateNodesHavePodsWithRequiredAntiAffinity {
      cache.updateNodeInfoSnapshotList(nodeSnapshot, updateAllLists)
   }

   if len(nodeSnapshot.nodeInfoList) != cache.nodeTree.numNodes {
      errMsg := fmt.Sprintf("snapshot state is not consistent, length of NodeInfoList=%v not equal to length of nodes in tree=%v "+
         ", length of NodeInfoMap=%v, length of nodes in cache=%v"+
         ", trying to recover",
         len(nodeSnapshot.nodeInfoList), cache.nodeTree.numNodes,
         len(nodeSnapshot.nodeInfoMap), len(cache.nodes))
      klog.Error(errMsg)
      // We will try to recover by re-creating the lists for the next scheduling cycle, but still return an
      // error to surface the problem, the error will likely cause a failure to the current scheduling cycle.
      cache.updateNodeInfoSnapshotList(nodeSnapshot, true)
      return fmt.Errorf(errMsg)
   }

   return nil
}

func (cache *schedulerCache) updateNodeInfoSnapshotList(snapshot *Snapshot, updateAll bool) {
   snapshot.havePodsWithAffinityNodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
   snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
   if updateAll {
      // Take a snapshot of the nodes order in the tree
      snapshot.nodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
      nodesList, err := cache.nodeTree.list()
      if err != nil {
         klog.Error(err)
      }
      for _, nodeName := range nodesList {
         if nodeInfo := snapshot.nodeInfoMap[nodeName]; nodeInfo != nil {
            snapshot.nodeInfoList = append(snapshot.nodeInfoList, nodeInfo)
            if len(nodeInfo.PodsWithAffinity) > 0 {
               snapshot.havePodsWithAffinityNodeInfoList = append(snapshot.havePodsWithAffinityNodeInfoList, nodeInfo)
            }
            if len(nodeInfo.PodsWithRequiredAntiAffinity) > 0 {
               snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = append(snapshot.havePodsWithRequiredAntiAffinityNodeInfoList, nodeInfo)
            }
         } else {
            klog.Errorf("node %q exist in nodeTree but not in NodeInfoMap, this should not happen.", nodeName)
         }
      }
   } else {
      for _, nodeInfo := range snapshot.nodeInfoList {
         if len(nodeInfo.PodsWithAffinity) > 0 {
            snapshot.havePodsWithAffinityNodeInfoList = append(snapshot.havePodsWithAffinityNodeInfoList, nodeInfo)
         }
         if len(nodeInfo.PodsWithRequiredAntiAffinity) > 0 {
            snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = append(snapshot.havePodsWithRequiredAntiAffinityNodeInfoList, nodeInfo)
         }
      }
   }
}

// If certain nodes were deleted after the last snapshot was taken, we should remove them from the snapshot.
func (cache *schedulerCache) removeDeletedNodesFromSnapshot(snapshot *Snapshot) {
   toDelete := len(snapshot.nodeInfoMap) - cache.nodeTree.numNodes
   for name := range snapshot.nodeInfoMap {
      if toDelete <= 0 {
         break
      }
      if n, ok := cache.nodes[name]; !ok || n.info.Node() == nil {
         delete(snapshot.nodeInfoMap, name)
         toDelete--
      }
   }
}
```