# Install Docker

> docker-io 是以前早期的版本，版本号是 1.*，而 docker-ce 是新的版本，分为社区版 docker-ce 和企业版 docker-ee，版本号是 17.* 。
Docker CE 17.03，可理解为Docker 1.13.1的Bug修复版本。因此，从Docker 1.13升级到Docker CE 17.03风险相对是较小的。

## 官网install

**install docker on Ubuntu**
> https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1

**install docker on Centos**
> https://docs.docker.com/install/linux/docker-ce/centos/#install-docker-ce-1

```
[root@master]# yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
[root@master]# yum-config-manager --add-repo  https://download.docker.com/linux/centos/docker-ce.repo
Loaded plugins: fastestmirror
adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
grabbing file https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
Could not fetch/save url https://download.docker.com/linux/centos/docker-ce.repo to file /etc/yum.repos.d/docker-ce.repo: [Errno 12] Timeout on https://download.docker.com/linux/centos/docker-ce.repo: (28, 'Operation timed out after 30001 milliseconds with 0 out of 0 bytes received')
```

> **Note:** 由于有GW的缘故，一般都会install 失败。

## 其他源install

**清华源 install**
> https://mirror.tuna.tsinghua.edu.cn/help/docker-ce/

**aliyuan install**
> https://blog.csdn.net/doegoo/article/details/80062132

```
[root@master]# sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

## Docker Registry

**aliyuan registry**

> https://cr.console.aliyun.com/?spm=a2c4e.11153940.blogcont29941.9.4cc569d6IVqmDa#/accelerator

```
[root@master]# sudo mkdir -p /etc/docker
[root@master]# sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://q89qk1rw.mirror.aliyuncs.com"]
}
EOF
[root@master]# sudo systemctl daemon-reload
[root@master]# sudo systemctl restart docker
```

**Docker Hub cn**

```
[root@master]# sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
EOF
[root@master]# sudo systemctl daemon-reload
[root@master]# sudo systemctl restart docker
```

**Private Registry**

```
[root@master]# cat /etc/docker/daemon.json 
{
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "insecure-registries": ["192.0.0.0/8"]
}

# or

[root@master]# cat /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd --insecure-registry 192.168.137.100:30060
```

## Issues

**Issue 1**

```
[root@master]# yum install docker-ce-17.03.1.ce
Error: Package: docker-ce-17.03.1.ce-1.el7.centos.x86_64 (docker-ce-stable)
           Requires: docker-ce-selinux >= 17.03.1.ce-1.el7.centos
           Available: docker-ce-selinux-17.03.0.ce-1.el7.centos.noarch (docker-ce-stable)
               docker-ce-selinux = 17.03.0.ce-1.el7.centos
           Available: docker-ce-selinux-17.03.1.ce-1.el7.centos.noarch (docker-ce-stable)
               docker-ce-selinux = 17.03.1.ce-1.el7.centos
           Available: docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch (docker-ce-stable)
               docker-ce-selinux = 17.03.2.ce-1.el7.centos
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest
```

解决：

```
需要先安装：
yum install http://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm

再安装：
yum install docker-ce-17.03.2.ce
```
