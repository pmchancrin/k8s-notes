# calico

## install 

> https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/

# calicoctrl

## install 

installing calicoctl as a binary, a container or a K8S pod.

> https://docs.projectcalico.org/v3.3/usage/calicoctl/install

```
# curl -O -L https://github.com/projectcalico/calicoctl/releases/download/v3.3.2/calicoctl
# chmod +x calicoctl
```

## configue

> https://docs.projectcalico.org/v3.3/usage/calicoctl/configure/

### configuration file

- */etc/calico/calicoctl.cfg*  
- *--config xxx*

with ETCD datastore

```
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  etcdEndpoints: https://etcd1:2379,https://etcd2:2379,https://etcd3:2379
  etcdKeyFile: /etc/calico/key.pem
  etcdCertFile: /etc/calico/cert.pem
  etcdCACertFile: /etc/calico/ca.pem
```

with the Kubernetes API datastore

```
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "kubernetes"
  kubeconfig: "~/.kube/config"
```

### environment variables

with ETCD datastore

```
# export ETCD_ENDPOINTS=http://localhost:6666
or 
# ETCD_ENDPOINTS=http://localhost:6666 ./calicoctl node status
```


with the Kubernetes API datastore

```
# export CALICO_DATASTORE_TYPE=kubernetes
# export CALICO_KUBECONFIG=~/.kube/config
or 
# DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl get nodes
```

## commands

> https://docs.projectcalico.org/v3.3/reference/calicoctl/commands/

```
# ./calicoctl node status
# ./calicoctl get node
# ./calicoctl get ipPool -o wide
```
