# Overview

> https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/

K8s-OS  
Container-Process  
Pod-Process Group  

Pod是一个逻辑概念，是一组共享某些资源的容器。  
Pod里的所有容器，共享同一个Network Namespace。
凡是调度，网络，存储，安全相关的属性都是Pod级别的。  

**Networking**
> Each Pod is assigned a **unique IP address**. Every container in a Pod shares the network namespace, including the IP address and network ports. Containers inside a Pod can communicate with one another using localhost. When containers in a Pod communicate with entities outside the Pod, they must coordinate how they use the shared network resources (such as ports).

 **Storage**
> A Pod can specify a set of shared storage volumes. All containers in the Pod can access the shared volumes, allowing those containers to share data. Volumes also allow persistent data in a Pod to survive in case one of the containers within needs to be restarted. 

# infra container

k8s.gcr.io/pause

# Spec

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  nodeSelector:
    disktype:ssd
  hostAliases:
  - ip: "1.1.1.1"
    hostname:
    - "foo.remote"
    - "bar.remote"
  shareProcessNamespace:true
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:      
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh","-c","echo hello > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]

```
* nodeSelcetor： Pod和Node绑定的字段
* HostAliases：定义了/etc/hosts的你内容
* shareProcessNamespace：共享PID namespaces
* lifecycle: Container lifecycle hooks.  
* postStart: Container启动后立即执行的操作。不严格保证在ENTRYPOINT后。
* preStop：容器被kill前执行的操作。同步的，会阻塞kill流程，完成后才能被kill。

> https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/

# status

* Pending 已创建保存到ETCD，但没有被顺利创建。
* Running 已调度成功，至少一个容器在运行中。
* Ready 已running并且可以对外提供服务了。

# initContainer

Pod can have one or more Init Container, which are run before the app Containers are started.  
* They always run to completion.  
* Each one must complete successfully before the next one is started.

## Example

```
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}

```

# Design Pattern for Container-based Distribute Systems

> https://www.usenix.org/conference/hotcloud16/workshop-program/presentation/burns

sidecar pattern like using Init Container

# Projected Volume

> https://kubernetes.io/docs/concepts/storage/volumes/#projected  
> https://kubernetes.io/docs/tasks/configure-pod-container/configure-projected-volume-storage/

* Secret
* ConfigMap
* Downward API
* ServiceAccountToken

```
pods/storage/projected.yaml  
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume
spec:
  containers:
  - name: test-projected-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass

```

# Liveness and Readiness Probes

> https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/

* The kubelet uses liveness probes to know when to restart a Container.   
* The kubelet uses readiness probes to know when a Container is ready to start accepting traffic.

liveness example:
```
pods/probe/exec-liveness.yaml  
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

liveness can be definded by command,HTTP  request,TCP, named prot.

Readiness probes are configured similarly to liveness probes. The only difference is that you use the readinessProbe field instead of the livenessProbe field.
```
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

# restartPolicy
 > https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#example-states
 
* Pod的restartPolicy.Kubernetes没有stop，有restart，但实际上是重新创建。   
* Pod恢复过程拥有都发生在当前节点上，不会跑到其他节点上去。 
* 一旦Pod和一个节点绑定，除非这个绑定关系发生变化，否则它永远都不会离开这个节点。当这个节点宕机了，这个Pod也不会主动迁移的。

1. Pod的restartPolicy指定的策略允许重启一场的容器（比如，always），那么这个Pod就会保持running状态，并进行容器重启，否正就会进入Failed状态。
2. 包含多个容器的Pod，只要它里面所有的容器都进入异常状态后，Pod才会进入Failed状态。在此之前，Pod都是running状态。此时Ready会显示正常容器的个数。

# PodPreset

> https://kubernetes.io/docs/concepts/workloads/pods/podpreset/   
> https://kubernetes.io/docs/tasks/inject-data-application/podpreset/

## enable PodPreset

modify /etc/kubernetes/manifests/kube-apiserver.yaml
```
- command:
    ...
    - --enable-admission-plugins=NodeRestriction,PodPreset
    - --runtime-config=settings.k8s.io/v1alpha1=true
```
