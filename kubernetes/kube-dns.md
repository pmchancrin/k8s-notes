kube NDS
--------

Kubernetes DNS pod 中包括 3 个容器：
> * kubedns：kubedns 进程监视 Kubernetes master 中的 Service 和 Endpoint 的变化，并维护内存查找结构来服务DNS请求。
> * dnsmasq：dnsmasq 容器添加 DNS 缓存以提高性能。
> * sidecar：sidecar 容器在执行双重健康检查（针对 dnsmasq 和 kubedns）时提供单个健康检查端点（监听在10054端口）。

DNS pod 具有静态 IP 并作为 Kubernetes 服务暴露出来。该静态 IP 分配后，kubelet 会将使用 --cluster-dns = <dns-service-ip> 标志配置的 DNS 传递给每个容器。
DNS 名称也需要域名。本地域可以使用标志 --cluster-domain = <default-local-domain> 在 kubelet 中配置。
```
[root@node1 nginx]# kubectl get svc --all-namespaces -o wide
NAMESPACE     NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE       SELECTOR
default       kubernetes   10.233.0.1   <none>        443/TCP         5d        <none>
kube-system   kube-dns     10.233.0.3   <none>        53/UDP,53/TCP   13m       k8s-app=kube-dns
```

不管那种部署很是，kubernetes 对外提供的 DNS 服务是一致的。每个 service 都会有对应的 DNS 记录，kubernetes 保存 DNS 记录的格式如下：
> <service_name>.<namespace>.svc.<domain>  

在 pod 中可以通过 *service_name.namespace.svc.domain* 来访问任何的服务，也可以使用缩写 *service_name.namespace*，如果 pod 和 service 在同一个 namespace，甚至可以直接使用 service_name。

> NOTE：正常的 service 域名会被解析成 service vip，而 headless service 域名会被直接解析成背后的 pods ip。

虽然不会经常用到，但是 pod 也会有对应的 DNS 记录，格式是 *pod-ip-address.<namespace>.pod.<domain>*，其中 *pod-ip-address* 为 pod ip 地址的用 - 符号隔开的格式，比如 pod ip 地址是 1.2.3.4 ，那么对应的域名就是 *1-2-3-4.default.pod.cluster.local*。

有一个名称为 "my-service" 的 Service，它在 Kubernetes 集群中名为 "my-ns" 的 Namespace中，为 "my-service.my-ns" 创建了一条 DNS 记录。 在名称为 "my-ns" 的 Namespace 中的 Pod 应该能够简单地通过名称查询找到 "my-service"。 在另一个 Namespace 中的 Pod 必须限定名称为 "my-service.my-ns"。 这些名称查询的结果是 Cluster IP。
```
[root@node1 nginx]# kubectl get svc --all-namespaces -o wide
NAMESPACE     NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE       SELECTOR
default       kubernetes   10.233.0.1   <none>        443/TCP         5d        <none>
kube-system   kube-dns     10.233.0.3   <none>        53/UDP,53/TCP   13m       k8s-app=kube-dns

[root@node1 nginx]# kubectl exec -it  busybox cat /etc/resolv.conf 
nameserver 10.233.0.3
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
[root@node1 nginx]# 

[root@node1 nginx]# kubectl exec -it  busybox -- nslookup kubernetes.default.svc.cluster.local
Server:    10.233.0.3
Address 1: 10.233.0.3 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default.svc.cluster.local
Address 1: 10.233.0.1 kubernetes.default.svc.cluster.local

[root@node1 nginx]# kubectl exec -it  busybox -- nslookup kubernetes
Server:    10.233.0.3
Address 1: 10.233.0.3 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.233.0.1 kubernetes.default.svc.cluster.local

[root@node1 nginx]# kubectl exec -it  busybox -- nslookup kube-dns.kube-system
Server:    10.233.0.3
Address 1: 10.233.0.3 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system
Address 1: 10.233.0.3 kube-dns.kube-system.svc.cluster.local
```

```
apiVersion: v1
kind: Service
metadata:
  name: default-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
  - name: foo # Actually, no port is needed.
    port: 1234
    targetPort: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: default-subdomain
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    name: busybox
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
  labels:
    name: busybox
spec:
  hostname: busybox-2
  subdomain: default-subdomain
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    name: busybox
```
```
[root@node1 ~]# kubectl exec -it  busybox1 -- nslookup default-subdomain
Server:    10.233.0.3
Address 1: 10.233.0.3 kube-dns.kube-system.svc.cluster.local

Name:      default-subdomain
Address 1: 10.233.71.15 busybox-1.default-subdomain.default.svc.cluster.local
Address 2: 10.233.71.16 busybox-2.default-subdomain.default.svc.cluster.local
[root@node1 ~]# kubectl exec -it  busybox1 ping busybox-2.default-subdomain.default.svc.cluster.local
PING busybox-2.default-subdomain.default.svc.cluster.local (10.233.71.16): 56 data bytes
64 bytes from 10.233.71.16: seq=0 ttl=63 time=0.056 ms
64 bytes from 10.233.71.16: seq=1 ttl=63 time=0.052 ms
64 bytes from 10.233.71.16: seq=2 ttl=63 time=0.054 ms
^C
--- busybox-2.default-subdomain.default.svc.cluster.local ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.052/0.054/0.056 ms
```
#### 参考
* https://jimmysong.io/kubernetes-handbook/practice/configuring-dns.html
* http://cizixs.com/2017/04/11/kubernetes-intro-kube-dns
* http://www.cnblogs.com/iiiiher/p/7891713.html
* https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#dns-policy