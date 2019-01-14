# DaemonSet

DaemonSet的主要作用，让你在Kubernetes集群里，运行一个Daemon Pod。  
1. 这个Pod运行在每一个Node上；
2. 每个Node上只又一个这样的Pod实例；
3. 当又新的节点加入后，这个Pod会在新的节点上被创建出来，当旧节点被删除后，上面的Pod也会被回收。


```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

DaemonSet是依靠Toleration实现的。

## nodeAffinity tolerations

在指定的Node上创建新的Pod。

```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operator: In
            values:
            - node-geektime
```

DaemonSet Controller会在创建Pod的时候，自动在这个Pod的API对象里，加上nodeAffinity定义。 需要绑定的节点名称，就是当前正在遍历的这个Node。  

DaemonSet 还会给这个Pod自动加上一个与调度相关的字段 tolerations。这个字段会 “容忍” （Toleration）某些 “污点”（Taint）。  

```
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule

```

正常情况下，被标记了 unscheduleable “污点”的Node，是不会又任何Pod被调度上去的（effect: NoSchedule）。  DeamonSet不需要修改用户的YAML文件里的Pod模板，而是在向Kubernetes发起请求之前，直接修改根据模板生成的Pod对象。

tolerations“容忍” 所有被标记为 unschedulable “污点”的Node，允许被调度。 

DeamonSet自动地给被管理的Pod加上了这个特殊的Toleration，就使得这些Pod可以忽略这个限制，继而保证每个节点上都会被调度一个Pod。如果这个节点又给故障的话，这个Pod可能会启动失败，而DaemonSet则始终尝试下去，直到Pod启动成功。