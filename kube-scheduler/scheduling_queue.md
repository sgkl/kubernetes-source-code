# 调度队列

调度队列中保存还没有被调度或之前有调度但是没能调度成功的pod。在调度器运行过程中，每一个周期从中取出一个pod进行调度操作。调度队列中的pod安装给定的笔记函数进行排序。

## 接口
```go
// SchedulingQueue is an interface for a queue to store pods waiting to be scheduled.
// The interface follows a pattern similar to cache.FIFO and cache.Heap and
// makes it easy to use those data structures as a SchedulingQueue.
type SchedulingQueue interface {
	framework.PodNominator
	Add(pod *v1.Pod) error
	// Activate moves the given pods to activeQ iff they're in unschedulableQ or backoffQ.
	// The passed-in pods are originally compiled from plugins that want to activate Pods,
	// by injecting the pods through a reserved CycleState struct (PodsToActivate).
	Activate(pods map[string]*v1.Pod)
	// AddUnschedulableIfNotPresent adds an unschedulable pod back to scheduling queue.
	// The podSchedulingCycle represents the current scheduling cycle number which can be
	// returned by calling SchedulingCycle().
	AddUnschedulableIfNotPresent(pod *framework.QueuedPodInfo, podSchedulingCycle int64) error
	// SchedulingCycle returns the current number of scheduling cycle which is
	// cached by scheduling queue. Normally, incrementing this number whenever
	// a pod is popped (e.g. called Pop()) is enough.
	SchedulingCycle() int64
	// Pop removes the head of the queue and returns it. It blocks if the
	// queue is empty and waits until a new item is added to the queue.
	Pop() (*framework.QueuedPodInfo, error)
	Update(oldPod, newPod *v1.Pod) error
	Delete(pod *v1.Pod) error
	MoveAllToActiveOrBackoffQueue(event framework.ClusterEvent, preCheck PreEnqueueCheck)
	AssignedPodAdded(pod *v1.Pod)
	AssignedPodUpdated(pod *v1.Pod)
	PendingPods() []*v1.Pod
	// Close closes the SchedulingQueue so that the goroutine which is
	// waiting to pop items can exit gracefully.
	Close()
	// NumUnschedulablePods returns the number of unschedulable pods exist in the SchedulingQueue.
	NumUnschedulablePods() int
	// Run starts the goroutines managing the queue.
	Run()
}

```

## nominator

### 数据结构

nominator维护pod被提名的node的信息

```go
// PodNominator abstracts operations to maintain nominated Pods.
type PodNominator interface {
	// AddNominatedPod adds the given pod to the nominator or
	// updates it if it already exists.
	AddNominatedPod(pod *PodInfo, nodeName string)
	// DeleteNominatedPodIfExists deletes nominatedPod from internal cache. It's a no-op if it doesn't exist.
	DeleteNominatedPodIfExists(pod *v1.Pod)
	// UpdateNominatedPod updates the <oldPod> with <newPod>.
	UpdateNominatedPod(oldPod *v1.Pod, newPodInfo *PodInfo)
	// NominatedPodsForNode returns nominatedPods on the given node.
	NominatedPodsForNode(nodeName string) []*PodInfo
}

// nominator is a structure that stores pods nominated to run on nodes.
// It exists because nominatedNodeName of pod objects stored in the structure
// may be different than what scheduler has here. We should be able to find pods
// by their UID and update/delete them.
type nominator struct {
	// podLister is used to verify if the given pod is alive.
	podLister listersv1.PodLister
	// nominatedPods is a map keyed by a node name and the value is a list of
	// pods which are nominated to run on the node. These are pods which can be in
	// the activeQ or unschedulableQ.
	nominatedPods map[string][]*framework.PodInfo    // 保存提名在node上运行的pod
	// nominatedPodToNode is map keyed by a Pod UID to the node name where it is
	// nominated.
	nominatedPodToNode map[types.UID]string          // 通过pod名查找提名的node

	sync.RWMutex
}
```

构造nominator时需要传入PodLister作为参数

```go
// NewPodNominator creates a nominator as a backing of framework.PodNominator.
// A podLister is passed in so as to check if the pod exists
// before adding its nominatedNode info.
func NewPodNominator(podLister listersv1.PodLister) framework.PodNominator {
   return &nominator{
      podLister:          podLister,
      nominatedPods:      make(map[string][]*framework.PodInfo),
      nominatedPodToNode: make(map[types.UID]string),
   }
}
```

### 方法

#### add

`add(pi *framework.PodInfo, nodeName string)`向nomiator添加pod和提名node的信息

```go
func (npm *nominator) add(pi *framework.PodInfo, nodeName string) {
   // always delete the pod if it already exist, to ensure we never store more than
   // one instance of the pod.
   npm.delete(pi.Pod)

   nnn := nodeName
   if len(nnn) == 0 {
      nnn = NominatedNodeName(pi.Pod)
      if len(nnn) == 0 {
         return
      }
   }

   if npm.podLister != nil {
      // If the pod is not alive, don't contain it.
      if _, err := npm.podLister.Pods(pi.Pod.Namespace).Get(pi.Pod.Name); err != nil {
         klog.V(4).InfoS("Pod doesn't exist in podLister, aborting adding it to the nominator", "pod", klog.KObj(pi.Pod))
         return
      }
   }

   npm.nominatedPodToNode[pi.Pod.UID] = nnn
   for _, npi := range npm.nominatedPods[nnn] {
      if npi.Pod.UID == pi.Pod.UID {
         klog.V(4).InfoS("Pod already exists in the nominator", "pod", klog.KObj(npi.Pod))
         return
      }
   }
   npm.nominatedPods[nnn] = append(npm.nominatedPods[nnn], pi)
}
```

​	`delete(p *v1.Pod)`从nominator中删除pod的相关信息

```go
func (npm *nominator) delete(p *v1.Pod) {
   nnn, ok := npm.nominatedPodToNode[p.UID]
   if !ok {
      return
   }
   for i, np := range npm.nominatedPods[nnn] {
      if np.Pod.UID == p.UID {
         npm.nominatedPods[nnn] = append(npm.nominatedPods[nnn][:i], npm.nominatedPods[nnn][i+1:]...)
         if len(npm.nominatedPods[nnn]) == 0 {
            delete(npm.nominatedPods, nnn)
         }
         break
      }
   }
   delete(npm.nominatedPodToNode, p.UID)
}
```

`UpdateNominatedPod(oldPod *v1.Pod, newPodInfo *framwork.PodInfo)`用newPod更新nominator中oldPod相关的信息

```go
// UpdateNominatedPod updates the <oldPod> with <newPod>.
func (npm *nominator) UpdateNominatedPod(oldPod *v1.Pod, newPodInfo *framework.PodInfo) {
   npm.Lock()
   defer npm.Unlock()
   // In some cases, an Update event with no "NominatedNode" present is received right
   // after a node("NominatedNode") is reserved for this pod in memory.
   // In this case, we need to keep reserving the NominatedNode when updating the pod pointer.
   nodeName := ""
   // We won't fall into below `if` block if the Update event represents:
   // (1) NominatedNode info is added
   // (2) NominatedNode info is updated
   // (3) NominatedNode info is removed
   if NominatedNodeName(oldPod) == "" && NominatedNodeName(newPodInfo.Pod) == "" {
      if nnn, ok := npm.nominatedPodToNode[oldPod.UID]; ok {
         // This is the only case we should continue reserving the NominatedNode
         nodeName = nnn
      }
   }
   // We update irrespective of the nominatedNodeName changed or not, to ensure
   // that pod pointer is updated.
   npm.delete(oldPod)
   npm.add(newPodInfo, nodeName)
}
```

`DeleteNominatedPodIfExists(pod *v1.Pod)`从nominator中删除pod相关信息

```go
// DeleteNominatedPodIfExists deletes <pod> from nominatedPods.
func (npm *nominator) DeleteNominatedPodIfExists(pod *v1.Pod) {
   npm.Lock()
   npm.delete(pod)    // 调用delete删除
   npm.Unlock()
}
```

`AddNominatedPod(pi *framework.PodInfo, nodeName string)` 向nominator中添加pod和提名node的信息

```go
// AddNominatedPod adds a pod to the nominated pods of the given node.
// This is called during the preemption process after a node is nominated to run
// the pod. We update the structure before sending a request to update the pod
// object to avoid races with the following scheduling cycles.
func (npm *nominator) AddNominatedPod(pi *framework.PodInfo, nodeName string) {
   npm.Lock()
   npm.add(pi, nodeName)    // 调用add删除
   npm.Unlock()
}
```

`NominatedPodsForNode(nodeName string) []*framework.PodInfo` 通过node的名称查找提名在此node上运行的所有pod

```go
// NominatedPodsForNode returns pods that are nominated to run on the given node,
// but they are waiting for other pods to be removed from the node.
func (npm *nominator) NominatedPodsForNode(nodeName string) []*framework.PodInfo {
	npm.RLock()
	defer npm.RUnlock()
	// TODO: we may need to return a copy of []*Pods to avoid modification
	// on the caller side.
  // 言外之意不要修改此处返回的值
	return npm.nominatedPods[nodeName]
}
```



## UnschedulablePodsMap

UnschedulablePodsMap保存所有不能被调度的pod

### 数据结构

```go
// UnschedulablePodsMap holds pods that cannot be scheduled. This data structure
// is used to implement unschedulableQ.
type UnschedulablePodsMap struct {
   // podInfoMap is a map key by a pod's full-name and the value is a pointer to the QueuedPodInfo.
   podInfoMap map[string]*framework.QueuedPodInfo    // key到QueuedPodInfo的映射
   keyFunc    func(*v1.Pod) string                   // 通过此函数获取podInfoMap中的key值
   // metricRecorder updates the counter when elements of an unschedulablePodsMap
   // get added or removed, and it does nothing if it's nil
   metricRecorder metrics.MetricRecorder             // metric相关
}
```

构造函数

```go
// newUnschedulablePodsMap initializes a new object of UnschedulablePodsMap.
func newUnschedulablePodsMap(metricRecorder metrics.MetricRecorder) *UnschedulablePodsMap {
   return &UnschedulablePodsMap{
      podInfoMap:     make(map[string]*framework.QueuedPodInfo),
      keyFunc:        util.GetPodFullName,
      metricRecorder: metricRecorder,
   }
}
```

### 方法

`addOrUpdate(pInfo *framework.QueuedPodInfo)` 将一个pod添加或者更新到队列中

```go
// Add adds a pod to the unschedulable podInfoMap.
func (u *UnschedulablePodsMap) addOrUpdate(pInfo *framework.QueuedPodInfo) {
   podID := u.keyFunc(pInfo.Pod)
   if _, exists := u.podInfoMap[podID]; !exists && u.metricRecorder != nil {
      u.metricRecorder.Inc()
   }
   u.podInfoMap[podID] = pInfo
}
```

`delete(pod *v1.Pod)` 从不能调度队列中删除pod

```go
// Delete deletes a pod from the unschedulable podInfoMap.
func (u *UnschedulablePodsMap) delete(pod *v1.Pod) {
   podID := u.keyFunc(pod)
   if _, exists := u.podInfoMap[podID]; exists && u.metricRecorder != nil {
      u.metricRecorder.Dec()
   }
   delete(u.podInfoMap, podID)
}
```

`get(pod *v1.Pod) *framework.QueuedPodInfo` 从队列中获取pod相对应的QueuedPodInfo

```go
// Get returns the QueuedPodInfo if a pod with the same key as the key of the given "pod"
// is found in the map. It returns nil otherwise.
func (u *UnschedulablePodsMap) get(pod *v1.Pod) *framework.QueuedPodInfo {
   podKey := u.keyFunc(pod)
   if pInfo, exists := u.podInfoMap[podKey]; exists {
      return pInfo
   }
   return nil
}
```

`clear()`清除队列

```go
// Clear removes all the entries from the unschedulable podInfoMap.
func (u *UnschedulablePodsMap) clear() {
   u.podInfoMap = make(map[string]*framework.QueuedPodInfo)
   if u.metricRecorder != nil {
      u.metricRecorder.Clear()
   }
}
```

## PriorityQueue

PriorityQueue是SchedulingQueue的一个实现.

PriorityQueue每次返回优先级最高的Pod作为下一个被调度的Pod。PriorityQueue中有三个队列：activeQ、PodBackoffQ和unschedulableQ。

* activeQ：保存后续被用来调度的pod
* PodBackoffQ：
* unschedulerQ：保存已经被调度过，但是被标记unschedulable的pod

### 数据结构

```go
// PriorityQueue implements a scheduling queue.
// The head of PriorityQueue is the highest priority pending pod. This structure
// has three sub queues. One sub-queue holds pods that are being considered for
// scheduling. This is called activeQ and is a Heap. Another queue holds
// pods that are already tried and are determined to be unschedulable. The latter
// is called unschedulableQ. The third queue holds pods that are moved from
// unschedulable queues and will be moved to active queue when backoff are completed.
type PriorityQueue struct {
	// PodNominator abstracts the operations to maintain nominated Pods.
	framework.PodNominator    // 见nominator的定义

	stop  chan struct{}       // 告诉停止routine执行的chan
	clock util.Clock

	// pod initial backoff duration.
	podInitialBackoffDuration time.Duration
	// pod maximum backoff duration.
	podMaxBackoffDuration time.Duration

	lock sync.RWMutex
	cond sync.Cond

	// activeQ is heap structure that scheduler actively looks at to find pods to
	// schedule. Head of heap is the highest priority pod.
	activeQ *heap.Heap
	// podBackoffQ is a heap ordered by backoff expiry. Pods which have completed backoff
	// are popped from this heap before the scheduler looks at activeQ
	podBackoffQ *heap.Heap
	// unschedulableQ holds pods that have been tried and determined unschedulable.
	unschedulableQ *UnschedulablePodsMap      // 见上面UnschedulablePodsMap
	// schedulingCycle represents sequence number of scheduling cycle and is incremented
	// when a pod is popped.
	schedulingCycle int64                    
	// moveRequestCycle caches the sequence number of scheduling cycle when we
	// received a move request. Unschedulable pods in and before this scheduling
	// cycle will be put back to activeQueue if we were trying to schedule them
	// when we received move request.
	moveRequestCycle int64

	clusterEventMap map[framework.ClusterEvent]sets.String

	// closed indicates that the queue is closed.
	// It is mainly used to let Pop() exit its control loop while waiting for an item.
	closed bool

	nsLister listersv1.NamespaceLister
}
```

创建PriorityQueue需要的Options：

```go
type priorityQueueOptions struct {
	clock                     util.Clock
	podInitialBackoffDuration time.Duration       // 第一次调度等待的时间
	podMaxBackoffDuration     time.Duration       // 调度等待的最大时间
	podNominator              framework.PodNominator
	clusterEventMap           map[framework.ClusterEvent]sets.String
}
```

构造函数

```go
// NewPriorityQueue creates a PriorityQueue object.
func NewPriorityQueue(
	lessFn framework.LessFunc,
	informerFactory informers.SharedInformerFactory,
	opts ...Option,
) *PriorityQueue {
	options := defaultPriorityQueueOptions    // 默认options
	for _, opt := range opts {                // 通过传入的Option设置options
		opt(&options)
	}

	comp := func(podInfo1, podInfo2 interface{}) bool {   // 队列中元素排序的比较函数
		pInfo1 := podInfo1.(*framework.QueuedPodInfo)
		pInfo2 := podInfo2.(*framework.QueuedPodInfo)
		return lessFn(pInfo1, pInfo2)
	}

	if options.podNominator == nil {   // 如果podNominator还未赋值则调用NewPodNominator创建
		options.podNominator = NewPodNominator(informerFactory.Core().V1().Pods().Lister())
	}

	pq := &PriorityQueue{
		PodNominator:              options.podNominator,
		clock:                     options.clock,
		stop:                      make(chan struct{}),
		podInitialBackoffDuration: options.podInitialBackoffDuration,
		podMaxBackoffDuration:     options.podMaxBackoffDuration,
		activeQ:                   heap.NewWithRecorder(podInfoKeyFunc, comp, metrics.NewActivePodsRecorder()),   // 具体的数据结构是Heap，最大堆，通过传入的comp对比
		unschedulableQ:            newUnschedulablePodsMap(metrics.NewUnschedulablePodsRecorder()),
		moveRequestCycle:          -1,
		clusterEventMap:           options.clusterEventMap,
	}
	pq.cond.L = &pq.lock
	pq.podBackoffQ = heap.NewWithRecorder(podInfoKeyFunc, pq.podsCompareBackoffCompleted, metrics.NewBackoffPodsRecorder()) // 具体的数据结构是Heap，最大堆，比较的是backoff的时间，时间早的靠前
	if utilfeature.DefaultFeatureGate.Enabled(features.PodAffinityNamespaceSelector) {
		pq.nsLister = informerFactory.Core().V1().Namespaces().Lister()
	}

	return pq
}
```

### 方法

`Run()` 启动线程进行处理：

1. 如果backoff对列中的pod的backoff时间以到，则将pod从backoff队列中删除，并添加到active对列中
2. 检查未调度队列中的pod，如果pod在未调度队列中的时间大于unschedulableQTimeInterval，则将其移动到active或backoff队列中

```go
// Run starts the goroutine to pump from podBackoffQ to activeQ
func (p *PriorityQueue) Run() {
	go wait.Until(p.flushBackoffQCompleted, 1.0*time.Second, p.stop)
	go wait.Until(p.flushUnschedulableQLeftover, 30*time.Second, p.stop)
}

// flushBackoffQCompleted Moves all pods from backoffQ which have completed backoff in to activeQ
func (p *PriorityQueue) flushBackoffQCompleted() {
	p.lock.Lock()
	defer p.lock.Unlock()
	for {
		rawPodInfo := p.podBackoffQ.Peek()
		if rawPodInfo == nil {
			return
		}
		pod := rawPodInfo.(*framework.QueuedPodInfo).Pod
		boTime := p.getBackoffTime(rawPodInfo.(*framework.QueuedPodInfo))
		if boTime.After(p.clock.Now()) {
			return // backoff队列中的pod是按照backoff time排序的，如果第一个pod backoff time未到，则后面的pod的backoff time都未到
		}
		_, err := p.podBackoffQ.Pop()
		if err != nil {
			klog.ErrorS(err, "Unable to pop pod from backoff queue despite backoff completion", "pod", klog.KObj(pod))
			return
		}
		p.activeQ.Add(rawPodInfo)
		metrics.SchedulerQueueIncomingPods.WithLabelValues("active", BackoffComplete).Inc()
		defer p.cond.Broadcast()   // 广播， 等待active队列的函数收到广播后会返回
	}
}

// flushUnschedulableQLeftover moves pod which stays in unschedulableQ longer than the unschedulableQTimeInterval
// to activeQ.
func (p *PriorityQueue) flushUnschedulableQLeftover() {
	p.lock.Lock()
	defer p.lock.Unlock()

	var podsToMove []*framework.QueuedPodInfo
	currentTime := p.clock.Now()
	for _, pInfo := range p.unschedulableQ.podInfoMap {
		lastScheduleTime := pInfo.Timestamp
		if currentTime.Sub(lastScheduleTime) > unschedulableQTimeInterval {
			podsToMove = append(podsToMove, pInfo)
		}
	}

	if len(podsToMove) > 0 {
		p.movePodsToActiveOrBackoffQueue(podsToMove, UnschedulableTimeout)
	}
}

// NOTE: this function assumes lock has been acquired in caller
func (p *PriorityQueue) movePodsToActiveOrBackoffQueue(podInfoList []*framework.QueuedPodInfo, event framework.ClusterEvent) {
	moved := false
	for _, pInfo := range podInfoList {
		// If the event doesn't help making the Pod schedulable, continue.
		// Note: we don't run the check if pInfo.UnschedulablePlugins is nil, which denotes
		// either there is some abnormal error, or scheduling the pod failed by plugins other than PreFilter, Filter and Permit.
		// In that case, it's desired to move it anyways.
		if len(pInfo.UnschedulablePlugins) != 0 && !p.podMatchesEvent(pInfo, event) {
      // 上面的判断条件可以看出只有plugin发出UnschedulableTimeout时才二次调度
			continue
		}
		moved = true
		pod := pInfo.Pod
		if p.isPodBackingoff(pInfo) {   // 需要backoff时，需要搞清楚pod的backoff是在哪设置的
			if err := p.podBackoffQ.Add(pInfo); err != nil {
				klog.ErrorS(err, "Error adding pod to the backoff queue", "pod", klog.KObj(pod))
			} else {
				metrics.SchedulerQueueIncomingPods.WithLabelValues("backoff", event.Label).Inc()
				p.unschedulableQ.delete(pod)
			}
		} else {
			if err := p.activeQ.Add(pInfo); err != nil {
				klog.ErrorS(err, "Error adding pod to the scheduling queue", "pod", klog.KObj(pod))
			} else {
				metrics.SchedulerQueueIncomingPods.WithLabelValues("active", event.Label).Inc()
				p.unschedulableQ.delete(pod)
			}
		}
	}
	p.moveRequestCycle = p.schedulingCycle
	if moved {
		p.cond.Broadcast()
	}
}
```

`Add(pod *v1.Pod) error` 将一个pod添加到active队列中。只在有新pod时才调用此函数

```go
// Add adds a pod to the active queue. It should be called only when a new pod
// is added so there is no chance the pod is already in active/unschedulable/backoff queues
func (p *PriorityQueue) Add(pod *v1.Pod) error {
   p.lock.Lock()
   defer p.lock.Unlock()
   pInfo := p.newQueuedPodInfo(pod)
   if err := p.activeQ.Add(pInfo); err != nil {
      klog.ErrorS(err, "Error adding pod to the scheduling queue", "pod", klog.KObj(pod))
      return err
   }
   if p.unschedulableQ.get(pod) != nil {
      klog.ErrorS(nil, "Error: pod is already in the unschedulable queue", "pod", klog.KObj(pod))
      p.unschedulableQ.delete(pod)
   }
   // Delete pod from backoffQ if it is backing off
   if err := p.podBackoffQ.Delete(pInfo); err == nil {
      klog.ErrorS(nil, "Error: pod is already in the podBackoff queue", "pod", klog.KObj(pod))
   }
   metrics.SchedulerQueueIncomingPods.WithLabelValues("active", PodAdd).Inc()
   p.PodNominator.AddNominatedPod(pInfo.PodInfo, "") // 如果pInfo.PodInfo中无提名node，则不会保存到PodNominator中
   p.cond.Broadcast()

   return nil
}
```

`Active(pods map[string]*v1.Pod)` 将一个pod从

```go
// Activate moves the given pods to activeQ iff they're in unschedulableQ or backoffQ.
func (p *PriorityQueue) Activate(pods map[string]*v1.Pod) {
   p.lock.Lock()
   defer p.lock.Unlock()

   activated := false
   for _, pod := range pods {
      if p.activate(pod) {
         activated = true
      }
   }

   if activated {
      p.cond.Broadcast()
   }
}
func (p *PriorityQueue) activate(pod *v1.Pod) bool {
	// Verify if the pod is present in activeQ.
	if _, exists, _ := p.activeQ.Get(newQueuedPodInfoForLookup(pod)); exists {
		// No need to activate if it's already present in activeQ.
		return false
	}
	var pInfo *framework.QueuedPodInfo
	// Verify if the pod is present in unschedulableQ or backoffQ.
	if pInfo = p.unschedulableQ.get(pod); pInfo == nil {
		// If the pod doesn't belong to unschedulableQ or backoffQ, don't activate it.
		if obj, exists, _ := p.podBackoffQ.Get(newQueuedPodInfoForLookup(pod)); !exists {
			klog.ErrorS(nil, "To-activate pod does not exist in unschedulableQ or backoffQ", "pod", klog.KObj(pod))
			return false
		} else {
			pInfo = obj.(*framework.QueuedPodInfo)
		}
	}

	if pInfo == nil {
		// Redundant safe check. We shouldn't reach here.
		klog.ErrorS(nil, "Internal error: cannot obtain pInfo")
		return false
	}

	if err := p.activeQ.Add(pInfo); err != nil {
		klog.ErrorS(err, "Error adding pod to the scheduling queue", "pod", klog.KObj(pod))
		return false
	}
	p.unschedulableQ.delete(pod)
	p.podBackoffQ.Delete(pInfo)
	metrics.SchedulerQueueIncomingPods.WithLabelValues("active", ForceActivate).Inc()
	p.PodNominator.AddNominatedPod(pInfo.PodInfo, "") // 如果pInfo.PodInfo中无提名node，则不会保存到PodNominator中
	return true
}
```

`SchedulingCycle() int64` 返回scheduling cycle

```go
// SchedulingCycle returns current scheduling cycle.
func (p *PriorityQueue) SchedulingCycle() int64 {
   p.lock.RLock()
   defer p.lock.RUnlock()
   return p.schedulingCycle
}
```

`AddUnschedulableIfNotPresent(pInfo *framework.QueuePodInfo, podSchedulingCycle int64) error`对于一个不能被调度的pod，如果它当前不在unschedulableQ则将其添加进去

```go
// AddUnschedulableIfNotPresent inserts a pod that cannot be scheduled into
// the queue, unless it is already in the queue. Normally, PriorityQueue puts
// unschedulable pods in `unschedulableQ`. But if there has been a recent move
// request, then the pod is put in `podBackoffQ`.
func (p *PriorityQueue) AddUnschedulableIfNotPresent(pInfo *framework.QueuedPodInfo, podSchedulingCycle int64) error {
   p.lock.Lock()
   defer p.lock.Unlock()
   pod := pInfo.Pod
   if p.unschedulableQ.get(pod) != nil {
      return fmt.Errorf("Pod %v is already present in unschedulable queue", klog.KObj(pod))
   }

   if _, exists, _ := p.activeQ.Get(pInfo); exists {
      return fmt.Errorf("Pod %v is already present in the active queue", klog.KObj(pod))
   }
   if _, exists, _ := p.podBackoffQ.Get(pInfo); exists {
      return fmt.Errorf("Pod %v is already present in the backoff queue", klog.KObj(pod))
   }

   // Refresh the timestamp since the pod is re-added.
   pInfo.Timestamp = p.clock.Now()

   // If a move request has been received, move it to the BackoffQ, otherwise move
   // it to unschedulableQ.
   if p.moveRequestCycle >= podSchedulingCycle {
      if err := p.podBackoffQ.Add(pInfo); err != nil {
         return fmt.Errorf("error adding pod %v to the backoff queue: %v", pod.Name, err)
      }
      metrics.SchedulerQueueIncomingPods.WithLabelValues("backoff", ScheduleAttemptFailure).Inc()
   } else {
      p.unschedulableQ.addOrUpdate(pInfo)
      metrics.SchedulerQueueIncomingPods.WithLabelValues("unschedulable", ScheduleAttemptFailure).Inc()
   }

   p.PodNominator.AddNominatedPod(pInfo.PodInfo, "")
   return nil
}
```

#### NumUnschedulablePods函数

`NumUnschedulablePods() int` 返回Unschedulable pod的数量

```go
// NumUnschedulablePods returns the number of unschedulable pods exist in the SchedulingQueue.
func (p *PriorityQueue) NumUnschedulablePods() int {
   p.lock.RLock()
   defer p.lock.RUnlock()
   return len(p.unschedulableQ.podInfoMap)
}
```

#### Pod函数

`Pop() (*framework.QueuePodInfo, error)` 返回active队列中的第一个pod，调度器每次通过此函数获取一个pod，然后执行调度框架的Plugin。这个函数是此队列的消费者。

```go
// Pop removes the head of the active queue and returns it. It blocks if the
// activeQ is empty and waits until a new item is added to the queue. It
// increments scheduling cycle when a pod is popped.
func (p *PriorityQueue) Pop() (*framework.QueuedPodInfo, error) {
   p.lock.Lock()
   defer p.lock.Unlock()
   for p.activeQ.Len() == 0 {
      // When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
      // When Close() is called, the p.closed is set and the condition is broadcast,
      // which causes this loop to continue and return from the Pop().
      if p.closed {
         return nil, fmt.Errorf(queueClosed)
      }
      p.cond.Wait()
   }
   obj, err := p.activeQ.Pop()
   if err != nil {
      return nil, err
   }
   pInfo := obj.(*framework.QueuedPodInfo)
   pInfo.Attempts++
   p.schedulingCycle++
   return pInfo, err
}
```

#### Update函数

`Update(oldPod, newPod *v1.Pod) error` 其功能如下

* 如果pod在active队列或backoff队列中，则对其进行更新

* 如果pod在unschedulable队列中
  + 如果pod未被更新（需要的值），则直接在unschedulable队列中添加或更新pod
  + 如果pod被更新（需要的值），并且backoff时间未到，则从unschedulable队列中删除pod，并将其添加到backoff对列中
  + 如果pod被更新（需要的值），并且backoff时间已到，则从unschedulable队列中删除pod，并将其添加到active对立中
*  如果pod在三个队列中都不存在，则直接将其添加到active队列中。

```go
// Update updates a pod in the active or backoff queue if present. Otherwise, it removes
// the item from the unschedulable queue if pod is updated in a way that it may
// become schedulable and adds the updated one to the active queue.
// If pod is not present in any of the queues, it is added to the active queue.
func (p *PriorityQueue) Update(oldPod, newPod *v1.Pod) error {
   p.lock.Lock()
   defer p.lock.Unlock()

   if oldPod != nil {
      oldPodInfo := newQueuedPodInfoForLookup(oldPod)
      // If the pod is already in the active queue, just update it there.
      if oldPodInfo, exists, _ := p.activeQ.Get(oldPodInfo); exists {
         pInfo := updatePod(oldPodInfo, newPod)
         p.PodNominator.UpdateNominatedPod(oldPod, pInfo.PodInfo)
         return p.activeQ.Update(pInfo)
      }

      // If the pod is in the backoff queue, update it there.
      if oldPodInfo, exists, _ := p.podBackoffQ.Get(oldPodInfo); exists {
         pInfo := updatePod(oldPodInfo, newPod)
         p.PodNominator.UpdateNominatedPod(oldPod, pInfo.PodInfo)
         return p.podBackoffQ.Update(pInfo)
      }
   }
  
   // If the pod is in the unschedulable queue, updating it may make it schedulable.
   // 这个地方需注意，因为unschedulableQ创建时传入的keyFunc时util.GetPodFullName，而pod的name和namespace时不能修改的，所以下面p.unschedulableQ.get(newPod)获取的和p.unschedulabelQ.get(oldPod)效果是一样的
   if usPodInfo := p.unschedulableQ.get(newPod); usPodInfo != nil {
      pInfo := updatePod(usPodInfo, newPod)
      p.PodNominator.UpdateNominatedPod(oldPod, pInfo.PodInfo)
      if isPodUpdated(oldPod, newPod) {
         if p.isPodBackingoff(usPodInfo) {
            if err := p.podBackoffQ.Add(pInfo); err != nil {
               return err
            }
            p.unschedulableQ.delete(usPodInfo.Pod)
         } else {
            if err := p.activeQ.Add(pInfo); err != nil {
               return err
            }
            p.unschedulableQ.delete(usPodInfo.Pod)
            p.cond.Broadcast()
         }
      } else {
         // Pod update didn't make it schedulable, keep it in the unschedulable queue.
         p.unschedulableQ.addOrUpdate(pInfo)
      }

      return nil
   }
   // If pod is not in any of the queues, we put it in the active queue.
   pInfo := p.newQueuedPodInfo(newPod)
   if err := p.activeQ.Add(pInfo); err != nil {
      return err
   }
   p.PodNominator.AddNominatedPod(pInfo.PodInfo, "")
   p.cond.Broadcast()
   return nil
}
```

#### Delete函数

`Delete(pod *v1.Pod) error` 从队列中删除pod

```go
// Delete deletes the item from either of the two queues. It assumes the pod is
// only in one queue.
func (p *PriorityQueue) Delete(pod *v1.Pod) error {
   p.lock.Lock()
   defer p.lock.Unlock()
   p.PodNominator.DeleteNominatedPodIfExists(pod)
   if err := p.activeQ.Delete(newQueuedPodInfoForLookup(pod)); err != nil {
      // The item was probably not found in the activeQ.
      p.podBackoffQ.Delete(newQueuedPodInfoForLookup(pod))
      p.unschedulableQ.delete(pod)
   }
   return nil
}
```

#### AssignedPodAdded

`AssignedPodAdded(pod *v1.pod)`获取unschedulabel队列中所有和pod具体亲和性的pod，分情况将其移动到active队列或backoff队列中。

有两点没搞清楚：

* 此函数具体在什么情况下被调用？
* ClusterEvent AssignedPodAdd是做什么用的？

```go
// AssignedPodAdded is called when a bound pod is added. Creation of this pod
// may make pending pods with matching affinity terms schedulable.
func (p *PriorityQueue) AssignedPodAdded(pod *v1.Pod) {
	p.lock.Lock()
	p.movePodsToActiveOrBackoffQueue(p.getUnschedulablePodsWithMatchingAffinityTerm(pod), AssignedPodAdd)
	p.lock.Unlock()
}
```

#### AssignedPodUpdated

`AssignedPodUpdated(pod *v1.Pod)`获取unschedulabel队列中所有和pod具体亲和性的pod，分情况将其移动到active队列或backoff队列中。注意和AssignedPodAdded的区别是AssignedPodAdded匹配的ClusterEvent是AssignedPodAdd，而这个函数匹配的ClusterEvent是AssignedPodUpdate。

```go
// AssignedPodUpdated is called when a bound pod is updated. Change of labels
// may make pending pods with matching affinity terms schedulable.
func (p *PriorityQueue) AssignedPodUpdated(pod *v1.Pod) {
	p.lock.Lock()
	p.movePodsToActiveOrBackoffQueue(p.getUnschedulablePodsWithMatchingAffinityTerm(pod), AssignedPodUpdate)
	p.lock.Unlock()
}
```

#### MoveAllToActiveOrBackoffQueue

`MoveAllToActiveOrBackoffQueue(event framework.ClusterEvent, preCheck PreEnqueueCheck)`将unschedulable队列中所有通过preCheck检查的pod移动到active队列或backoff队列中

```go
// MoveAllToActiveOrBackoffQueue moves all pods from unschedulableQ to activeQ or backoffQ.
// This function adds all pods and then signals the condition variable to ensure that
// if Pop() is waiting for an item, it receives it after all the pods are in the
// queue and the head is the highest priority pod.
func (p *PriorityQueue) MoveAllToActiveOrBackoffQueue(event framework.ClusterEvent, preCheck PreEnqueueCheck) {
	p.lock.Lock()
	defer p.lock.Unlock()
	unschedulablePods := make([]*framework.QueuedPodInfo, 0, len(p.unschedulableQ.podInfoMap))
	for _, pInfo := range p.unschedulableQ.podInfoMap {
		if preCheck == nil || preCheck(pInfo.Pod) {
			unschedulablePods = append(unschedulablePods, pInfo)
		}
	}
	p.movePodsToActiveOrBackoffQueue(unschedulablePods, event)
}
```

#### PendingPods函数

`PendingPods() []*v1.Pod`返回队列中所有的pod。debug用

```go
// PendingPods returns all the pending pods in the queue. This function is
// used for debugging purposes in the scheduler cache dumper and comparer.
func (p *PriorityQueue) PendingPods() []*v1.Pod {
	p.lock.RLock()
	defer p.lock.RUnlock()
	var result []*v1.Pod
	for _, pInfo := range p.activeQ.List() {
		result = append(result, pInfo.(*framework.QueuedPodInfo).Pod)
	}
	for _, pInfo := range p.podBackoffQ.List() {
		result = append(result, pInfo.(*framework.QueuedPodInfo).Pod)
	}
	for _, pInfo := range p.unschedulableQ.podInfoMap {
		result = append(result, pInfo.Pod)
	}
	return result
}
```

#### Close函数

`Close()`终止PriorityQueue的执行

```go
// Close closes the priority queue.
func (p *PriorityQueue) Close() {
	p.lock.Lock()
	defer p.lock.Unlock()
	close(p.stop)
	p.closed = true
	p.cond.Broadcast()
}
```



#### getBackoffTime

`getBackoffTime(podInfo *framework.QueuedPodInfo) time.Time` 返回pod backoff的时间，我的理解是此pod排队等待的时间，也即下此被尝试调度的时间，如果当前时间早于此时间此pod不会被尝试调度 。当当前时间大于此时间时才回尝试调度此pod

```go
// getBackoffTime returns the time that podInfo completes backoff
func (p *PriorityQueue) getBackoffTime(podInfo *framework.QueuedPodInfo) time.Time {
   duration := p.calculateBackoffDuration(podInfo)
   backoffTime := podInfo.Timestamp.Add(duration)
   return backoffTime
}

// calculateBackoffDuration is a helper function for calculating the backoffDuration
// based on the number of attempts the pod has made.
func (p *PriorityQueue) calculateBackoffDuration(podInfo *framework.QueuedPodInfo) time.Duration {
   duration := p.podInitialBackoffDuration
   for i := 1; i < podInfo.Attempts; i++ {
      duration = duration * 2
      if duration > p.podMaxBackoffDuration {
         return p.podMaxBackoffDuration
      }
   }
   return duration
}
```

