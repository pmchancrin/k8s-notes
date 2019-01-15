
# Docker Dommands

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

> `<none>` 镜像叫悬挂 dangling 镜像

# APP Commands

**yum server**

```
docker run --name nginx_yum -v /data/nginx/nginx.conf:/etc/nginx/nginx.conf:ro -v /data/yum:/opt:ro -p 8082:80 --restart always -d nginx
```

**file server**

```
docker run --name nginx_file -v /data/nginx/nginx.conf:/etc/nginx/nginx.conf:ro -v /data/file/:/opt:ro -p 8083:80 --restart always -d nginx
```

**pip server**

```
docker run --name pypiserver -v /data/pip/:/packages -p 8084:3141 --restart always -d pypiserver
```

**docker registry**

```
docker run -d -p 80:5000 -v /media/sf_share/registry:/var/lib/registry --restart always --name registry registry:latest

curl -X GET http://localhost:80/v2/_catalog

docker run -d --name registry_frontend -e ENV_DOCKER_REGISTRY_HOST=172.17.0.3 -e ENV_DOCKER_REGISTRY_PORT=5000 -p 8081:80 --restart always konradkleine/docker-registry-frontend:v2

docker run -it -d -p 8081:8080 --name registry_web --link registry -e REGISTRY_URL=http://registry:5000/v2 -e REGISTRY_NAME=localhost:5000 hyper/docker-registry-web

docker run --name nginx_web -v /media/sf_share/nginx/nginx.conf:/etc/nginx/nginx.conf:ro -v /media/sf_share/nginx/web:/opt:ro -p 8080:80 --restart always -d nginx
nginx 访问403 原因目录没有权限  
/etc/fstab
share /media/sf_share vboxsf defaults 0 0 
```

**jenkins**

```
docker run -u root -d -p 8080:8080 -p 50000:50000 -v /srv/jenkins:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock --env JAVA_OPTS="-Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Shanghai" --name jenkins --restart always jenkinsci/blueocean
```

**gitlab**

```
docker run --detach  --hostname 192.168.137.102  --publish 20443:443  --publish 20080:80  --publish 20022:22  --name gitlab  --restart always --volume /srv/gitlab/config:/etc/gitlab  --volume /srv/gitlab/logs:/var/log/gitlab  --volume /srv/gitlab/data:/var/opt/gitlab  gitlab/gitlab-ce

修改配置文件 /srv/gitlab/config/gitlab.rb
external_url 'http://192.168.137.102:20080'
nginx['listen_port'] = 80
gitlab_rails['gitlab_shell_ssh_port'] = 20022
gitlab_rails['gitlab_ssh_host'] = '192.168.137.102'

docker exec -it gitlab /bin/bash
gitlab-ctl reconfigure 
gitlab-ctl restart

```
