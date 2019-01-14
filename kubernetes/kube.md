# API Server

<img src="https://kubernetes.feisky.xyz/zh/components/images/kube-apiserver.png" width="500" hegiht="313" align=center />

## 访问API server

* kubectl 命令访问
```
#打开调试开关
kubectl -v=8 get pods
kubectl get --raw /api/v1/namespaces
```

* REST API
* client库

## swagger-ui

* apiserver 启动参数
> * ***--insecure-bind-address=0.0.0.0***  The IP address on which to serve the --insecure-port (set to 0.0.0.0 for all interfaces). (default 127.0.0.1)
> * ***--insecure-port=8080***
> * ***--enable-swagger-ui=true***

* 浏览器打开：
> * ***http://192.168.137.101:8080/swagger-ui/***
> * ***http://192.168.137.101:8080/swagger.json***
> * ***http://192.168.137.101:8080/swaggerapi***

## 访问控制

Kubernetes API 的每个请求都会经过多阶段的访问控制之后才会被接受，这包括认证、授权以及准入控制（Admission Control）等。

* 认证 失败返回401
* 授权 失败返回403

## 资源配额
> http://docs.kubernetes.org.cn/750.html

# kube-schedule

负责分配调度 Pod 到集群内的节点上，它监听 kube-apiserver，查询还未分配 Node 的 Pod，然后根据调度策略为这些 Pod 分配节点（更新 Pod 的 NodeName 字段）

## 指定Node节点调度

## nodeSelector  只调度到匹配指定 label 的 Node 上

```
# node 打label
kubectl label nodes node-01 disktype=ssd
# pod 设置
spec:
  nodeSelector:
    disktype: ssd
```

## 参考
> https://kubernetes.io/docs/concepts/configuration/assign-pod-node/

## nodeAffinity

## podAffinity

## taints 用于node

* NoSchedule：新的 Pod 不调度到该 Node 上，不影响正在运行的 Pod
* PreferNoSchedule：soft 版的 NoSchedule，尽量不调度到该 Node 上
* NoExecute：新的 Pod 不调度到该 Node 上，并且删除（evict）已在运行的 Pod。Pod 可以增加一个时间（tolerationSeconds）

```

#设置 taint
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
#删除 taint
kubectl taint nodes node1 key1:NoSchedule-
kubectl taint nodes node1 key1:NoExecute-
kubectl taint nodes node1 key2:NoSchedule-
#查询
[root@lzg-test-dnw55arvno6m-master-0 kubernetes]# kubectl describe nodes lzg-test-dnw55arvno6m-master-0
Name:			lzg-test-dnw55arvno6m-master-0
Role:			
Labels:			beta.kubernetes.io/arch=amd64
			beta.kubernetes.io/os=linux
			dedicated=master
			failure-domain.beta.kubernetes.io/zone=nova
			kubernetes.io/hostname=lzg-test-dnw55arvno6m-master-0
Annotations:		node-mgnt-ip=172.16.106.102
			node.alpha.kubernetes.io/ttl=0
			volumes.kubernetes.io/controller-managed-attach-detach=true
Taints:			dedicated=master:NoSchedule

```

## tolerations 用于pod

配了tolerations 可以被分配到taints的节点，也可以分配到其他节点，如果您希望这些 pod 只能被分配到上述专用节点，那么您还需要给这些专用节点另外添加一个和上述 taint 类似的 label。
```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600

tolerations:
- key: "node.alpha.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

## 参考：
* https://k8smeetup.github.io/docs/concepts/configuration/taint-and-toleration/
* http://blog.51cto.com/nosmoking/2097380
* https://blog.csdn.net/yevvzi/article/details/77966171

## kubectl cordon/uncordon node

禁止/取消禁止pod调度到该节点

## kubectl drain node

驱逐该节点上的所有pod

## 参考

* https://kubernetes.feisky.xyz/zh/components/scheduler.html
* https://jimmysong.io/kubernetes-handbook/concepts/node.html
* https://jimmysong.io/kubernetes-handbook/concepts/taint-and-toleration.html
* https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/
* https://k8smeetup.github.io/docs/concepts/configuration/taint-and-toleration/



# kube-controller-manager

<img src="https://kubernetes.feisky.xyz/zh/components/images/post-ccm-arch.png" width="500" hegiht="313" align=center />

## controller manager由一系列控制器组成

- Replication Controller
- Node Controller
- CronJob Controller
- Daemon Controller
- Deployment Controller
- Endpoint Controller
- Garbage Collector
- Namespace Controller
- Job Controller
- Pod AutoScaler
- RelicaSet
- Service Controller
- ServiceAccount Controller
- StatefulSet Controller
- Volume Controller
- Resource quota Controller

## 高可用

在启动时设置 --leader-elect=true 后，controller manager 会使用多节点选主的方式选择主节点。只有主节点才会调用 StartControllers() 启动所有控制器，而其他从节点则仅执行选主算法。

## 参考

* https://kubernetes.feisky.xyz/zh/components/controller-manager.html

# kubelet

每个节点上都运行一个 kubelet 服务进程，默认监听 10250 端口，接收并执行 master 发来的指令，管理 Pod 及 Pod 中的容器。每个 kubelet 进程会在 API Server 上注册节点自身信息，定期向 master 节点汇报节点的资源使用情况，并通过 cAdvisor 监控节点和容器的资源。

## 端口

* 10250端口
* cAdvisor访问端口4149：
http://192.168.137.101:4194/containers/
* 10255端口

## 静态pod

* 配置清单位置：--pod-manifest-path=/etc/kubernetes/manifests
* kubelet将会周期扫描这个目录，根据这个目录下出现或消失的YAML/JSON文件来创建或删除静态pod
* 我们不能通过API服务器来删除静态pod（例如，通过 kubectl 命令），kebelet不会删除它。

## 参考

* https://kubernetes.io/docs/tasks/administer-cluster/static-pod/