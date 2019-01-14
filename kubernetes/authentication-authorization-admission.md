认证Authenticating & 授权Authorization & 准入Admission Control
==============================================================
账户分为两类：
* User Account：普通用户被假定为由外部独立服务管理。管理员分发私钥，用户存储（如Keystone或Google帐户），甚至包含用户名和密码列表的文件。在这方面，kubernetes没有代表普通用户帐户的对象。无法通过API调用的方式向集群中添加普通用户。用户账号为全局设计的。命名必须在一个集群的所有命名空间中唯一。**这个账户是给人用的。**
* Service Account：service account是由kubernetes API管理的帐户。它们都绑定到了特定的 namespace，并由api-server自动创建，或者通过API调用手动创建。service accoun关联了一套凭证，存储在Secret，这些凭证同时被挂载到pod中，从而允许pod与kubernetes API之间的调用。服务账号是在命名空间里的。**这个账户是给pod中的进程用的。**

概述
----
* k8s通过kube-apiserver组件对外提供REST服务，有两类客户端：普通用户和集群内的pod
* k8s默认https安全端口6443，一个API请求到达该端口后，要经过认证，授权，准入控制，实际API请求。
* K8s默认非安全端口8080（只能本机访问，绑定的是localhost）
* **认证**：对客户端的认证，Authenticaton verifies who you are
* **授权**：对不同用户不同的访问权限，Authorization verifies what you are authorized to do.
* **准入**：Admission Control 有一个准入控制列表，我们可以通过命令行设置选择执行哪几个准入控制器。只有所有的准入控制器都检查通过之后，apiserver 才执行该请求，否则返回拒绝。

认证
----
apiserver认证配置
```
[root@node1 manifests]# cat /etc/kubernetes/manifests/kube-apiserver.manifest 
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
  labels:
    k8s-app: kube-apiserver
    kubespray: v2
  annotations:
    kubespray.etcd-cert/serial: "{'stderr_lines': [], u'changed': True, u'end': u'2018-06-06 22:41:46.202119', 'failed': False, u'stdout': u'B2EDC360B1BF0E77', u'cmd': u'openssl x509 -in /etc/ssl/etcd/ssl/node-node1.pem -noout -serial | cut -d= -f2', u'rc': 0, u'start': u'2018-06-06 22:41:46.185670', u'stderr': u'', u'delta': u'0:00:00.016449', 'stdout_lines': [u'B2EDC360B1BF0E77']}"
    kubespray.apiserver-cert/serial: "D72C56208C860176"
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirst
  containers:
  - name: kube-apiserver
    image: yinzw/hyperkube:v1.7.5_coreos.0
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: 800m
        memory: 2000M
      requests:
        cpu: 100m
        memory: 256M
    command:
    - /hyperkube
    - apiserver
    - --advertise-address=192.168.137.101
    - --etcd-servers=https://192.168.137.101:2379,https://192.168.137.102:2379,https://192.168.137.103:2379
    - --etcd-quorum-read=true
    - --etcd-cafile=/etc/ssl/etcd/ssl/ca.pem
    - --etcd-certfile=/etc/ssl/etcd/ssl/node-node1.pem
    - --etcd-keyfile=/etc/ssl/etcd/ssl/node-node1-key.pem
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
> * **--client-ca-file:** 指定CA根证书文件为 */etc/kubernetes/ssl/ca.pem* ，内置CA公钥用于验证某证书是否是CA签发的证书.
> * **--tls-private-key-file:**  指定ApiServer私钥文件为 */etc/kubernetes/ssl/apiserver-key.pem* .
> * **--tls-cert-file:** 指定ApiServer证书文件为 */etc/kubernetes/ssl/apiserver.pem*
> * **--token-auth-file** 静态token认证
> * **--basic-auth-file** 用户密码认证

* user account认证有三种：CA证书，token，Base认证，可以配多种
* sa 认证
* 开启了https认证，访问集群，提示unauthorized


#### CA认证
```
[root@node3 kubernetes]# curl https://node1:6443/api/v1/nodes
Unauthorized
[root@node3 kubernetes]# curl https://node1:6443/api/v1/nodes --cert /etc/kubernetes/ssl/node-node3.pem --key /etc/kubernetes/ssl/node-node3-key.pem
{
  "kind": "NodeList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/nodes",
    "resourceVersion": "434825"
  },
  "items": [
    {
      "metadata": {
        "name": "node1",
        "selfLink": "/api/v1/nodes/node1",
...

[root@node3 ssl]# curl https://node1:6443/api/v1/nodes --cert /etc/kubernetes/ssl/kube-proxy-node3.pem --key /etc/kubernetes/ssl/kube-proxy-node3-key.pem 
{
  "kind": "NodeList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/nodes",
    "resourceVersion": "435193"
  },
  "items": [
    {
      "metadata": {
        "name": "node1",
        "selfLink": "/api/v1/nodes/node1",
        "uid": "de4e2334-699b-11e8-ba8f-0800277b75c4",
        "resourceVersion": "435188",
        "creationTimestamp": "2018-06-06T15:11:20Z",
...
[root@node2 kubernetes]# curl https://node1:6443/api/v1/nodes --cert /etc/kubernetes/ssl/kube-scheduler.pem --key /etc/kubernetes/ssl/kube-scheduler-key.pem
{
  "kind": "NodeList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/nodes",
    "resourceVersion": "435809"
  },
  "items": [
    {
      "metadata": {
        "name": "node1",
        "selfLink": "/api/v1/nodes/node1",


```
* kubelet 的配置
```
[root@node3 kubernetes]# cat node-kubeconfig.yaml 
apiVersion: v1
kind: Config
clusters:
- name: local
  cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    server: https://localhost:6443
users:
- name: kubelet
  user:
    client-certificate: /etc/kubernetes/ssl/node-node3.pem
    client-key: /etc/kubernetes/ssl/node-node3-key.pem
contexts:
- context:
    cluster: local
    user: kubelet
  name: kubelet-cluster.local
current-context: kubelet-cluster.local

```
* kubeproxy 的配置
```
[root@node3 kubernetes]# cat kube-proxy-kubeconfig.yaml 
apiVersion: v1
kind: Config
clusters:
- name: local
  cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    server: https://localhost:6443
users:
- name: kube-proxy
  user:
    client-certificate: /etc/kubernetes/ssl/kube-proxy-node3.pem
    client-key: /etc/kubernetes/ssl/kube-proxy-node3-key.pem
contexts:
- context:
    cluster: local
    user: kube-proxy
  name: kube-proxy-cluster.local
current-context: kube-proxy-cluster.local
```
* kube-scheduler 的配置
```
[root@node2 kubernetes]# cat kube-scheduler-kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
- name: local
  cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    server: https://127.0.0.1:6443
users:
- name: kube-scheduler
  user:
    client-certificate: /etc/kubernetes/ssl/kube-scheduler.pem
    client-key: /etc/kubernetes/ssl/kube-scheduler-key.pem
contexts:
- context:
    cluster: local
    user: kube-scheduler
  name: kube-scheduler-cluster.local
current-context: kube-scheduler-cluster.local
```
* 手动生成证书和key，可以在集群其他节点上使用
```
[root@node1 ssl]# openssl genrsa -out test-key.pem 2048
Generating RSA private key, 2048 bit long modulus
......+++
...+++
e is 65537 (0x10001)
[root@node1 ssl]# openssl req -new -key test-key.pem -out test.csr -subj "/CN=test" 
[root@node1 ssl]# 
[root@node1 ssl]# openssl x509 -req -in test.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out test.pem -days 3650 -extensions v3_req 
Signature ok
subject=/CN=test
Getting CA Private Key
[root@node1 ssl]# scp test* node3:/root/test
root@node3's password: 
test.csr                                                                                                                                                                       100%  883     1.7MB/s   00:00    
test-key.pem                                                                                                                                                                   100% 1679     3.4MB/s   00:00    
test.pem
[root@node3 test]# curl https://node1:6443/api/v1/nodes --cert ./test.pem --key ./test-key.pem
{
  "kind": "NodeList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/nodes",
    "resourceVersion": "439941"
  },

```

* 将证书拷贝到非集群节点上，报错
```
[root@node test]# curl https://node1:6443/api/v1/nodes --cert ./test.pem --key ./test-key.pem
curl: (60) Peer's certificate has an invalid signature.
More details here: http://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
```
需要将ca根证书拷贝过来
```
[root@node test]# curl https://node1:6443/api/v1/nodes --cert ./test.pem --key ./test-key.pem --cacert ./ca.pem 
{
  "kind": "NodeList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/nodes",
    "resourceVersion": "441981"
  },
  "items": [
    {
      "metadata": {
        "name": "node1",
        "selfLink": "/api/v1/nodes/node1",
```
为什么集群节点不需要ca根证书？
> 因为集群ca证书已经在ca-bundle.crt里面了，curl时不用带cacert。
> ==但是将ca证书加到ca-bundle里面了也不行，不知道为啥？==

```
[root@node3 ssl]# cd /etc/pki/tls/certs/
[root@node3 certs]# ls
ca-bundle.crt  ca-bundle.trust.crt  make-dummy-cert  Makefile  renew-dummy-cert
[root@node3 certs]# cat ca-bundle.crt | grep MIIC9zCCAd+gAwIBAgIJAP+vWZXVLCBqMA0GCSqGSIb3DQEBCwUAMBIxEDAOBgNV
MIIC9zCCAd+gAwIBAgIJAP+vWZXVLCBqMA0GCSqGSIb3DQEBCwUAMBIxEDAOBgNV

```
#### token 认证
和静态密码一样，这种方法也是名存实亡，不推荐使用。
* 查看token  
token,user,uid,"group1,group2,group3"
> ==NOTE：如果该静态token文件更改的话，需要重启apiserver==
```
[root@node1 tokens]# ls
known_tokens.csv  system:kubectl-node1.token  system:kubectl-node2.token  system:kubelet-node1.token  system:kubelet-node2.token  system:kubelet-node3.token
[root@node1 tokens]# cat known_tokens.csv 
UPVzqw3LVaqpPCrijfuc2rwKadjvGNUq,system:kubectl-node1,system:kubectl-node1
uPjAGKBVwFCN9Iyam7Y3150tgrGmXekm,system:kubectl-node2,system:kubectl-node2
fPv22M1DYnVDXpXKPPlzMaZDLgmGZCKp,system:kubelet-node1,system:kubelet-node1
GwpmzUtoIbODKavaNXoNqf4P8XcaUz0K,system:kubelet-node2,system:kubelet-node2
MjkEgEkG8L9vcmy1zEdF68OXRkriyONL,system:kubelet-node3,system:kubelet-node3
[root@node1 tokens]# cat system\:kubectl-node1.token 
UPVzqw3LVaqpPCrijfuc2rwKadjvGNUq

```
* 访问1
```
[root@node3 test]# curl -k --header "Authorization: Bearer UPVzqw3LVaqpPCrijfuc2rwKadjvGNUq" https://node1:6443/api/v1/nodes
{
  "kind": "NodeList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/nodes",
    "resourceVersion": "446733"
  },
  "items": [
    {
      "metadata": {
        "name": "node1",

```
* 访问2
```
[root@node3 test]# ./kubectl --server=https://node1:6443  get nodes
Please enter Username: admin
Please enter Password: ****
error: You must be logged in to the server (the server has asked for the client to provide credentials (get nodes))
[root@node3 test]# ./kubectl --server=https://node1:6443 --token=UPVzqw3LVaqpPCrijfuc2rwKadjvGNUq --insecure-skip-tls-verify=true get nodes
NAME      STATUS    AGE       VERSION
node1     Ready     20d       v1.7.5+coreos.0
node2     Ready     20d       v1.7.5+coreos.0
node3     Ready     20d       v1.7.5+coreos.0
[root@node3 test]# 

```
#### password file认证
这种方式很不灵活，也不安全，可以说名存实亡，不推荐使用。
* 查看

格式: password,user,uid,"group1,group2,group3"
```
[root@node1 users]# ls
known_users.csv
[root@node1 users]# cat known_users.csv 
ekxj0taObdTNTXc,kube,admin,"system:masters"
```
* 使用
```
[root@node3 test]# ./kubectl --server=https://node1:6443  get nodes
Please enter Username: kube
Please enter Password: ***************
NAME      STATUS    AGE       VERSION
node1     Ready     20d       v1.7.5+coreos.0
node2     Ready     20d       v1.7.5+coreos.0
node3     Ready     20d       v1.7.5+coreos.0
[root@node3 test]# 

```
#### 参考
* https://kubernetes.feisky.xyz/zh/plugins/auth.html#%20%E8%AE%A4%E8%AF%81

#### SA



授权
----

* AlwaysDeny：表示拒绝所有的请求，该配置一般用于测试
* AlwaysAllow：表示接收所有请求，如果集群不需要授权，则可以采取这个策略
* ABAC：基于属性的访问控制，表示基于配置的授权规则去匹配用户请求，判断是否有权限。Beta 版本
* RBAC：基于角色的访问控制，允许管理员通过 api 动态配置授权策略。Beta 版本


* 参考   
>* https://jimmysong.io/kubernetes-handbook/guide/kubectl-user-authentication-authorization.html

#### RBAC
> * Role   
某个ns下的资源
> * ClusterRole  
集群范围内的资源(node, endpoint pods)
> * RoleBinding  
> * ClusterRoleBinding  
将role中定义的权限授予users、groups、service accounts

* 参考  
>*  https://mritd.me/2018/03/20/use-rbac-to-control-kubectl-permissions/  
>*  https://blog.qikqiak.com/post/add-authorization-for-kubernetes-dashboard/

* 配置
```
[root@node1 manifests]# cat kube-apiserver.manifest 
...
spec:
 command:
- --authorization-mode=Node,RBAC

```
* 查询roles clusterroles
```
[root@node1 manifests]# kubectl get roles --all-namespaces
NAMESPACE     NAME                                             AGE
kube-public   system:controller:bootstrap-signer               22d
kube-system   extension-apiserver-authentication-reader        22d
kube-system   kubernetes-dashboard-minimal                     22d
kube-system   system::leader-locking-kube-controller-manager   22d
kube-system   system::leader-locking-kube-scheduler            22d
kube-system   system:controller:bootstrap-signer               22d
kube-system   system:controller:cloud-provider                 22d
kube-system   system:controller:token-cleaner                  22d

[root@node1 manifests]# kubectl get clusterroles
NAME                                                                   AGE
admin                                                                  22d
calico-node                                                            22d
cluster-admin                                                          22d
cluster-proportional-autoscaler                                        22d
edit                                                                   22d
kubernetes-dashboard-anonymous                                         22d
system:aggregate-to-admin                                              22d
system:aggregate-to-edit                                               22d
system:aggregate-to-view                                               22d
system:auth-delegator                                                  22d
...
```
* 查询 rolebinding clusterrolebinding
> RoleBinding把Role绑定到账户主体Subject，让Subject继承Role所在namespace下的权限。 ClusterRoleBinding把ClusterRole绑定到Subject，让Subject集成ClusterRole在整个集群中的权限。
账户主体Subject在这里还是叫“用户”吧，包含组group，用户user和ServiceAccount。
```
[root@node1 manifests]# kubectl get rolebinding --all-namespaces
NAMESPACE     NAME                                             AGE
kube-public   system:controller:bootstrap-signer               22d
kube-system   kubernetes-dashboard-minimal                     22d
kube-system   system::leader-locking-kube-controller-manager   22d
kube-system   system::leader-locking-kube-scheduler            22d
kube-system   system:controller:bootstrap-signer               22d
kube-system   system:controller:cloud-provider                 22d
kube-system   system:controller:token-cleaner                  22d
[root@node1 manifests]# kubectl get clusterrolebinding --all-namespaces
NAME                                                   AGE
calico-node                                            22d
cluster-admin                                          22d
cluster-proportional-autoscaler                        22d
kubernetes-dashboard-anonymous                         22d
kubespray:system:node                                  22d
system:aws-cloud-provider                              22d
system:basic-user                                      22d
system:controller:attachdetach-controller              22d
...
```


准入
----
当前可配置的准入控制器主要有：

* AlwaysAdmit：允许所有请求
* AlwaysDeny：拒绝所有请求
* AlwaysPullImages：在启动容器之前总是去下载镜像
* ServiceAccount：将 secret 信息挂载到 pod 中，比如 service account token，registry key 等
* ResourceQuota 和 LimitRanger：实现配额控制
* SecurityContextDeny：禁止创建设置了 Security Context 的 pod
*

通过kubeconfig配置访问多集群
----
* kubeconfig介绍  
kubeconfig文件用于组织关于集群、用户、命名空间和认证机制的信息。命令行工具kubectl从 kubeconfig文件中得到它要选择的集群以及跟集群api-server交互的信息。  
默认情况下，kubectl会从$HOME/.kube目录下查找文件名为config的文件。可以通过设置环境变量KUBECONFIG或者通过设置--kubeconfig去指定其它kubeconfig文件。
* Context  
context指定了kubectl命令运行的上下文环境，kubectl与当前context中指定的集群和命名空间进行通信，并且使用当前context中包含的用户凭证。  
每个context都是一个由（集群、命名空间、用户）描述的三元组。可以使用kubectl config use-context去设置当前的context。
```
[root@node1 ~]#  kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://192.168.137.101:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
[root@node1 ~]# kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin  
```



参考
----
* https://kubernetes.io/docs/reference/access-authn-authz/authentication/
* https://jimmysong.io/kubernetes-handbook/guide/authentication.html
* https://zhangchenchen.github.io/2017/08/17/kubernetes-authentication-authorization-admission-control/
* http://www.cnblogs.com/breg/p/5923604.html
* https://www.kubernetes.org.cn/1995.html
* https://jimmysong.io/kubernetes-handbook/guide/managing-tls-in-a-cluster.html
* https://jimmysong.io/kubernetes-handbook/practice/create-tls-and-secret-key.html
* https://jimmysong.io/kubernetes-handbook/guide/auth-with-kubeconfig-or-token.html
* https://blog.frognew.com/2017/04/kubernetes-1.6-rbac.html