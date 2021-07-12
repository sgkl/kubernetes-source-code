# Scheduler

调度器监控集群中未被调度的pods，并尝试找到pod可以运行的nodes，并从中选出一个得分最高的，然后将绑定结果通过api server写入集群。

## Scheduler数据结构

Scheduler数据结构位于pkg/scheduler/scheduler.go文件中

```go
// Scheduler watches for new unscheduled pods. It attempts to find
// nodes that they fit on and writes bindings back to the api server.
type Scheduler struct {
   // It is expected that changes made via SchedulerCache will be observed
   // by NodeLister and Algorithm.
   SchedulerCache internalcache.Cache

   Algorithm ScheduleAlgorithm

   Extenders []framework.Extender

   // NextPod should be a function that blocks until the next pod
   // is available. We don't use a channel for this, because scheduling
   // a pod may take some amount of time and we don't want pods to get
   // stale while they sit in a channel.
   NextPod func() *framework.QueuedPodInfo

   // Error is called if there is an error. It is passed the pod in
   // question, and the error
   Error func(*framework.QueuedPodInfo, error)

   // Close this to shut down the scheduler.
   StopEverything <-chan struct{}

   // SchedulingQueue holds pods to be scheduled
   SchedulingQueue internalqueue.SchedulingQueue

   // Profiles are the scheduling profiles.
   Profiles profile.Map

   client clientset.Interface
}
```

