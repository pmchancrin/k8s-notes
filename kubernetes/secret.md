# Secret

Secrete有三种类型：
* **Opaque (default)**: 任意字符串。
* **kubernetes.io/service-account-token**: 给 *service account*  用的。
* **kubernetes.io/dockercfg**: 给*Docker registry* 用的，用户下载 *docker* 镜像认证使用。

## Opaque

base64编码格式，用于存储密码，密钥等，可以通过base64 -decode解码，加密性弱。

命令创建secret

```
[root@node1 ~]# echo -n 'my-test' > ./username.txt
[root@node1 ~]# echo  -n '123456' > ./password.txt
[root@node1 ~]# kubectl create secret generic test-1 --from-file=./username.txt --from-file=./password.txt 
secret "test-1" created
```

手工创建secret

```
[root@node1 ~]# echo -n 'my-test' | base64
bXktdGVzdA==
[root@node1 ~]# echo -n '123456' | base64
MTIzNDU2
[root@node1 ~]#
[root@node1 ~]# cat test-1.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
[root@node1 ~]#
[root@node1 ~]# kubectl apply -f test-1.yaml 
secret "mysecret" created
```

查询
```
[root@node1 ~]# kubectl get secret 
NAME                  TYPE                                  DATA      AGE
default-token-xz2pl   kubernetes.io/service-account-token   3         18d
mysecret              Opaque                                2         2h
sa-test-secret        kubernetes.io/service-account-token   3         9d
sa-test-token-xqb7q   kubernetes.io/service-account-token   3         9d
test-1                Opaque                                2         2h
[root@node1 ~]# 
[root@node1 ~]# kubectl describe secret test-1
Name:		test-1
Namespace:	default
Labels:		<none>
Annotations:	<none>

Type:	Opaque

Data
====
password.txt:	6 bytes
username.txt:	7 bytes
[root@node1 ~]# 
[root@node1 ~]# kubectl get secret test-1 -o yaml
apiVersion: v1
data:
  password.txt: MTIzNDU2
  username.txt: bXktdGVzdA==
kind: Secret
metadata:
  creationTimestamp: 2018-06-25T02:52:12Z
  name: test-1
  namespace: default
  resourceVersion: "356787"
  selfLink: /api/v1/namespaces/default/secrets/test-1
  uid: c25fd9bf-7822-11e8-a0b4-0800277b75c4
type: Opaque
[root@node1 ~]# kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"MWYyZDFlMmU2N2Rm","username":"YWRtaW4="},"kind":"Secret","metadata":{"annotations":{},"name":"mysecret","namespace":"default"},"type":"Opaque"}
  creationTimestamp: 2018-06-25T03:06:21Z
  name: mysecret
  namespace: default
  resourceVersion: "357857"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: bc8b2691-7824-11e8-a0b4-0800277b75c4
type: Opaque

```

使用secret

*通过加到卷来访问secret*
```
[root@node1 ~]# cat busybox3.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command:
    - sleep
    - "3600"
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret-volume
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: mysecret
[root@node1 ~]# 
[root@node1 ~]# kubectl create -f busybox3.yaml 
pod "busybox" created
[root@node1 ~]# 
[root@node1 ~]# kubectl exec busybox -it ls /etc/secret-volume
password  username
[root@node1 ~]# 
```

通过环境变量访问secret*
```
[root@node1 ~]# cat busybox3.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command:
    - sleep
    - "3600"
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: username
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: password
[root@node1 ~]# kubectl create -f busybox3.yaml 
pod "busybox" created
```

* *多个pods可以使用同一个secret*
* *可以设置挂载目录权限*
* *可以通过secret.items来重新定义挂载目录*
* *secret存在etcd里，可以被自动刷新，kubelet定时维护容器的卷*

## default

每个pod都会自动挂载一个default-token-xxx的volume的secret到/var/run/secrets/kubernetes.io/serviceaccount目录，正好是默认Service Account对应的ServiceAccountToken

```
[root@node11 ~]# kubectl describe pod nginx-deployment-5896fbb489-4966s 
...
Containers:
  nginx:
    Container ID:   docker://7c84cb527400623792da7da1e4c13697f8929c6237257219e6ff3e81c2cfa782
  ...
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-782hd (ro)
...
Volumes:
  default-token-782hd:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-782hd
    Optional:    false
...

[root@node11 ~]# kubectl exec nginx-deployment-5896fbb489-4966s ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token

```

容器中的应用就可以通过直接加载这些授权文件，访问操作Kubernetes API了。通过Kubernetes官方的Client包(k8s.io/client-go)，可以自动加载这个目录的文件，不需要配置和编码。  

这种把Kubernetes客户端以容器的方式运行在集群里，然后通过default Service Account自动授权的方式，成为"InClusterConfig"，是推荐的Kubernetes API编程的授权方式。

## kubernetes.io/service-account-token

创建sa时会自动创建默认的secret，或者可以手动创建；

## kubernetes.io/dockercfg

存放私有Dokcer Registry的认证信息，当Kubernetes在创建Pod并且需要从私有Docker Registry pull镜像时，需要使用认证信息，就会用到kubernetes.io/dockercfg类型的Secret

创建
```
kubectl create secret docker-registry regsecret \
--docker-server=harbor.frognew.com \
--docker-username=the-user \
--docker-password=the-password \
--docker-email=the-email \
--namespace=the-namspace 
```

使用
```
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
    - name: private-reg-container
      image: <your-private-image>
  imagePullSecrets:
    - name: regsecret
```

## 参考
* https://kubernetes.io/docs/concepts/configuration/secret/
* https://kubernetes.io/cn/docs/concepts/configuration/secret/
* https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/