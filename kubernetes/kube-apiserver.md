API Server
===========
提供了k8s各类资源对象（pod,RC,Service等）的增删改查及watch等REST接口，是整个系统的数据总线和数据中心。
* 集群管理的API接口（包括授权，认证，访问控制等）
* 其他模块间数据通信的枢纽（只有API Server才能直接操作etcd）
* 资源配额控制入口；

<img src="https://kubernetes.feisky.xyz/zh/components/images/kube-apiserver.png" width="500" hegiht="313" align=center />

### 启动参数

```
$ cat /etc/kubernetes/manifests/kube-apiserver.manifest
...
 command:
    - /hyperkube
    - apiserver
    - --advertise-address=192.168.137.101
    - --etcd-servers=https://192.168.137.101:2379,https://192.168.137.102:2379,https://192.168.137.103:2379
    - --etcd-quorum-read=true
    - --etcd-cafile=/etc/ssl/etcd/ssl/ca.pem
    - --etcd-certfile=/etc/ssl/etcd/ssl/node-node2.pem
    - --etcd-keyfile=/etc/ssl/etcd/ssl/node-node2-key.pem
    - --insecure-bind-address=127.0.0.1
    - --apiserver-count=2
    - --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota
    - --service-cluster-ip-range=10.233.0.0/18
    - --service-node-port-range=30000-32767
    - --client-ca-file=/etc/kubernetes/ssl/ca.pem
    - --basic-auth-file=/etc/kubernetes/users/known_users.csv
    - --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem
    - --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --token-auth-file=/etc/kubernetes/tokens/known_tokens.csv
    - --service-account-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --secure-port=6443
    - --insecure-port=8080
    - --storage-backend=etcd3
    - --v=2
    - --allow-privileged=true
    - --anonymous-auth=False

```
### API说明
#### API groups
* core group.  path: **/api/v1**, **aipVersion:v1**
* other group. path: **/apis/$GROUP_NAME/$VERSION**, **apiVersion: $GROUP_NAME/$VERSION**

```
## api 
[root@node1 ~]# curl localhost:8080/
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1beta1",
    "/apis/apps",
    "/apis/apps/v1beta1",
...
## version
[root@node1 ~]# curl localhost:8080/version

## core group
[root@node1 ~]# curl localhost:8080/api
[root@node1 ~]# curl localhost:8080/api/v1
[root@node1 ~]# curl localhost:8080/api/v1/pods
[root@node1 ~]# curl localhost:8080/api/v1/pods/status

## other group
[root@node1 ~]# curl localhost:8080/apis
[root@node1 ~]# curl localhost:8080/apis/apiextensions.k8s.io
[root@node1 ~]# curl localhost:8080/apis/apiextensions.k8s.io/v1beta1
```
#### 启用禁用API groups
默认API groups都是是启动的,通用的group的资源也是默认启动的，可以通过apiserver启动参数 --runtime-config来设置group以及其资源。
```
--runtime-config=batch/v1=false,batch/v2alpha1
--runtime-config=extensions/v1beta1/deployments=false,extensions/v1beta1/ingress=false
```
> 修改后需要重启apiserver和controller-manager。

#### 参考
* https://kubernetes.io/docs/concepts/overview/kubernetes-api/

### 访问API Server
默认情况下，kube-apiserver进程在本机通过：
* insecure-bind-address:insecure-port（127.0.0.1:8080或者localhost:8080）端口提供非安全认证的REST服务；
* secure-port（6443）端口提供HTTPS安全服务；==（用的哪个IP？？）==
#### kubectl 命令访问
kubectrl 跟api service 也是通过REST方式通信的。

* 打开调试开关
```
[root@node1 ~]# kubectl -v=8 get pods
I0704 09:57:25.641993    8087 loader.go:357] Config loaded from file /root/.kube/config
I0704 09:57:25.754547    8087 round_trippers.go:383] GET https://192.168.137.101:6443/api
I0704 09:57:25.754574    8087 round_trippers.go:390] Request Headers:
I0704 09:57:25.754588    8087 round_trippers.go:393]     Accept: application/json, */*
I0704 09:57:25.754600    8087 round_trippers.go:393]     User-Agent: kubectl/v1.7.5+coreos.0 (linux/amd64) kubernetes/070d238
I0704 09:57:25.766195    8087 round_trippers.go:408] Response Status: 200 OK in 11 milliseconds
I0704 09:57:25.766224    8087 round_trippers.go:411] Response Headers:
I0704 09:57:25.766229    8087 round_trippers.go:414]     Date: Wed, 04 Jul 2018 01:57:25 GMT
I0704 09:57:25.766234    8087 round_trippers.go:414]     Content-Type: application/json
I0704 09:57:25.766237    8087 round_trippers.go:414]     Content-Length: 138
I0704 09:57:25.766272    8087 request.go:991] Response Body: {"kind":"APIVersions","versions":["v1"],"serverAddressByClientCIDRs":[{"clientCIDR":"0.0.0.0/0","serverAddress":"192.168.137.101:6443"}]}
I0704 09:57:25.766389    8087 round_trippers.go:383] GET https://192.168.137.101:6443/apis
I0704 09:57:25.766395    8087 round_trippers.go:390] Request Headers:
I0704 09:57:25.766400    8087 round_trippers.go:393]     Accept: application/json, */*
I0704 09:57:25.766419    8087 round_trippers.go:393]     User-Agent: kubectl/v1.7.5+coreos.0 (linux/amd64) kubernetes/070d238
I0704 09:57:25.767277    8087 round_trippers.go:408] Response Status: 200 OK in 0 milliseconds
I0704 09:57:25.767367    8087 round_trippers.go:411] Response Headers:
I0704 09:57:25.767372    8087 round_trippers.go:414]     Content-Type: application/json
I0704 09:57:25.767376    8087 round_trippers.go:414]     Content-Length: 3274
I0704 09:57:25.767381    8087 round_trippers.go:414]     Date: Wed, 04 Jul 2018 01:57:25 GMT

```
* kubectl 直接访问API
```
## 版本信息
[root@node1 ~]# kubectl get --raw /api
{"kind":"APIVersions","versions":["v1"],"serverAddressByClientCIDRs":[{"clientCIDR":"0.0.0.0/0","serverAddress":"192.168.137.101:6443"}]}
## 支持的资源
[root@node1 ~]# kubectl get --raw /api/v1
## 集群资源
[root@node1 ~]# kubectl get --raw /api/v1/pods
[root@node1 ~]# kubectl get --raw /api/v1/services
[root@node1 ~]# kubectl get --raw /api/v1/replicationcontrollers
```
#### REST API

> **api-reference:**
>  https://v1-7.docs.kubernetes.io/docs/api-reference/v1.7/

##### swagger-ui
* apiserver 启动参数
> * ***--insecure-bind-address=0.0.0.0***  The IP address on which to serve the --insecure-port (set to 0.0.0.0 for all interfaces). (default 127.0.0.1)
> * ***--insecure-port=8080***
> * ***--enable-swagger-ui=true***

* 浏览器打开：
> * ***http://192.168.137.101:8080/swagger-ui/***
> * ***http://192.168.137.101:8080/swagger.json***
> * ***http://192.168.137.101:8080/swaggerapi***

##### API访问
Kubernetes API 的每个请求都会经过多阶段的访问控制之后才会被接受，这包括认证、授权以及准入控制（Admission Control）等。
* 认证 失败返回401
* 授权 失败返回403
```
[root@node1 ~]# curl https://node1:6443/api/v1/nodes
Unauthorized

[root@node1 ~]# curl localhost:8080/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.137.101:6443"
    }
  ]

## CA
[root@node test]# curl https://node1:6443/api/v1/nodes --cert ./test.pem --key ./test-key.pem --cacert ./ca.pem 

## token
[root@node3 test]# curl -k --header "Authorization: Bearer UPVzqw3LVaqpPCrijfuc2rwKadjvGNUq" https://node1:6443/api/v1/nodes

## password
[root@node3 test]# ./kubectl --server=https://node1:6443  get nodes
Please enter Username: kube
Please enter Password: ***************
```

##### client库
通过编程的方式访问，这种方式有两种使用场景：
* pod需要发现同属于一个service的其它pod副本的信息来组建集群（如elasticsearch集群）；
* 需要通过调用api来我开发基于k8s的管理平台；

pod中的进程通过API server的service来访问API Server。
```
[root@node1 ~]# kubectl get services
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.233.0.1   <none>        443/TCP   27d

```

client-go ：https://github.com/kubernetes/client-go

##### kubectl proxy 
启动内部代理，然后访问内部代理进行调用，可以进行访问控制，如限制要访问的接口，增加允许访问的client的白名单。
```
## 8001端口启动代理，拒绝客户端访问pods
[root@node1 ~]# kubectl proxy --reject-paths="^/api/v1/pods" --port=8001 --v=2
[root@node1 ~]# curl localhost:8001/api/v1/pods
<h3>Unauthorized</h3>
[root@node1 ~]# 

```
#### proxy API接口
API Server把收到的REST请求转发到某个node上的kubelet进程的REST端口上，由kubelet进程负责响应。
这里获取的数据来源于node，并非ETCD，会有时间上偏差。
```
[root@node1 ~]# curl localhost:8080/api/v1/proxy/nodes/node3/pods/
[root@node1 ~]# curl localhost:8080/api/v1/proxy/nodes/node3/stats/
[root@node1 ~]# curl localhost:8080/api/v1/proxy/nodes/node3/spec/

```

### 资源配额

### 其他模块通信
* kubelet --> apiserver (定期调用REST接口汇报自身状态，API server更新到etcd中， watch监听pod信息，监听到有pod被调度到本节点，或者本节点pod被删或者修改后执行相应操作)
* controller-manager --> apiserver (node controller 通过watch接口实时监控Node的信息，并做相应的处理。)
* scheduler --> apiserver （watch侦听新建pod的信息， 检索符合要求的node列表，并进行调度逻辑的执行）

==为缓解apiserver的压力， 各个模块会缓存从apiserver获取的数据， 某些情况下会直接使用缓存的数据（什么情况使用缓存， 如何保证数据的一致性？）==
```
[root@node3 node1_6443]# ll /root/.kube/cache/discovery/node1_6443/
total 64
drwxr-xr-x. 3 root root 4096 Jun 27 16:35 apiextensions.k8s.io
drwxr-xr-x. 3 root root 4096 Jun 27 16:35 apiregistration.k8s.io
drwxr-xr-x. 3 root root 4096 Jun 27 16:35 apps
drwxr-xr-x. 4 root root 4096 Jun 27 16:35 authentication.k8s.io
drwxr-xr-x. 4 root root 4096 Jun 27 16:35 authorization.k8s.io
drwxr-xr-x. 3 root root 4096 Jun 27 16:35 autoscaling
drwxr-xr-x. 3 root root 4096 Jun 27 16:35 batch
drwxr-xr-x. 3 root root 4096 Jun 27 16:35 certificates.k8s.io
drwxr-xr-x. 3 root root 4096 Jun 27 16:35 extensions
drwxr-xr-x. 3 root root 4096 Jun 27 16:35 networking.k8s.io
drwxr-xr-x. 3 root root 4096 Jun 27 16:35 policy
drwxr-xr-x. 4 root root 4096 Jun 27 16:35 rbac.authorization.k8s.io
-rwxr-xr-x. 1 root root 3426 Jun 27 16:35 servergroups.json
drwxr-xr-x. 3 root root 4096 Jun 27 16:35 settings.k8s.io
drwxr-xr-x. 4 root root 4096 Jun 27 16:35 storage.k8s.io
drwxr-xr-x. 2 root root 4096 Jun 27 16:35 v1

```


### 参考
> * http://docs.kubernetes.org.cn/750.html
> * http://www.huweihuang.com/article/kubernetes/core-principle/kubernetes-core-principle-api-server/
https://github.com/feiskyer/kubernetes-handbook/blob/master/zh/components/apiserver.md


源码分析
========
* command github.com/spf13/pflag
* http
* go-restful github.com/emicklei/go-restful
* etcd 