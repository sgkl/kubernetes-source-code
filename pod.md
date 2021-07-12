+ Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.
+ A Pod is a group of one or more containers, with shared storage and network resources, and a specification for how to run the containers.
+ The shared context of a Pod is a set of Linux namespaces, cgroups, and potentially other facets of isolation - the same things that isolate a Docker container.

+ Static Pods are managed directly by the kubelet daemon on a specific node, without the API server observing them. Whereas most Pods are managed by the control plane, for static Pods, the kubelet directly supervises each static Pod.

The kubelet can optionally perform and react to three kinds of probes on running containers:
+ livenessProbe: Indicates whether the container is running. If the liveness probe fails, the kubelet kills the container, and the container is subjected to its restart policy.
+ readinessProbe: Indicates whether the container is ready to respond to requests. If the readiness probe fails, the endpoints controller removes the Pod's IP address from the endpoints of all Services that match the Pod.
+ startupProbe: Indicates whether the application within the container is started. All other probes are disabled if a startup probe is provided, until it succeeds. 


> Pod的恢复过程，永远都是发生在当前节点上，而不会跑到别的节点上去。事实上，一旦一个 Pod 与一个节点（Node）绑定，除非这个绑定发生了变化（pod.spec.node 字段被修改），否则它永远都不会离开这个节点。这也就意味着，如果这个宿主机宕机了，这个 Pod 也不会主动迁移到其他节点上去。

> 如果你想让 Pod 出现在其他的可用节点上，就必须使用 Deployment 这样的“控制器”来管理Pod，哪怕你只需要一个 Pod 副本。



> StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。而当 StatefulSet 的“控制循环”发现 Pod 的“实际状态”与“期望状态”不一致，需要新建或者删除Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。
