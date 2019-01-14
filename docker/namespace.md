# Namespace

> Reference:  
> http://man7.org/linux/man-pages/man7/namespaces.7.html  
> https://lwn.net/Articles/531114/

Namespace是对全局系统资源的一种封装隔离，使得处于不同namespace的进程拥有独立的全局系统资源，改变一个namespace中的系统资源只会影响当前namespace里的进程，对其他namespace中的进程没有影响。

现在Linux支持的namespace有：

Namespace  |	Constant   |	Isolates
---|---|---
IPC |	CLONE_NEWIPC |	System V IPC, POSIX message queues
Network|	CLONE_NEWNET |	Network devices, stacks, ports, etc.
Mount |	CLONE_NEWNS|	Mount points
PID	| CLONE_NEWPID	| Process IDs
User |	CLONE_NEWUSER| 	User and group IDs
UTS	| CLONE_NEWUTS	| Hostname and NIS domain name

查询当前进程使用的namespace，都是以文件形式存在：
```
yzw@yzw-vm:~$ sudo ls -l /proc/$$/ns
total 0
lrwxrwxrwx 1 yzw yzw 0 Dec 28 14:56 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 yzw yzw 0 Dec 28 14:56 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 yzw yzw 0 Dec 28 14:56 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 yzw yzw 0 Dec 28 14:56 net -> 'net:[4026531992]'
lrwxrwxrwx 1 yzw yzw 0 Dec 28 14:56 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 yzw yzw 0 Dec 28 14:56 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 yzw yzw 0 Dec 28 14:56 user -> 'user:[4026531837]'
lrwxrwxrwx 1 yzw yzw 0 Dec 28 14:56 uts -> 'uts:[4026531838]'

```

每个Namespace的个数限制：
```
yzw@yzw-vm:~$ ls -l /proc/sys/user/
total 0
-rw-r--r-- 1 root root 0 Dec 28 15:14 max_cgroup_namespaces
-rw-r--r-- 1 root root 0 Dec 28 15:14 max_inotify_instances
-rw-r--r-- 1 root root 0 Dec 28 15:14 max_inotify_watches
-rw-r--r-- 1 root root 0 Dec 28 15:14 max_ipc_namespaces
-rw-r--r-- 1 root root 0 Dec 28 15:14 max_mnt_namespaces
-rw-r--r-- 1 root root 0 Dec 28 15:14 max_net_namespaces
-rw-r--r-- 1 root root 0 Dec 28 15:14 max_pid_namespaces
-rw-r--r-- 1 root root 0 Dec 28 15:14 max_user_namespaces
-rw-r--r-- 1 root root 0 Dec 28 15:14 max_uts_namespaces
```


namespace的API：
* clone  
创建新的进程，通过flag参数CLONE_NEW*(CLONE_NEWNET,CLONE_NEWIPC,CLONE_NEWCGROUP )创建新的ns，并将新的进程加入ns，当前进程保持不变。
* setns  
将进程加入到已有的ns
* unshare  
将当前进程退出指定类型ns，加入到新创建的ns
* usenter  
在ns里面运行程序

当一个namespace的所有进程都退出，这个namespace就会被销毁。

## Example setns
> Reference:  
> http://man7.org/linux/man-pages/man2/setns.2.html

int setns(int fd, int nstype)  
fd： namespace的文件句柄 即：`/proc/[pid]/ns/` 中的ns文件。   
nstype：0，允许所有类型ns加入，CLONE_NEW* fd必须为指定类型ns。

```
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg) do { perror(msg);exit(EXIT_FAILURE);} while(0)

int main(int argc, char *argv[]){
    int fd;
    fd =open(argv[1],O_RDONLY);
    if (setns(fd,0) == -1){
        errExit("setns");
    }
    execvp(argv[2],&argv[2]);
    errExit("execvp");
}
```
将指定程序的进程加入指定ns
```
root@yzw-vm:/home/yzw/docker# gcc -o setnc setns.c 

root@yzw-vm:/home/yzw/code/ns# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
c17865d18a7c        centos:7.4.1708     "/bin/bash"         2 hours ago         Up 2 hours                              upbeat_jennings
root@yzw-vm:/home/yzw/code/ns# docker inspect --format ‘{{.State.Pid}}’ c17865d18a7c
‘3807’
root@yzw-vm:/home/yzw/code/ns# 
root@yzw-vm:/home/yzw/code/ns# ./setns /proc/3807/ns/mnt /bin/bash
[root@yzw-vm /]# ls
anaconda-post.log  bin  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
[root@yzw-vm /]# 
root@yzw-vm:/home/yzw/code/ns# ./setns /proc/3807/ns/uts /bin/bash
root@c17865d18a7c:/home/yzw/code/ns# 

```
## Example unshare

```
# 创建新的uts的ns，保存到uts-ns，运行hostname FOO 命令修改hostname
root@yzw-vm:/home/yzw/code# touch uts-ns
root@yzw-vm:/home/yzw/code# unshare --uts=/root/uts-ns hostname FOO
root@yzw-vm:/home/yzw/code# hostname
yzw-vm

# 进入ns执行hostname命令
root@yzw-vm:/home/yzw/code# nsenter --uts=./uts-ns hostname
FOO

#或者进入ns执行bash，查看hostname
root@yzw-vm:/home/yzw/code# nsenter --uts=./uts-ns /bin/bash
root@FOO:/home/yzw/code# hostname
FOO

```