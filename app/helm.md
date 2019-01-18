# Helm

> * git：https://github.com/kubernetes/helm  
> * doc：https://docs.helm.sh/  
> * ref：https://jimmysong.io/kubernetes-handbook/practice/helm.html  

## 安装

### 客户端 helm

```
[root@kubespray-node-1 ~]# curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  6740  100  6740    0     0   2147      0  0:00:03  0:00:03 --:--:--  2148
[root@kubespray-node-1 ~]# 
[root@kubespray-node-1 ~]# chmod 700 get_helm.sh 
[root@kubespray-node-1 ~]# ./get_helm.sh 
Helm v2.9.1 is already latest
helm installed into /usr/local/bin/helm
Run 'helm init' to configure helm.
[root@kubespray-node-1 ~]# helm version
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
```

### 服务端 tiller

> helm init 下载的地址是：gcr.io/kubernetes-helm/tiller

```
[root@kubespray-node-1 ~]# helm init --skip-refresh -i yinzw/tiller:v2.9.1
Creating /root/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```

## 使用

命令自动补全：

``` 
source <(helm completion bash)
```

命令：

```
helm create xxx
helm install ./
helm list
helm delete
helm search
```

使用：

```
[root@kubespray-node-1 helm]# git clone https://github.com/kubernetes/charts.git
Cloning into 'charts'...
remote: Counting objects: 34086, done.
remote: Total 34086 (delta 0), reused 0 (delta 0), pack-reused 34086
Receiving objects: 100% (34086/34086), 8.66 MiB | 136.00 KiB/s, done.
Resolving deltas: 100% (23722/23722), done.

[root@kubespray-node-1 helm]# cd charts/
[root@kubespray-node-1 charts]# ls
code-of-conduct.md  CONTRIBUTING.md  incubator  LICENSE  OWNERS  PROCESSES.md  README.md  REVIEW_GUIDELINES.md  stable  test

[root@kubespray-node-1 helm]# cd stable/mysql/
[root@kubespray-node-1 mysql]# ls
Chart.yaml  README.md  templates  values.yaml

[root@kubespray-node-1 mysql]# helm package .
Successfully packaged chart and saved it to: /root/helm/charts/stable/mysql/mysql-0.8.2.tgz

[root@kubespray-node-1 mysql]# pwd
/root/helm/charts/stable/mysql
[root@kubespray-node-1 mysql]# ls
Chart.yaml  mysql-0.8.2.tgz  README.md  templates  values.yaml

[root@kubespray-node-1 mysql]# helm search mysql
WARNING: Repo "stable" is corrupt or missing. Try 'helm repo update'.NAME       	CHART VERSION	APP VERSION	DESCRIPTION                                       
local/mysql	0.8.2        	5.7.14     	Fast, reliable, scalable, and easy to use open-...

[root@kubespray-node-1 mysql]# helm repo list
NAME  	URL                                             
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts                    


## install
[root@kubespray-node-1 mysql]# helm install ./mysql-0.8.2.tgz 
[root@kubespray-node-1 mysql]# helm install ./

## 检查配置和模板是否有效
[root@kubespray-node-1 mysql]# helm install --dry-run --debug .
[debug] Created tunnel using local port: '36168'

[debug] SERVER: "127.0.0.1:36168"

[debug] Original chart version: ""
[debug] CHART PATH: /root/helm/charts/stable/mysql

NAME:   impressive-snail
REVISION: 1
RELEASED: Mon Jul  9 18:06:12 2018

[root@kubespray-node-1 mysql]# helm delete --purge ingress-kube-system
[root@kubespray-node-1 mysql]# helm delete --purge ingress-openstack
[root@kubespray-node-1 mysql]# helm delete --purge ingress-ceph
```

