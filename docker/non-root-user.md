# 非root用户无法运行docker命令

> Reference:  
> https://docs.docker.com/install/linux/linux-postinstall/#manage-docker-as-a-non-root-user

使用非root用户运行docker命令的时候会提示没有权限，需要加上sudo：
```
yzw@yzw-vm:~$ docker ps
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.39/containers/json: dial unix /var/run/docker.sock: connect: permission denied
```

是因为当前用户没有访问 /var/run/docker.sock 的权限：

```
yzw@yzw-vm:~$ ll /var/run/docker.sock 
srw-rw---- 1 root docker 0 Dec 26 10:24 /var/run/docker.sock=

yzw@yzw-vm:~$ cat /etc/group | grep docker
docker:x:998:
yzw@yzw-vm:~$ id
uid=1000(yzw) gid=1000(yzw) groups=1000(yzw),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lpadmin),126(sambashare),999(vboxsf)
```

需要将当前用户加入到docker组里面：
```
yzw@yzw-vm:~$ sudo gpasswd --add yzw docker
Adding user yzw to group docker

# 或者 sudo usermod -aG docker $USER

yzw@yzw-vm:~$ cat /etc/group | grep docker
docker:x:998:yzw
```

当前用户log out再 log back：

```
yzw@yzw-vm:~$ id
uid=1000(yzw) gid=1000(yzw) groups=1000(yzw),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lpadmin),126(sambashare),998(docker),999(vboxsf)
yzw@yzw-vm:~$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

```
