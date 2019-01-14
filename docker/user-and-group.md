# 概述

> reference:
> https://wiki.archlinux.org/index.php/Users_and_groups_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)

UNIX系统一切皆文件。  
GNU/Linux 通过用户和用户组实现访问控制 —— 包括对文件访问、设备使用的控制。

每一个文件都从属一个用户（属主）和一个用户组（属组），有三类访问权限：R，W，X。

```
yzw@yzw-vm:~$ ll
drwxr-xr-x 2 yzw yzw 4096 Jun 19  2018 Desktop
-rw-r--r--  1 root root  168 Sep 12 22:08 Dockerfile
```
**第一列 权限：**  

首字母d为目录，-为文件；  
后面每3个字符为一组，分别对应：   用户u，组g，其他用户o，-表示无此权限；  
读r，写w，执行x；  
  
要给g添加权限读(r)和执行权限(x)就是：  
chmod g+rx 文件名 ---加号表示添加权限  
要取消其他用户的写(w)权限：  
chmod g-w 文件名 ---减号表示取消权限

**第三列： User**  
**第四列： Goup**

## User
User分为：  
1. root用户，超级管理员。
2. 虚拟用户，无登陆系统能力，系统运行必不可少的，比如bin，damen，ftp，nobody等。
3. 普通实体用户，能登陆系统，只能操作自己的目录和内容，权限有限。

### 查看User信息

**/etc/passwd:**  
用户名：口令：UID：GID：注释性描述：主目录：登陆shell 
```
yzw@yzw-vm:~$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
yzw:x:1000:1000:yzw,,,:/home/yzw:/bin/bash
...
```

密码口令为x，加密后保存在/etc/shadow

**/etc/shadow：**  
用户名:加密口令:最后一次修改时间:最小时间间隔:最大时间间隔:警告时间:不活动时间:失效时间:标志
```
yzw@yzw-vm:~$ sudo cat /etc/shadow
[sudo] password for yzw: 
root:!:17668:0:99999:7:::
daemon:*:17647:0:99999:7:::
man:*:17647:0:99999:7:::
nobody:*:17647:0:99999:7:::
yzw:$6$Kv6eIYhy$0ucI91b0KnzHl37y1WbfmMyxX.Jf7xSSukRBusXAM7sTaapYvS9HXWhDjgK9dozIJpl1FAzsGvWS5aqo.AVwL0:17668:0:99999:7:::
...
```
### 命令
**useradd 或 adduesr**：   
添加用户  
```
$ useradd -g docker -G root,adm -u 1000 -s /bin/bash -d /home/yzw yzw
$ useradd abc
```
**passwd:**  
为用户设置密码,**`新增加的user不设置密码无法登陆的`**。
```
# root用户重置不需要输入旧密码，普通用户需要输入旧密码。
$ passwd yzw
# -d 指定空密码
$ passwd -d yzw
```
**userdel**    
删除用户,主要删除/etc/group shadow passwd等记录和用户目录
```
# -r 删除目录
$ userdel -r yzw
```

**gpasswd**  
把user加到group，或从group删除
```
$ gpasswd -a yzw docker
$ gpasswd -d yzw docker
```
**usermod**：  
修改用户属性，如登录名、用户的home目录等  
```
# 修改用户组,离开其他组，只加入这个组
$ usermod -G groupTest test123
# 添加多个组
$ usermode -G A,B,C yzw
# 修改用户名称
$ usermode -l newname oldname
```
pwcov： 同步用户从/etc/passwd 到/etc/shadow  
pwck： 校验用户配置文件/etc/passwd 和/etc/shadow 文件内容是否合法或完整；  
pwunconv：与pwconv功能相反，用来关闭用户的投影密码。它会把密码从shadow文件内，重回存到passwd文件里，然后删除 /etc/shadow 文件。  
finger：查看用户信息工具  
id：查看用户的UID、GID及所归属的用户组  
chfn：更改用户信息工具  
su：用户切换工具，默认切换到root，需要输入切换的用户密码  
sudo：用来以其他身份来执行命令，预设的身份为root。在/etc/sudoers中设置了可执行sudo指令的用户。若其未经授权的用户企图使用sudo，则会发出警告的邮件给管理员。用户使用sudo时，必须先输入用户自己的密码，之后有5分钟的有效期限，超过期限则必须重新输入密码。  
visudo： 用于编辑 /etc/sudoers   的命令；也可以不用这个命令，直接用vi 来编辑 /etc/sudoers 的效果是一样的。



# Group

Group就是具由相同特征的用户User的集合体。  
一个User可以属于多个Group，一个主Group，其他都是附属Group。
一个User都有一个同名的Group。  

### 查询信息
**/etc/group**  
组名称：密码：GID：用户
```
yzw@yzw-vm:~$ cat /etc/group
root:x:0:
daemon:x:1:
man:x:12:
sudo:x:27:yzw
yzw:x:1000:
nobody:x:997:
...
```

**/etc/gshadow**  
组名称：密码：管理员账号：所属账号
```
yzw@yzw-vm:~$ sudo cat /etc/gshadow
[sudo] password for yzw: 
root:*::
daemon:*::
man:*::
yzw:!::
nobody:!!::

```

### 命令
**groupadd**：  
添加用户组
```
$ groupadd -g 101 group2
```
groupdel：删除用户组  
groupmod：修改用户组信息  
groups：显示用户所属的用户组  
grpck：用于验证组文件的完整性  
grpconv：通过/etc/group和/etc/gshadow 的文件内容来同步或创建/etc/gshadow ，如果/etc/gshadow 不存在则创建；  
grpunconv：通过/etc/group 和/etc/gshadow 文件内容来同步或创建/etc/group ，然后删除gshadow文件；

# UID/EUID/RUID and GID/EGID/RGID

**UID/GID**
* UID/GID是Kernel识别User/Group的标识，是唯一的， 但是一套UID/GID可以对应多个User/Group。  
* 相同UID/GID的不同User/Group的权限是相同的。
* root的UID和GID都是0，可以访问一切资源，叫privileged UID/GID。  
* 大部分Linux系统的前100 UID/GID都是系统使用的，新用户从500或者1000开始，Ubuntu/Centos上新用户从1000开始。  

```
yzw@yzw-vm:~$ id
uid=1000(yzw) gid=1000(yzw) groups=1000(yzw),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lpadmin),126(sambashare),998(docker),999(vboxsf)

```

每一个process有：   
**RUID/RGID**  Real UID/GID：identify process owner  
当用户使用用户名密码登陆系统后，旧有唯一确定的RUID了，就是这个用户的UID。创建用户的时候已经定义好了。    

**EUID/EGID** Effective UID/GDID： used in access control decisions  
进程在运行中对文件的实际范围权限，通常就是RUID。

**Saved UID/GID** saved UID/GID：stores a previous UID so that  it can be restored later  
**FSUID/FSGID**  used for access control to the file system

```
yzw@yzw-vm:~$ ps -o pid,uid,euid,ruid,suid,egid,rgid,sgid,cmd 
  PID   UID  EUID  RUID  SUID  EGID  RGID  SGID CMD
 3741  1000  1000  1000  1000  1000  1000  1000 bash
```

一个可执行文件（程序），在执行时，一般该文件只拥有调用这个文件的用户具有的权限。因为用户登陆系统后会执行Shell（bash或者sh），这是创建用户时就定义好的，这个用户执行的程序的父进程就是这个Shell，权限也就继承了父进程的权限。通过SUID/GID可以改变这种设置，这时候EUID就不是从父进程那继承来的了，就变成了文件本身的User/Group了。  

**SUID/SGID** Set UID/GID：对外权限的开放。跟文件而不是用户绑定。 用来设置可执行文件在执行阶段具有文件所有者的权限。 

普通用户执行su或者passwd时，这两个程序都会访问`/etc/passwd`和`/etc/shadow`文件，而这两个文件是需要root权限的。
```
yzw@yzw-vm:~$ ll /etc/passwd
-rw-r--r-- 1 root root 2774 Dec 27 00:10 /etc/passwd
yzw@yzw-vm:~$ ll /etc/shadow
-rw-r----- 1 root shadow 1545 Dec 27 00:10 /etc/shadow
```
su和passwd这两个可执行文件权限里面带了**`s`**：
```
yzw@yzw-vm:~$ ll /bin/su 
-rwsr-xr-x 1 root root 44664 Jan 25  2018 /bin/su*
yzw@yzw-vm:~$ ll /usr/bin/passwd 
-rwsr-xr-x 1 root root 59640 Jan 25  2018 /usr/bin/passwd*
```
**`s`** 就是set-user-id标志。有这个标志表示，任何普通用户运行su或者passwd程序时，其EUID就是该程序的所有者。

用户yzw执行`passwd`命令时，yzw的Shell会fork一个子进程，子进程的`RUID` == `EUID` == yzw的`UID`,然后exec程序`/usr/bin/passwd`,exec 会根据`/usr/bin/passwd`的`SUID` 把子进程的 `EUID` 设置成 `root`, 此时这个进程就有了`root`权限，可以读`/etc/shadow`文件了，从而完成yzw密码修改。  

通过`chmod`修改去掉`/usr/bin/passwd`的SUID权限：
```
yzw@yzw-vm:~$ sudo chmod u-s /usr/bin/passwd 
[sudo] password for yzw: 
yzw@yzw-vm:~$ ll /usr/bin/passwd 
-rwxr-xr-x 1 root root 59640 Jan 25  2018 /usr/bin/passwd*
```
修改密码就会报权限错误：
```
yzw@yzw-vm:~$ passwd 
Changing password for yzw.
(current) UNIX password: 
Enter new UNIX password: 
You must choose a longer password
passwd: Authentication token manipulation error
passwd: password unchanged
```


> **Reference**:  
> https://skednet.wordpress.com/2010/07/07/uid-euid-suid-fsuid/  
>
> When a process is created by FORK, the created process inherits its
parent's UIDs
>
> When a process executes a NEW FILE via EXEC, that executing process keeps
ITS OWN UIDs unless the SETUID in the new file is set.
 -- if the SETUID bit is set in the file we're EXECing, then  
    --> process's EUID == file owner's UID  
    --> process's SUID == file owner's UID
>
> To drop privilege temporarily, a process will remove the privileged UID
from its EUID but keep it saved as its saved UID (SUID); later the process
can restore the privileges by setting its EUID to its SUID. To drop
privileges permanently a process removes the privileged UID from all three
of its UIDs; thereafter the process cannot restore that privileged access

# Capabilities 

Linux 给普通用户尽可能低的权限，给root全部的权限，依赖单一账户执行特权操作的方式加大了系统风险，比如我们使用SUID知识需要一小部分特权，但是却获取了root的全部权限。引入了Capabilities的概念限制特权用户的权限。  

Capability 有三种：
* effective（e）：当前有效的能力集，当执行某特权操作时，系统检查cap_effective对应位是否有效，不再检查进程的有效UID是是否位0.
* permitted（p）：当前进程允许使用的能力集，effective包含于permitted。
* inherited（i）：可以被继承的能力集。比如执行`exec`允许其他命令时，能够被新命令继承的能力就在inherited集里面。
* Ambient： 外界的，这是一个为非特权程序的跨 execve(2) 保留的权能集合。外界的权能集合服从不可变性，如果权能既不是被允许的也不是可继承的，则它也从不可能是外界的。  

1. Capabilities 细分到线程，每个线程都有自己的Capabilities。 
2. 可执行文件也具有一定的Capabilities。  
3. POSIX能力只能分配给进程，不能分配给文件。  
4. 如果进程的真实 uid 或有效 uid 是 0（根用户），或者文件是 setuid root，那么文件的可继承集和允许集就是满的。  
5. 如果进程的有效 uid 是根用户，或者文件是 setuid root，那么文件有效集就是满的。
```
root@yzw-vm:/home/yzw# cat /proc/$$/task/10973/status | grep Cap
CapInh:	0000000000000000
CapPrm:	0000003fffffffff
CapEff:	0000003fffffffff
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000
```

Liunx定义了很多特权操作和对应的Capability：  
* CAP_CHOWN 改变文件的属性chown()  
* CAP_KILL 发送kill()，signal()信号
* CAP_SETUID  改变进程uid,setuid()
* CAP_SYS_PTRACE trace进程ptrace()
* CAP_SYS_TIME 系统时间settimeofday()
* ...



程序 | Capabilities
---|---
/bin/ping | CAP_NET_RAW
/bin/mount | CAP_SYS_ADMIN
/bin/su | CAP_DAC_OVERRIDE,CAP_SETGID,CAP_SETUID
/usr/bin/passwd | CAP_CHOWN ,CAP_DAC_OVERRIDE ,CAP_FOWNER





```
# ping 设置了SUID，所有非root用户可以使用
yzw@yzw-vm:~$ ll /bin/ping
-rwsr-xr-x 1 root root 68520 Aug 29 16:25 /bin/ping*

yzw@yzw-vm:~$ ping www.baidu.com
PING www.a.shifen.com (220.181.111.37) 56(84) bytes of data.
64 bytes from 220.181.111.37 (220.181.111.37): icmp_seq=1 ttl=53 time=20.5 ms
64 bytes from 220.181.111.37 (220.181.111.37): icmp_seq=2 ttl=53 time=19.10 ms

# 取消ping的SUID权限，非root用户就没权限了
yzw@yzw-vm:~$ sudo chmod u-s /bin/ping
[sudo] password for yzw: 
yzw@yzw-vm:~$ ping www.baidu.com
ping: socket: Operation not permitted

# 给ping设置cap_new_raw的Capability，没有SUID但有权限了
yzw@yzw-vm:~$ sudo setcap cap_net_raw+ep /bin/ping
yzw@yzw-vm:~$ ll /bin/ping
-rwxr-xr-x 1 root root 68520 Aug 29 16:25 /bin/ping*
yzw@yzw-vm:~$ sudo getcap /bin/ping
/bin/ping = cap_net_raw+ep

yzw@yzw-vm:~$ ping www.baidu.com
PING www.a.shifen.com (220.181.111.37) 56(84) bytes of data.
64 bytes from 220.181.111.37 (220.181.111.37): icmp_seq=1 ttl=53 time=19.9 ms
64 bytes from 220.181.111.37 (220.181.111.37): icmp_seq=2 ttl=53 time=20.2 ms

#删除能力
yzw@yzw-vm:~$ sudo setcap -r  /bin/ping
yzw@yzw-vm:~$ sudo setcap cap_net_raw-ep /bin/ping
```



> Reference:   
> https://www.hrwhisper.me/introduction-to-linux-capability/  
> https://rk700.github.io/2016/10/26/linux-capabilities/  
> https://linux.die.net/man/7/capabilities

# User Namespace  

Linux User Namespace 为正在运行的进程提供安全相关的隔离，包括UID/GID，Capabilities，限制他们对系统资源的访问。  

user namespace可以嵌套（目前内核控制最多32层），除了系统默认的user namespace外，所有的user namespace都有一个父user namespace，每个user namespace都可以有零到多个子user namespace。 当在一个进程中调用unshare或者clone创建新的user namespace时，当前进程原来所在的user namespace为父user namespace，新的user namespace为子user namespace.

在不同的user namespace中，同样一个用户的user ID 和group ID可以不一样，换句话说，一个用户可以在父user namespace中是普通用户，在子user namespace中是超级用户（超级用户只相对于子user namespace所拥有的资源，无法访问其他user namespace中需要超级用户才能访问资源）

从Linux 3.8开始，创建新的user namespace不需要root权限。

当前yzw用户信息
```
yzw@yzw-vm:~/code/ns$ id
uid=1000(yzw) gid=1000(yzw) groups=1000(yzw),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lpadmin),126(sambashare),998(docker),999(vboxsf)
yzw@yzw-vm:~/code/ns$ readlink /proc/$$/ns/user
user:[4026531837]
yzw@yzw-vm:~/code/ns$ cat /proc/$$/status | grep Cap
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000
# 非root没有Cap，改不了hostname
yzw@yzw-vm:~/code/ns$ hostname FOOO
hostname: you must be root to change the host name
```

创建user namespace 需要指定跟它父ns的uid/gid进行映射，这样系统才能控制一个user ns里的用户在其他user ns里面的权限，比如向其他user ns里面进程发信号，访问其他user ns里用户权限的文件。noboy的uid/gid值是溢出值，在`/proc/sys/kernel/overflowuid`和 `/proc/sys/kernel/overflowgid`中定义。
```
yzw@yzw-vm:~/code/ns$ unshare --user /bin/bash
nobody@yzw-vm:~/code/ns$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)

```
创建user namespace后，需要修改映射关系，映射ID的方法是添加配置到`/proc/PID/uid_map`和`/proc/PID/gid_map`（这里的PID是新user namespace中的进程ID，刚开始时这两个文件都是空的）。  
格式：ID-inside-ns ID-outside-ns length  
比如：0 1000 256 父user ns的1000-1256映射到新user ns的0-256

系统默认的user namespace没有父user namespace，但为了保持一致，kernel提供了一个虚拟的uid和gid map文件，看起来是这样子的:
```
yzw@yzw-vm:~/code/ns$ cat /proc/$$/uid_map
         0          0 4294967295
yzw@yzw-vm:~/code/ns$ cat /proc/$$/gid_map
         0          0 4294967295
```

创建映射：
```
#新创建user ns 的map文件时空的
yzw@yzw-vm:~/code/ns$ unshare --user /bin/bash
nobody@yzw-vm:~/code/ns$ cat /proc/$$/uid_map 
nobody@yzw-vm:~/code/ns$ cat /proc/$$/gid_map 
nobody@yzw-vm:~/code/ns$ ls -l /proc/$$/uid_map 
-rw-r--r-- 1 nobody nogroup 0 Dec 28 23:54 /proc/2888/uid_map
#nobody有读写权限，但还是不让写
nobody@yzw-vm:~/code/ns$  echo '0 1000 100' > /proc/$$/uid_map
bash: echo: write error: Operation not permitted
nobody@yzw-vm:~/code/ns$ echo $$
2888
#新开个终端到yzw用户下
#因为这个文件需要CAP_SETUID和CAP_SETGID的权限，新user ns的bash进程没有
yzw@yzw-vm:~$ ll /proc/2888/uid_map /proc/2888/gid_map
-rw-r--r-- 1 yzw yzw 0 Dec 28 23:54 /proc/2888/gid_map
-rw-r--r-- 1 yzw yzw 0 Dec 29 00:11 /proc/2888/uid_map
yzw@yzw-vm:~$ echo '0 1000 100' > /proc/2888/uid_map
-bash: echo: write error: Operation not permitted
yzw@yzw-vm:~$ echo '0 1000 100' > /proc/2888/gid_map
-bash: echo: write error: Operation not permitted
#给当前bash增加Cap
yzw@yzw-vm:~$ sudo setcap cap_setgid,cap_setuid+ep /bin/bash
yzw@yzw-vm:~$ getcap /bin/bash
/bin/bash = cap_setgid,cap_setuid+ep
yzw@yzw-vm:~$ cat /proc/$$/status | grep Cap
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000
yzw@yzw-vm:~$ exec bash
yzw@yzw-vm:~$ cat /proc/$$/status | grep Cap
CapInh:	0000000000000000
CapPrm:	00000000000000c0
CapEff:	00000000000000c0
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000
yzw@yzw-vm:~$ echo '0 1000 100' > /proc/2888/gid_map
yzw@yzw-vm:~$ echo '0 1000 100' > /proc/2888/uid_map
#恢复Cap
yzw@yzw-vm:~$ sudo setcap cap_setgid,cap_setuid-ep /bin/bash
yzw@yzw-vm:~$ getcap /bin/bash
/bin/bash =
yzw@yzw-vm:~$ 
#在新建user ns下，重新载入bash，变成root用户了映射的是yzw用户
nobody@yzw-vm:~/code/ns$ exec /bin/bash
root@yzw-vm:~/code/ns# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
root@yzw-vm:~/code/ns# 

```

root用户映射：
```
#创建新的user ns，-r 是将root用户映射到yzw用户
yzw@yzw-vm:~/code/ns$ unshare --user -r /bin/bash
root@yzw-vm:~/code/ns# readlink /proc/$$/ns/user
user:[4026532301]

# 新的user ns 里面root用户的bash进程拥有全部Cap
root@yzw-vm:~/code/ns# cat /proc/$$/status | grep Cap
CapInh:	0000000000000000
CapPrm:	0000003fffffffff
CapEff:	0000003fffffffff
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000
root@yzw-vm:~/code/ns# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
root@yzw-vm:~/code/ns# cat /proc/$$/uid_map 
         0       1000          1
root@yzw-vm:~/code/ns# cat /proc/$$/gid_map 
         0       1000          1
root@yzw-vm:~/code/ns# hostname FOOOO

# 但是root用户依然改不了uts，因为yzw用户Cap都是空，没有改hostname权限
root@yzw-vm:~/code/ns# hostname FOOOO
hostname: you must be root to change the host name
```
使用root用户创建user ns，在子ns里面root依然不能改hostname。  
不管怎么映射，当用子 user namespace 的用户访问父 user namespace 的资源的时候，它启动的进程的 capability 都为空，所以这里子 user namespace 的 root 用户在父 user namespace 中就相当于一个普通的用户。
```
root@yzw-vm:/home/yzw/code/ns# unshare --user -r /bin/bash
root@yzw-vm:/home/yzw/code/ns# hostname FOO
hostname: you must be root to change the host name
root@yzw-vm:/home/yzw/code/ns# 

```




> Reference:  
> https://segmentfault.com/a/1190000006913195 
> https://segmentfault.com/a/1190000006913499  
> https://lwn.net/Articles/532593/  
> https://lwn.net/Articles/540087/