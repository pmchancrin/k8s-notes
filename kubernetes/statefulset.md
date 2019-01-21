# StatefulSet

StatefulSet is the workload API object used to manage stateful applications.

> https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

应用状态分类：
1. **拓扑状态** 多个实例启动有先后顺序，比如先A再B，如果A，B都删掉了，再次创建出来的时候也必须遵守这个顺序。新创建的Pod必须和原来的网络标识意义，这样原先访问者才能使用同样的方法，访问到这个新的Pod。  
2. **存储状态** 多个实例分别绑定不同的存储数据。重新创建后还需继续可访问之前的存储数据。

## Demo1

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web

---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```

```
[root@node11 ~]# kubectl apply -f statefulset.yaml 
statefulset.apps/web created
service/nginx created

[root@node11 ~]# kubectl get service nginx 
NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   None         <none>        80/TCP    15s

[root@node11 ~]# kubectl get statefulsets.apps 
NAME   READY   AGE
web    2/2     31s

[root@node11 ~]# kubectl get pods -w -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          61s
web-1   1/1     Running   0          60s

[root@node11 ~]# kubectl describe statefulsets web
...
Events:
  Type    Reason            Age   From                    Message
  ----    ------            ----  ----                    -------
  Normal  SuccessfulCreate  98s   statefulset-controller  create Pod web-0 in StatefulSet web successful
  Normal  SuccessfulCreate  97s   statefulset-controller  create Pod web-1 in StatefulSet web successful
 
```

StatefulSet 先后创建Pod，分别命名web-0，web-1。

查询下每个pod的hostname，于Pod名称相同：
```
[root@node11 ~]# kubectl exec web-0 -- sh -c 'hostname'
web-0
[root@node11 ~]# kubectl exec web-1 -- sh -c 'hostname'
web-1

```

使用nslookup 查询 DNS
```
[root@node11 ~]# kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh
If you don't see a command prompt, try pressing enter.
/ # nslookup web-0.nginx.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx.default.svc.cluster.local
Address 1: 192.168.72.22 web-0.nginx.default.svc.cluster.local
/ # nslookup web-1.nginx.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx.default.svc.cluster.local
Address 1: 192.168.226.81 web-1.nginx.default.svc.cluster.local
/ # ping web-0.nginx.default.svc.cluster.local
PING web-0.nginx.default.svc.cluster.local (192.168.72.22): 56 data bytes
64 bytes from 192.168.72.22: seq=0 ttl=63 time=0.056 ms
64 bytes from 192.168.72.22: seq=1 ttl=63 time=0.126 ms

```

有状态应用必须通过DNS或者hostname的方式来访问，IP会变化的。

当删除掉这两个有状态应用的Pod，Kubernetes会按照原先的顺序，创建出两个新的Pod，并且分配了于原来相同的“网络身份”。通过这种严格的对应规则，StatefulSet就保证了Pod网络标识的稳定性。

## Demo2

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

当我们创建StatefulSet后，Kubernetes集群里面会多2个PVC，这些PVC以 “<PVC名称>-<StatefulSet名称>-<编号>” 的方式命名。

当Pod被删除后，对应的PVC和PV并不会被删除，Kubernetes会按照顺序重新恢复Pod，并找到之前的的PVC，进而和这个PVC绑定的PV。

通过这种方式实现应用存储状态的管理。

## Demo3
https://github.com/oracle/kubernetes-website/blob/master/docs/tasks/run-application/mysql-statefulset.yaml

## update
```
$ kubectl patch statefulset mysql --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"mysql:5.7.23"}]'
statefulset.apps/mysql patched
```

StatefulSet Controller会按照Pod编号相反的顺序，从最后一个Pod开始，逐一更新每个Pod。  
sepc.updateStrategy.rollingUpdate的partition字段，可以更精细的控制“滚动更新”，比如Canary Deploy。  

```
$ kubectl patch statefulset mysql -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
statefulset.apps/mysql patched
```
使用path或者edit修改partition字段，只有序号大于或者等于2的Pod会被更新到这个版本，并且，删除或者重启序号小于2的Pod，等它再次启动，依然会保持之前的版本，不会被升级到新版本。