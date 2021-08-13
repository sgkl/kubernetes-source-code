## 核心功能模块

### PLEG

PLEG全称PodLifecycleEvent，PLEG会一直调用container runtime获取本节点的pods，之后与本模块之前缓存的pods信息作比较，获取最新的pods中的容器状态是否发生改变。当状态发生切换的时候，生成一个eventRecord事件，输出到eventChannel中。syncPod模块会接收eventChannel中的event事件，来触发pod同步处理过程，调用container runtime来重建pod，保证pod工作正常。

### cAdvisor

cAdvisor集成在kubelet中，收集本Node节点和运行容器的监控信息。cAdvisor启动一个Http Server服务器，对外接收rest api请求。cAdvisor模块对外提供了interface接口，外部可以通过interface接口获取到node节点信息、本地文件系统的状态等信息。该接口被imageManager、OOMWatcher、containerMananger等所使用。

### GPUManager

对于Node上可使用的GPU的管理，当前版本需要在kubelet启动参数中指定feature-gates中添加Accelerators=true，并且需要采用runtime=Docker的情况下才能支持使用GPU，并且当前只支持NvidiaGPU，GPUManager主要实现interface定义的Start()/Capacity()/AllocategPU()三个函数

### OOMWatcher

系统OOM的监听器，将会与cAdvisor模块之间建立SystemOOM，通过watch方式从cAdvisor那里收到的OOM信息，并发生相关事件

### ProbeManager

探针管理。依赖statusManager、livenessManager、containerRefManager，实现pod的健康检查的功能。当前支持两种类型的探针：LivenessProbe和ReadinessProbe

* LivenessProbe: 用于判断容器是否存活，如果探测到容器不健康，怎kubelet将杀掉该容器，并根据容器的重启策略做相应的处理
* ReadinessProbe: 用于判断容器是否启动完成

探针有三种实现方式

* exec probe: 在容器内部执行一个命令，如果命令返回码为0，则表明容器健康
* tcp probe: 通过容器的IP地址和端口号执行tcp检查，如果能够建立tcp连接，则表明容器健康
* http probe: 通过容器的IP地址，端口号以及路径调用http Get方法，如果响应status >= 200 && status <= 400的时候，则认为容器状态健康

### StatusManager

该模块负责pod里面的容器状态，接受从其它模块发送过来的pod状态改变的时间，进行处理，并更新到kube-apiserver中

### Container/RefManager

容器引用的管理，相对简单的Manager，通过定义map来实现containerID与v1.ObjectReference容器引用的映射

### EvictionManager

evictionManager当node的节点资源不足的时候，达到了配置的驱逐策略，将会从node上驱赶pod，来保证node节点的稳定性。可以通过kubelet启动参数来决定驱逐策略。另外当node的内存以及disk资源达到驱逐策略的时候会生成对应node状态记录

### ImageGC

imageGC负责node节点的镜像回收，当本地存放镜像的磁盘空间达到某阈值的时候，会触发镜像的回收，删掉不被pod所使用的镜像。回收镜像的阈值可以通过kubelet的启动参数来设置。

### ContainerGC

ContainerGC负责清理节点上dead状态的container，自动清理掉node上残留的容器。具体的GC操作由runtime来实现

### ImageManager

调用kubecontainer.Imageservice提供的PullImage/GetImageRef/ListImages/RemoveImage/ImageState的方法来保证pod运行所需要的镜像，主要是为了kubelet支持cni

### VolumeManager

负责节点上pod所使用的volume的管理，主要涉及如下功能：

* volume状态的同步，模块中会启动go routine去获取当前node上volume的状态信息以及期望的volume的状态信息，会去周期性的同步volume的状态，另外volume与pod的生命周期关联，pod的创建删除过程中volume的attach/detach流程。

### containerManager

负责节点上运行的容器的cgroup配置信息。kubelet启动参数如果指定-cgroupPerQos的时候，kubelet会启动goroutine来周期性的更新pod的cgroup信息，维持其正确。实现了pod的Guaranteed/BestEffort/Burstable三种级别的Qos，通过配置kubelet可以有效的保证了当有大量pod在node上运行时，保证node节点的稳定性。

### runtimeManager

runtimeManager负责kubelet与不同的runtime实现进行对接，实现对于底层container的操作，初始化之后得到的runtime实例将会被之前描述的组建所使用。

当前可以通过kubelet的启动参数-container-runtime来定义是使用docker还是rkt.runtime。需要实现的接口定义在src/k8s.io/kubernetes/pkg/kubelet/apis/cri/service.go文件里面

### PodManager

podManager提供了接口来存储和访问pod的信息，维持static pod和mirror pod的关系。

跟其他Manager之间的关系，PodManager会被statusManager/volumeManager/runtimeManager所调用，并且podManager的接口处理流程里面会调用secretManager和configMapManager