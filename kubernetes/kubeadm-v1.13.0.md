# install and enable docker-ce

> https://docs.docker.com/install/linux/docker-ce/centos/#install-docker-ce-1

## setup http_proxy

global proxy is only used to install app by yum.
```
$ export http_proxy=http://192.168.137.1:7777
$ export https_proxy=http:/192.168.137.1:7777
```
## setup docker proxy

```
$ vim /usr/lib/systemd/system/docker.service
[Service]
...
Environment="HTTP_PROXY=http://192.168.137.1:7777/" "HTTPS_PROXY=http://192.168.137.1:7777" "NO_PROXY=localhost,127.0.0.1,registry.docker-cn.com"

$ systemctl daemon-reload
$ systemctl restart docker
$ systemctl show --property=Environment docker
Environment=HTTP_PROXY=http://192.168.137.1:7777/ HTTPS_PROXY=https://192.168.137.1:7777 NO_PROXY=localhost,127.0.0.1,registry.docker-cn.com

$ docker pull k8s.gcr.io/kube-apiserver-amd64:v1.13.0
v1.13.0: Pulling from kube-apiserver-amd64
73e3e9d78c61: Pull complete 
bef6770497e3: Pull complete 
Digest: sha256:f88cb526ae4346a682d759397c085d6aba829748b862db8feeca5ff99330482f
Status: Downloaded newer image for k8s.gcr.io/kube-apiserver-amd64:v1.13.0
```

# install kubectl kubeadm kubelet

> https://kubernetes.io/docs/setup/independent/install-kubeadm/

```
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

$ setenforce 0
$ sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

$ systemctl stop firewalld
$ systemctl disable firewalld

$ yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

$ systemctl enable kubelet && systemctl start kubelet
```

# install master

## kubeadm init

```
[root@node11 ~]# kubeadm init --pod-network-cidr=111.111.0.0/16
[init] Using Kubernetes version: v1.13.0
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "node11" could not be reached
	[WARNING Hostname]: hostname "node11": lookup node11 on 192.168.137.1:53: server misbehaving
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [node11 localhost] and IPs [192.168.137.111 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [node11 localhost] and IPs [192.168.137.111 127.0.0.1 ::1]
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [node11 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.137.111]
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 25.513024 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "node11" as an annotation
[mark-control-plane] Marking the node node11 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node node11 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: o86g4m.6neumrafpc9n0zcw
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.137.111:6443 --token o86g4m.6neumrafpc9n0zcw --discovery-token-ca-cert-hash sha256:9b1f749a9dd839529b995bbd77576daaa3e3a4edcde2e234f4fd379428fe4341

```

```
[root@node11 ~]# mkdir -p $HOME/.kube
[root@node11 ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@node11 ~]# chown $(id -u):$(id -g) $HOME/.kube/config

```


### issue
```
[root@node11 ~]# kubeadm init --pod-network-cidr=111.111.0.0/16
[init] Using Kubernetes version: v1.13.0
[preflight] Running pre-flight checks
	[WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
	[WARNING HTTPProxy]: Connection to "https://192.168.137.111" uses proxy "http://192.168.137.1:7777". If that is not intended, adjust your proxy settings
	[WARNING HTTPProxyCIDR]: connection to "10.96.0.0/12" uses proxy "http://192.168.137.1:7777". This may lead to malfunctional cluster setup. Make sure that Pod and Services IP ranges specified correctly as exceptions in proxy configuration
	[WARNING HTTPProxyCIDR]: connection to "111.111.0.0/16" uses proxy "http://192.168.137.1:7777". This may lead to malfunctional cluster setup. Make sure that Pod and Services IP ranges specified correctly as exceptions in proxy configuration
	[WARNING Hostname]: hostname "node11" could not be reached
	[WARNING Hostname]: hostname "node11": lookup node11 on 192.168.137.1:53: server misbehaving
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR NumCPU]: the number of available CPUs 1 is less than the required 2
	[ERROR Swap]: running with swap on is not supported. Please disable swap
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`

### resolve
# 1 add cpu
# 2 swapoff -a && sed -i '/swap/d' /etc/fstab
```

## install calico

### Installing with the Kubernetes API datastore—50 nodes or less
> https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network  
> https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/calico  

```
# kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
# kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```
The default *CALICO_IPV4POOL_CIDR* in *calico.yaml* is *"192.168.0.0/16"*, because we pass *--pod-network-cidr=192.168.0.0/16* to kubeadm by default.  
if *pod-network-cidr* is not default value, you need to change the value of *CALICO_IPV4POOL_CIDR* in *calico.yaml*.
### Installing with the Kubernetes API datastore—more than 50 nodes  
> https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/calico  

```
# kubectl get pods --all-namespaces
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   calico-node-xjhgm                2/2     Running   0          4m38s
kube-system   coredns-86c58d9df4-hm94l         1/1     Running   0          3h32m
kube-system   coredns-86c58d9df4-mztp6         1/1     Running   0          3h32m
kube-system   etcd-node11                      1/1     Running   0          3h31m
kube-system   kube-apiserver-node11            1/1     Running   0          3h31m
kube-system   kube-controller-manager-node11   1/1     Running   0          3h31m
kube-system   kube-proxy-nvg9v                 1/1     Running   0          3h32m
kube-system   kube-scheduler-node11            1/1     Running   0          3h31m
# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
node11   Ready    master   3h51m   v1.13.0

```

# install nodes
on nodes
```
# kubeadm join 192.168.137.111:6443 --token o86g4m.6neumrafpc9n0zcw --discovery-token-ca-cert-hash sha256:9b1f749a9dd839529b995bbd77576daaa3e3a4edcde2e234f4fd379428fe4341
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "node12" could not be reached
	[WARNING Hostname]: hostname "node12": lookup node12 on 114.114.114.114:53: no such host
[discovery] Trying to connect to API Server "192.168.137.111:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.137.111:6443"
[discovery] Requesting info from "https://192.168.137.111:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.137.111:6443"
[discovery] Successfully established connection with API Server "192.168.137.111:6443"
[join] Reading configuration from the cluster...
[join] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.13" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "node12" as an annotation

```

## issue 

### failed to request cluster info

```
# kubeadm join 192.168.137.111:6443 --token o86g4m.6neumrafpc9n0zcw --discovery-token-ca-cert-hash sha256:9b1f749a9dd839529b995bbd77576daaa3e3a4edcde2e234f4fd379428fe4341
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "node12" could not be reached
	[WARNING Hostname]: hostname "node12": lookup node12 on 114.114.114.114:53: no such host
	[WARNING HTTPProxy]: Connection to "https://192.168.137.111" uses proxy "http://192.168.137.1:7777". If that is not intended, adjust your proxy settings
[discovery] Trying to connect to API Server "192.168.137.111:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.137.111:6443"
[discovery] Failed to request cluster info, will try again: [Get https://192.168.137.111:6443/api/v1/namespaces/kube-public/configmaps/cluster-info: Forbidden]
[discovery] Failed to request cluster info, will try again: [Get https://192.168.137.111:6443/api/v1/namespaces/kube-public/configmaps/cluster-info: Forbidden]
```

**resolve**

> https://github.com/kubernetes/kubeadm/issues/299  

```
# unset http_proxy
# unset https_proxy
```

### calico node CrashLoopBackOff
```
# kubectl get pods --all-namespaces
NAMESPACE     NAME                             READY   STATUS             RESTARTS   AGE
kube-system   calico-node-4mq2w                1/2     CrashLoopBackOff   6          16m
kube-system   calico-node-m6s96                1/2     CrashLoopBackOff   7          19m
# kubectl describe pod calico-node-m6s96 -n kube-system
Name:               calico-node-m6s96
...
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  21m                   default-scheduler  Successfully assigned kube-system/calico-node-m6s96 to node12
  Normal   Pulling    21m                   kubelet, node12    pulling image "quay.io/calico/cni:v3.3.2"
  Normal   Started    16m                   kubelet, node12    Started container
  Normal   Pulled     16m                   kubelet, node12    Successfully pulled image "quay.io/calico/cni:v3.3.2"
  Normal   Created    16m                   kubelet, node12    Created container
  Normal   Created    15m (x2 over 21m)     kubelet, node12    Created container
  Normal   Started    15m (x2 over 21m)     kubelet, node12    Started container
  Normal   Pulled     15m (x2 over 21m)     kubelet, node12    Container image "quay.io/calico/node:v3.3.2" already present on machine
  Normal   Killing    15m                   kubelet, node12    Killing container with id docker://calico-node:Container failed liveness probe.. Container will be killed and recreated.
  Warning  Unhealthy  14m (x7 over 16m)     kubelet, node12    Liveness probe failed: Get http://localhost:9099/liveness: dial tcp [::1]:9099: connect: connection refused
  Warning  Unhealthy  6m14s (x51 over 16m)  kubelet, node12    Readiness probe failed: calico/node is not ready: felix is not ready: Get http://localhost:9099/readiness: dial tcp [::1]:9099: connect: connection refused
  Warning  BackOff    66s (x30 over 9m22s)  kubelet, node12    Back-off restarting failed container
```

**resolve**

not resolved yet.   
