# Issue
> docker run 一个centos镜像，当通过non-root用户exec进入容器后，su切换到root用户时，不知道root密码。

```
yzw@yzw-vm:~$ docker run -it -d centos:7.4.1708 
c091aee509c85153f738815618e6fd8eec20d2795104a687b2f2a5b222161a5f
yzw@yzw-vm:~$ docker exec -it -u 1000 c091aee50 bash
bash-4.2$ id
uid=1000 gid=0(root) groups=0(root)
bash-4.2$ su
Password: 
su: Authentication failure
```

# UID and GID

Docker 默认没有开启User Namespace，Docker和Host公用一套UID/GID。

## Example 1
运行一个centos容器：
```
yzw@yzw-vm:~$ docker run -it -d centos:7.4.1708 
c091aee509c85153f738815618e6fd8eec20d2795104a687b2f2a5b222161a5f
```

在容器里面创建个test用户，UID=1000
```
yzw@yzw-vm:~$ docker exec -it  c091aee50 bash
[root@c091aee509c8 /]# useradd test
[root@c091aee509c8 /]# cat /etc/passwd | grep test
test:x:1000:1000::/home/test:/bin/bash
```

使用test用户进入容器，跑个sleep
```
yzw@yzw-vm:~$ docker exec -it -u test c091aee509 bash
[test@c091aee509c8 /]$ sleep 1000
```

使用默认root进入容器，sleep进程的User是test,UID=1000
```
yzw@yzw-vm:~$ docker exec -it c091aee509 bash
[root@c091aee509c8 /]# ps aux | grep -v grep | grep sleep
test        97  0.0  0.0   4328   716 pts/2    S+   07:30   0:00 sleep 1000
```

host主机上，sleep进程的User是yzw，因为yzw的UID也是1000
```
yzw@yzw-vm:~$ ps aux | grep -v grep | grep sleep
yzw       4121  0.0  0.0   4328   716 pts/2    S+   15:30   0:00 sleep 1000

yzw@yzw-vm:~$ id
uid=1000(yzw) gid=1000(yzw) groups=1000(yzw),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lpadmin),126(sambashare),998(docker),999(vboxsf)
```

## Example 2

在容器里面创建个test01用户，UID=1001
```
[root@c091aee509c8 /]# useradd test02
[root@c091aee509c8 /]# cat /etc/passwd | grep test02
test02:x:1001:1001::/home/test02:/bin/bash
[root@c091aee509c8 /]# cat /etc/group | grep test02
test02:x:1001:
```

使用test01进入容器，跑个sleep
```
yzw@yzw-vm:~$ docker exec -it -u test02 c091aee509 bash
[test02@c091aee509c8 /]$ id
uid=1001(test02) gid=1001(test02) groups=1001(test02)
[test02@c091aee509c8 /]$ sleep 1000
```

使用默认root进入容器，sleep进程的User是test02,UID=1001
```
[root@c091aee509c8 /]# ps aux | grep -v grep | grep sleep
test02     129  0.0  0.0   4328   612 pts/2    S+   01:29   0:00 sleep 1000
```

host主机上，sleep进程的User是1001
```
yzw@yzw-vm:~$ ps aux | grep -v grep | grep sleep
1001   6438  0.0  0.0   4328   612 pts/2    S+   09:29   0:00 sleep 1000
```

## Example 3

使用一个没有使用的UID登陆容器,跑个sleep：
```
# 只指定了UID，所有GID默认为0，可以通过-u <name|uid>[:<group|gid>]来指定GID。
yzw@yzw-vm:~$ docker exec -it -u 1003 c091aee509 sh
sh-4.2$ id
uid=1003 gid=0(root) groups=0(root)
sh-4.2$ sleep 1000
```

用root登陆容器查看：
```
[root@c091aee509c8 /]# ps aux | grep -v grep | grep sleep
1003       198  0.0  0.0   4328   660 pts/2    S+   01:48   0:00 sleep 1000
[root@c091aee509c8 /]# 
[root@c091aee509c8 /]# 
[root@c091aee509c8 /]# cat /etc/passwd | grep 1003
[root@c091aee509c8 /]# cat /etc/group | grep 1003
```

Host上查看sleep进程：
```
yzw@yzw-vm:~$ ps aux | grep -v grep | grep sleep
1003      7399  0.0  0.0   4328   660 pts/2    S+   09:48   0:00 sleep 1000
yzw@yzw-vm:~$ cat /etc/passwd | grep 1003
yzw@yzw-vm:~$ cat /etc/group | grep 1003
```


## Dockerfile指定用户
```
FROM ubuntu
RUN groupadd -g 999 appuser && \
    useradd -r -u 999 -g appuser appuser
USER appuser
```

# 安全

容器里面和主机root的权限是一样的，那安全问题怎么解决？Docker目前提供了2种方案：privileged和user namespace。

## privileged

Docker默认时关闭特权模式的，需要的时候直接 `docker run --privileged=true`
```
$ docker run --help
--privileged                     Give extended privileges to this container

```
这样全部的capabilities都被试能。
> Full container capabilities (–privileged)
>
> The –privileged flag gives all capabilities to the container, and it also lifts all the limitations enforced by the device cgroup controller. In other words, the container can then do almost everything that the host can do. This flag exists to allow special use-cases, like running Docker within Docker.

还可以通过 `docker run --cap-add/drop` 加和减指定的capabilities
```
$ docker run --help
--cap-add list                   Add Linux capabilities
--cap-drop list                  Drop Linux capabilities

```

Docker 默认不开启特权模式，默认只支持了一些基本的capabilities，从而限制了容器里面root用户权限。  
默认支持的有：CAP_CHOWN 、 CAP_DAC_OVERRIDE 、 CAP_FSETID 、 CAP_MKNOD 、 FOWNER 、 NET_RAW 、 SETGID 、 SETUID 、 SETFCAP 、 SETPCAP 、 NET_BIND_SERVICE 、 SYS_CHROOT 、 KILL 和 AUDIT_WRITE。  
在这 14 项中几乎没有一项涉及到系统管理权限，比如 Docker 容器的 root 用户不具备 CAP_SYS_ADMIN，磁盘限额操作、mount 操作、创建进程新命名空间等均无法实现；比如由于没有 CAP_NET_ADMIN，网络方面的配置管理也将受到管制。

* CAP_SYS_ADMIN：CAP_SYS_ADMIN  实现一系列的系统管理权限，比如实现磁盘配额的 quotactl，实现文件系统挂载的 mount 权限；比如在 fork 子进程时，通过 clone 和 unshare 系统调用，使用 CLONE_* 的 flag  参数来为子进程创建新的 namespaces；比如实现各种特权块设备以及文件系统的 ioctl 操作等。
* CAP_NET_ADMIN：CAP_NET_ADMIN  实现一系列的网络管理权限，比如网络设备的配置，IP 防火墙，IP 伪装以及统计等功能；比如路由表的修改，TOS 的配置，混杂模式的配置等。
* CAP_SETUID：CAP_SETUID  有能力对进程 UID 做出任何管控。
* CAP_SYS_MODULE：CAP_SYS_MODULE  帮助 root 用户加载或者卸载相应的 Linux 内核模块。
* CAP_SYS_NICE：CAP_SYS_NICE  有能力对任意进程修改其 NICE 值，同时支持对任何进程设置调度策略与优先级，还有在进程的 CPU 亲和性以及 I/O 调度方面有相应的配置权限

## User Namespace
开启：
```
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}

$ systemctl restart docker
```
验证
```
# 开启后系统创建了一个dockremap的用户
yzw@yzw-vm:~/code/ns$ id dockremap
uid=125(dockremap) gid=130(dockremap) groups=130(dockremap)
yzw@yzw-vm:~/code/ns$ sudo cat /etc/group | grep dockre
dockremap:x:130:
yzw@yzw-vm:~/code/ns$ sudo cat /etc/passwd | grep dockre
dockremap:x:125:130::/home/dockremap:/bin/false
# 新建了一个目录165536.165536,165536是dockremap映射出来的一个uid
yzw@yzw-vm:~/code/ns$ sudo ls -ld  /var/lib/docker/165536.165536 
drwx------ 14 165536 165536 4096 Dec 29 00:39 /var/lib/docker/165536.165536
# 该目录和/var/lib/docker目录基本一样，用户隔离的文件都放这
yzw@yzw-vm:~/code/ns$ sudo ls -l  /var/lib/docker/165536.165536 
total 48
drwx------ 2 root   root   4096 Dec 29 00:39 builder
drwx------ 4 root   root   4096 Dec 29 00:39 buildkit
drwx------ 2 165536 165536 4096 Dec 29 00:39 containers
drwx------ 3 root   root   4096 Dec 29 00:39 image
drwxr-x--- 3 root   root   4096 Dec 29 00:39 network
drwx------ 3 165536 165536 4096 Dec 29 00:39 overlay2
drwx------ 4 root   root   4096 Dec 29 00:39 plugins
drwx------ 2 root   root   4096 Dec 29 00:39 runtimes
drwx------ 2 root   root   4096 Dec 29 00:39 swarm
drwx------ 2 165536 165536 4096 Dec 29 00:39 tmp
drwx------ 2 root   root   4096 Dec 29 00:39 trust
drwx------ 2 165536 165536 4096 Dec 29 00:39 volumes

```

使用
```
# 拉个容器跑个sleep
yzw@yzw-vm:~/code/ns$ docker run -it -d centos bash
5028c0844804f6e5a5d57840448edeb596fa73a7d82d74eb40246ce9fc200e21
yzw@yzw-vm:~/code/ns$ docker exec -it 5028c084 bash
[root@5028c0844804 /]# id
uid=0(root) gid=0(root) groups=0(root)
[root@5028c0844804 /]# sleep 1000

# host上，sleep的uid是165536，uid 165536 是用户 dockremap 的一个从属 ID，在宿主机中并没有什么特殊权限。然而容器中的用户却是 root，
yzw@yzw-vm:~$ ps aux | grep sleep
165536    5249  0.0  0.0   4372   660 pts/1    S+   00:53   0:00 sleep 1000

# 启动某个容器不使用user namespace，加--userns=host
$ docker run -d --userns=host --name sleepme ubuntu sleep infinity

```
问题：
目前 docker 对它的支持还算不上完美，下面是已知的几个和现有功能不兼容的问题：

* 共享主机的 PID 或 NET namespace(--pid=host or --network=host)   
* 外部的存储、数据卷驱动可能不兼容、不支持 user namespace
* 使用 --privileged 而不指定 --userns=host

> Reference:  
> https://success.docker.com/article/introduction-to-user-namespaces-in-docker-engine  
> https://docs.docker.com/engine/security/userns-remap/
> https://medium.com/@mccode/understanding-how-uid-and-gid-work-in-docker-containers-c37a01d01cf

# QA
### 容器为什么不能识别到host的所有用户
因为/etc目录不是host的

### 容器exec进去为什么默认是root用户
dockerfile文件里面可以修改