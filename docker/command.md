**docker images**

> https://docs.docker.com/v1.11/engine/reference/commandline/images/

```
[root@yzw kolla]# docker images -f "label=kolla_version"  --format "{{.Repository}}:{{.Tag}}" | grep -E source-base
kolla-test/centos-source-base:1.1.1
[root@yzw kolla]# 
[root@yzw kolla]# docker images -f "label=kolla_version"  -q 
7cff68b428c8
f8140ec6ff6d
[root@yzw kolla]# docker images --format "table {{.Repository}}\t{{.ID}}\t{{.Tag}}"
REPOSITORY                                             IMAGE ID            TAG
kolla-test/centos-source-neutron-server-opendaylight   7cff68b428c8        1.1.1
kolla-test/centos-source-neutron-server-ovn            f8140ec6ff6d        1.1.1

```

**docker system**

> https://docs.docker.com/engine/reference/commandline/system/#description

```
root@yzw-vm:/home/yzw# docker system df 
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              2                   0                   131.3MB             131.3MB (100%)
Containers          0                   0                   0B                  0B
Local Volumes       4                   0                   19.85MB             19.85MB (100%)
Build Cache         0                   0                   0B                  0B
root@yzw-vm:/home/yzw# docker system prune --help

Usage:	docker system prune [OPTIONS]

Remove unused data

Options:
  -a, --all             Remove all unused images not just dangling ones
      --filter filter   Provide filter values (e.g. 'label=<key>=<value>')
  -f, --force           Do not prompt for confirmation
      --volumes         Prune volumes

#running的容器的镜像不会被删，stop的容器和镜像一块被删
[root@kolla ~]# docker system prune -af --filter 'label=kolla_version=5.0.4'
Deleted Containers:
6d597715565466e545c27e23a65da172bb74a909e95f747452d748d7cf072714
...
```

> &lt;none&gt; 镜像叫悬挂 dangling 镜像