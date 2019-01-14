# ConfigMap

ConfigMaps不是属性配置文件的替代品。ConfigMaps只是作为多个properties文件的引用。你可以把它理解为Linux系统中的/etc目录，专门用来存储配置文件的目录。

## 创建

目录创建

```
[root@node1 test]# ls config/
game.properties  ui.properties
[root@node1 test]# 
[root@node1 test]# cat config/game.properties 
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
[root@node1 test]# 
[root@node1 test]# cat config/ui.properties 
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
[root@node1 test]# 
[root@node1 test]# kubectl create configmap game-config --from-file=config/
configmap "game-config" created
[root@node1 test]#
[root@node1 test]# kubectl get configmap 
NAME          DATA      AGE
game-config   2         4m
[root@node1 test]# kubectl describe configmap game-config 
Name:		game-config
Namespace:	default
Labels:		<none>
Annotations:	<none>

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
ui.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice

Events:	<none>
[root@node1 test]# 
[root@node1 test]# kubectl get configmap game-config -o yaml
apiVersion: v1
data:
  game.properties: |-
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  creationTimestamp: 2018-06-25T11:32:50Z
  name: game-config
  namespace: default
  resourceVersion: "396096"
  selfLink: /api/v1/namespaces/default/configmaps/game-config
  uid: 7dd2d77c-786b-11e8-a0b4-0800277b75c4

```

文件创建

```
[root@node1 test]# kubectl create configmap game-config-2 --from-file=config/game.properties 
configmap "game-config-2" created
[root@node1 test]# 
```

环境变量文件创建

```
[root@node1 test]# cat config/game-env-file.properties 
enemies=aliens
lives=3
allowed="true"

# This comment and the empty line above it are ignored
[root@node1 test]# 
[root@node1 test]# kubectl create configmap game-config-3 --from-env-file=config/game-env-file.properties 
configmap "game-config-3" created
[root@node1 test]# 
[root@node1 test]# kubectl get configmap game-config-3 -o yaml
apiVersion: v1
data:
  allowed: '"true"'
  enemies: aliens
  lives: "3"
kind: ConfigMap
metadata:
  creationTimestamp: 2018-06-25T12:05:12Z
  name: game-config-3
  namespace: default
  resourceVersion: "398547"
  selfLink: /api/v1/namespaces/default/configmaps/game-config-3
  uid: 039aff2b-7870-11e8-a0b4-0800277b75c4
[root@node1 test]# 
```

key=value创建

```
[root@node1 test]# kubectl create configmap game-config-4 --from-literal=special.how=very --from-literal=special.type=charm
configmap "game-config-4" created
[root@node1 test]# kubectl get configmap game-config-4 -o yaml
apiVersion: v1
data:
  special.how: very
  special.type: charm
kind: ConfigMap
metadata:
  creationTimestamp: 2018-06-25T12:28:31Z
  name: game-config-4
  namespace: default
  resourceVersion: "400302"
  selfLink: /api/v1/namespaces/default/configmaps/game-config-4
  uid: 45a5d8fb-7873-11e8-a0b4-0800277b75c4
[root@node1 test]# 
```

## 使用

环境变量方式试用
```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: SPECIAL_LEVEL
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: SPECIAL_TYPE
  restartPolicy: Never
```

volume方式使用
```
[root@node1 test]# 
[root@node1 test]# cat busybox-config.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command:
    - sleep
    - "3600"
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: game-config-4 
[root@node1 test]# 
[root@node1 test]# vi busybox-config.yaml 
[root@node1 test]# kubectl create -f busybox-config.yaml 
pod "busybox" created
[root@node1 test]# kubectl exec -it busybox ls /etc/config
special.how   special.type
[root@node1 test]# 
```

## 热更新
* ==使用该 ConfigMap 挂载的 Env 不会同步更新。需要通过滚动更新 pod 的方式来强制重新挂载 ConfigMap==
* ==使用该 ConfigMap 挂载的 Volume 中的数据需要一段时间（实测大概10秒）才能同步更新==

## 参考
* https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/
* https://jimmysong.io/kubernetes-handbook/concepts/configmap-hot-update.html