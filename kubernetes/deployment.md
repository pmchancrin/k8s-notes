# Controller Pattern

Controller都放在 *pkg/controller* 目录下，都遵循Kubernetes项目中的一个通用编排模式：控制循环（control loop）

```
for {
  实际状态 := 获取集群中对象 X 的实际状态（Actual State）
  期望状态 := 获取集群中对象 X 的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```

实际状态一般来自于Kubernetes集群本身。  
期望状态一般来自于用户提交的YAML文件。

Deployment 控制器的实现步骤：

1. Deployment 控制器从Etcd里面获取所有携带“app:nginx”标签的Pod，统计他们的数量，这是实际状态；
2. Deployment 对象的Replicas字段是期望状态；
3. Deployment 控制器讲两个状态进行对比，根据对比结果，确定删除还是创建Pod。 这种对比操作叫调谐（Reconcile），这哥调谐的过程，称作调谐循环（Reconcile Loop）或者同步循环（Sync Loop）。

# Deployment

> https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

![Kubernetes Deployment Yaml](/images/kubernetes-deployment-yaml.png)

* Deployment用在Stateless Application场景。
* Deployment是个两层控制器。  
* 上半部分是控制器定义，包括期望状态，下半部分是被控制对象的模板叫PodTemplete组成。
* Deployment 控制器实际操作的是ReplicaSet对象，而不是Pod对象。

![Kubernetes Deployment ReplicaSet](/images/kubernetes-deployment-rs.png)

在所有API对象的Metadata里面，都有一个字段叫作 ownerReference，保持当前这个API对象的拥有者Owner的信息。

deployment创建了replicaSet，replicateSet创建了Pod：

```
[root@node11 ~]# kubectl get pods -o yaml nginx-deployment-5896fbb489-4966s
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2018-12-12T07:13:22Z"
  generateName: nginx-deployment-5896fbb489-
  labels:
    app: nginx
    pod-template-hash: 5896fbb489
  name: nginx-deployment-5896fbb489-4966s
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: nginx-deployment-5896fbb489
    uid: 6913b1a4-fddd-11e8-a06c-080027c2b927
  resourceVersion: "253112"
  selfLink: /api/v1/namespaces/default/pods/nginx-deployment-5896fbb489-4966s
  uid: 6916c238-fddd-11e8-a06c-080027c2b927
[root@node11 ~]# kubectl get replicaset
NAME                           DESIRED   CURRENT   READY   AGE
nginx-deployment-5896fbb489    2         2         2       43h
[root@node11 ~]# kubectl get deployment
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment    2/2     2            2           44h

[root@node11 ~]# kubectl get replicaset nginx-deployment-5896fbb489 -o yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  annotations:
    deployment.kubernetes.io/desired-replicas: "2"
    deployment.kubernetes.io/max-replicas: "3"
    deployment.kubernetes.io/revision: "2"
  creationTimestamp: "2018-12-12T07:13:22Z"
  generation: 2
  labels:
    app: nginx
    pod-template-hash: 5896fbb489
  name: nginx-deployment-5896fbb489
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: Deployment
    name: nginx-deployment
    uid: 0ab68b8c-fddd-11e8-a06c-080027c2b927
  resourceVersion: "177670"
  selfLink: /apis/extensions/v1beta1/namespaces/default/replicasets/nginx-deployment-5896fbb489
  uid: 6913b1a4-fddd-11e8-a06c-080027c2b927
```

ReplicaSet会把这个随机字符串家到它控制的所有Pod的标签里后面带的字符串叫pod-template-hash，ReplicaSet会把这个随机字符串家到它控制的所有Pod的标签里，包子这些Pod不会跟集群里其他Pod混淆。  

ReplicaSet会把pod-template-hash加到自己的Lable里面。

```
[root@node11 ~]# kubectl get rs
NAME                           DESIRED   CURRENT   READY   AGE
nginx-deployment-5896fbb489    4         4         4       44h
[root@node11 ~]# kubectl describe rs nginx-deployment-5896fbb489
Name:           nginx-deployment-5896fbb489
Namespace:      default
Selector:       app=nginx,pod-template-hash=5896fbb489
Labels:         app=nginx
                pod-template-hash=5896fbb489
Annotations:    deployment.kubernetes.io/desired-replicas: 4
                deployment.kubernetes.io/max-replicas: 5
                deployment.kubernetes.io/revision: 2
Controlled By:  Deployment/nginx-deployment

[root@node11 ~]# kubectl get pods 
NAME                                 READY   STATUS    RESTARTS   AGE
nginx-deployment-5896fbb489-4966s    1/1     Running   1          44h
nginx-deployment-5896fbb489-5ckfb    1/1     Running   0          7m39s

```

## Operation

* scale

```
[root@node11 ~]# kubectl scale deployment nginx-deployment --replicas=4
deployment.extensions/nginx-deployment scaled
```

* edit

```
[root@node11 ~]# kubectl edit deployment/nginx-deployment
deployment.extensions/nginx-deployment edited
[root@node11 ~]# kubectl get deployments
...
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  82s   deployment-controller  Scaled up replica set nginx-deployment-6fc57fccb6 to 1
  Normal  ScalingReplicaSet  82s   deployment-controller  Scaled down replica set nginx-deployment-5896fbb489 to 3
  Normal  ScalingReplicaSet  82s   deployment-controller  Scaled up replica set nginx-deployment-6fc57fccb6 to 2
  Normal  ScalingReplicaSet  80s   deployment-controller  Scaled down replica set nginx-deployment-5896fbb489 to 2
  Normal  ScalingReplicaSet  80s   deployment-controller  Scaled up replica set nginx-deployment-6fc57fccb6 to 3
  Normal  ScalingReplicaSet  80s   deployment-controller  Scaled down replica set nginx-deployment-5896fbb489 to 1
  Normal  ScalingReplicaSet  80s   deployment-controller  Scaled up replica set nginx-deployment-6fc57fccb6 to 4
  Normal  ScalingReplicaSet  78s   deployment-controller  Scaled down replica set nginx-deployment-5896fbb489 to 0

```

## RollingUpdateStrategy

deault

```
[root@node11 ~]# kubectl describe deployment nginx-deployment
...
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
```

configure

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1

```

![Kubernetes Deployment ReplicaSet](/images/kubernetes-deployment-rs-2.png)

Deployment控制的是ReplicaSet的数目和属性。  
一个版本对应的是一个ReplicaSet，这个版本应用的Pod数量，又RecplicaSet来保证。

通过kubectl set image更新image到1.91版本，这个版本不存在，新的ReplicaSet(hash=79dccf98ff)出错停止了，它创建了2个Pod，都没有READY，旧ReplicaSet(hash=76bf4969df)水平收缩，也自动停止了，一个旧Pod被删。

```
[root@node11 ~]# kubectl get deployments.apps nginx-deployment 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4/4     4            4           7s
[root@node11 ~]# 
[root@node11 ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-76bf4969df   4         4         4       19s
[root@node11 ~]# 
[root@node11 ~]# kubectl set image deployment nginx-deployment nginx=nginx:1.91 --record=true
deployment.extensions/nginx-deployment image updated
[root@node11 ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-76bf4969df   3         3         3       106s
nginx-deployment-79dccf98ff   2         2         0       5s
[root@node11 ~]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/4     2            3           113s
[root@node11 ~]# kubectl rollout status deployment nginx-deployment 
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 4 new replicas have been updated...
```

通过kubectl rollout undo 可以回滚到上一个版本

```
[root@node11 ~]# kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment rolled back
[root@node11 ~]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4/4     4            4           8m51s
[root@node11 ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-76bf4969df   4         4         4       8m54s
nginx-deployment-79dccf98ff   0         0         0       7m13s
```

通过kubectl rollout history 可以查看Deployment变更的版本，指定了--record都会被记录下来。可以查看具体版本，和回滚到具体版本。

```
[root@node11 ~]# kubectl rollout history deployment nginx-deployment 
deployment.extensions/nginx-deployment 
REVISION  CHANGE-CAUSE
2         kubectl set image deployment nginx-deployment nginx=nginx:1.91 --record=true
3         <none>

[root@node11 ~]# kubectl rollout history deployment nginx-deployment --revision=2
deployment.extensions/nginx-deployment with revision #2
Pod Template:
  Labels:	app=nginx
	pod-template-hash=79dccf98ff
  Annotations:	kubernetes.io/change-cause: kubectl set image deployment nginx-deployment nginx=nginx:1.91 --record=true
  Containers:
   nginx:
    Image:	nginx:1.91
    Port:	80/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>

[root@node11 ~]# kubectl rollout undo deployment nginx-deployment --to-revision=2
```

每次对Deployment进行更新操作，都会生成一个新的ReplicaSet对象。  
可以通过pause Deployment后进行更新操作，再resume，这样就只生成一个新的ReplicaSet。

```
$ kubectl rollout pause deployment/nginx-deployment
deployment.extensions/nginx-deployment paused

$ kubectl edit xxx /set image ...

$ kubectl rollout resume deploy/nginx-deployment
deployment.extensions/nginx-deployment resumed
```

## Blue-Green / Canary Deployment

https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/canary

## NOTE

1. 不同namespace相同的对象，是完全不同的对象。
2. Deployment 只允许容器的restartPolicy=Always
3. 将Deployment的spec.revisionHistoryLimit设置为0，就不能再回滚了。

