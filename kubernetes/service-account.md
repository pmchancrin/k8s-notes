# Service Account

- **ServiceAccount** 为 Pod 中的进程提供身份信息，用于pod 访问api server。
- 自动挂载到容器的 **/var/run/secrets/kubernetes.io/serviceaccount** 目录中。
- 在认证时，**ServiceAccount** 的用户名格式为 **system:serviceaccount:(NAMESPACE):(SERVICEACCOUNT)**，并从属于两个 group：***system:serviceaccounts*** 和 **system:serviceaccounts:(NAMESPACE)**

## 开启sa

- **apiserver** 启动参数 *--admission-control* 加上 _ServiceAccount_ 。

```
 /hyperkube apiserver 
 --advertise-address=192.168.137.101 --etcd-servers=https://192.168.137.101:2379,https://192.168.137.102:2379,https://192.168.137.103:2379
 --etcd-quorum-read=true
 --etcd-cafile=/etc/ssl/etcd/ssl/ca.pem
 --etcd-certfile=/etc/ssl/etcd/ssl/node-node1.pem
 --etcd-keyfile=/etc/ssl/etcd/ssl/node-node1-key.pem 
 --insecure-bind-address=127.0.0.1 
 --apiserver-count=2 
 --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota 
 --service-cluster-ip-range=10.233.0.0/18 
 --service-node-port-range=30000-32767 
 --client-ca-file=/etc/kubernetes/ssl/ca.pem
 --basic-auth-file=/etc/kubernetes/users/known_users.csv --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem --token-auth-file=/etc/kubernetes/tokens/known_tokens.csv
 --service-account-key-file=/etc/kubernetes/ssl/apiserver-key.pem 
 --secure-port=6443 
 --insecure-port=8080 
 --storage-backend=etcd3 
 --v=2 
 --allow-privileged=true 
 --anonymous-auth=False

```

## 查看sa

- 系统默认给每个 _namespace_ 下面都会创建一个默认的default的sa。
- 每个sa拥有一个加密的 _token_，在 _secrets_ 里。

```
[root@node1 ~]# kubectl get sa --all-namespaces 
NAMESPACE     NAME      SECRETS   AGE
default       default   1         8d
kube-public   default   1         8d
kube-system   coredns   1         23h
kube-system   default   1         8d

[root@node1 ~]# kubectl get sa default -n kube-system -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2018-06-06T15:11:35Z
  name: default
  namespace: kube-system
  resourceVersion: "277"
  selfLink: /api/v1/namespaces/kube-system/serviceaccounts/default
  uid: e77b00db-699b-11e8-aac2-080027d04b6c
secrets:
- name: default-token-17rhr
```

*secret* 的类型为 **kubernetes.io/service-account-token**
```
[root@node1 ~]# kubectl get secret default-token-17rhr -n kube-system -o yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM5ekNDQWQrZ0F3SUJBZ0lKQVArdldaWFZMQ0JxTUEwR0NTcUdTSWIzRFFFQkN3VUFNQkl4RURBT0JnTlYKQkFNTUIydDFZbVV0WTJFd0hoY05NVGd3TmpBMk1UUTBOVFEwV2hjTk5EVXhNREl5TVRRME5UUTBXakFTTVJBdwpEZ1lEVlFRRERBZHJkV0psTFdOaE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBCnNFMjBnbkJlQjNuVTF1WTJmR1FKUk96UVlpbXJvN0hFbUlYQVFVaVFiR2VacE5HMWc1MHYyU0RYdlFzS0dMUXIKakJRNVVMd2U2SHhXbEo3NGszQVVpMURLcVBTcEorc3R3TlpIY2xGRWtDbjA3alNPTGxrZjZLUlVkcTlNMHZFTgpreEprRHYrVkxON2dnNzhISVZRMGxkUkh5TWxVcEpTbG9kbkQ5YzVuSG9QWTFKeHo4OE42QWhuTng2RnlnRVVaCkN6QzZXZ202MktPckE1YmNRdGw2eURQbnRTYVV6S3Z6VE94S0tLYXp5cm52cWlDVmdjQzFRRFFKUFRCZFdnNUYKREV0Ukp6RlFtcDJHL3NEcENLOVRqVGZWc0h0R3NyVitUY1VGWWVWM2Eyam9FNC9vSEJINVdoSjRCcC95YUxtNwpuSlpJVmpPNEVmUEtjekhxZmZ6QWt3SURBUUFCbzFBd1RqQWRCZ05WSFE0RUZnUVVHVDNackF2WEFRcStZL2h4CnZTcitGQ3lpMXNNd0h3WURWUjBqQkJnd0ZvQVVHVDNackF2WEFRcStZL2h4dlNyK0ZDeWkxc013REFZRFZSMFQKQkFVd0F3RUIvekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBcG9PbExXL0dnZEtIRUsxSkFyQmtOclRjMlNFLwp2NjFZNWdSZndQdkFMbEwwOHQ1d0hlUjRXMC9BQm9PQUdjL0t5M21OMzFWRmF1cTVZVW8rVFZRdnNsTk5ZZG9ICnUzbVg3a1BlYjIxZkh5WHhubU9tLzdkdTVDcFk5OHBLdnVNaWI5U3Via05nSWpCOVhZN3ZQZFVNQ2M5dnZVaUcKUmVYeGZiczl4UE9FVEJjYTl5a0t3VlVndWtnSTQ3MVViSlRBYzdpbkJjeWJTbWlsT0ppODRrRGdhTzJpc0MvSQp3QVpUVWZiQlZJd0s3MkFmTlhYcWszR2ovMWwzRVRTeXd6VGczc3J2emlYS0NUamFZRDZicWRKZ1gwMkdOeWZFCnprZHVWZEQwKzVtaDhCNEhnVENuemRrQUVQTFJZWVpuVUFMd09VTFRBbExmaHJrRDNoTXJlcUN2T0E9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  namespace: a3ViZS1zeXN0ZW0=
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJblI1Y0NJNklrcFhWQ0o5LmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUpyZFdKbExYTjVjM1JsYlNJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZqY21WMExtNWhiV1VpT2lKa1pXWmhkV3gwTFhSdmEyVnVMVEUzY21oeUlpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVibUZ0WlNJNkltUmxabUYxYkhRaUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sY25acFkyVXRZV05qYjNWdWRDNTFhV1FpT2lKbE56ZGlNREJrWWkwMk9UbGlMVEV4WlRndFlXRmpNaTB3T0RBd01qZGtNRFJpTm1NaUxDSnpkV0lpT2lKemVYTjBaVzA2YzJWeWRtbGpaV0ZqWTI5MWJuUTZhM1ZpWlMxemVYTjBaVzA2WkdWbVlYVnNkQ0o5Lm1MeGFXN0lBWTgyYjNndUFOUXktbzNDcDBBOHUtZHA3dm5UUXFSejV3T0FiWHhzbUtmeVREV2M2RHdZektsVFduX0oybk9wdG82SFh4RjhKSE00UDJXa0lLZThYN2RUcExjVjRIV2JkdllXRUZOckNwNTV6VEkxbkxzVlJkOHFOSmZnRlNhVHk4QXgycXlJalEyOC1zTUR6WUJWTHA5eERwNkxaVkpoV1VvWXg5Rm5WQ3VVUVNBMkNFUXJzZ1BpWkJMc1N2cWZuMl84R0EtZHVDR2E2bHkwZ19sb3hLLTlCQVVEanl3ZFRNTUtpdFg3dktiSE5fYmVUTFdCZDYtaWh4RGw5QmtpSTJmcng5cEUtYmVkSGszUEM2ZDAzRWh2U1pPaFcxbVo2RmI0LTZpVWVRdVJYZEY3NmhZWWxqVDZTNGh6Q1QtQ2U3cUpEQ1dEbFJER0lvZw==
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: default
    kubernetes.io/service-account.uid: e77b00db-699b-11e8-aac2-080027d04b6c
  creationTimestamp: 2018-06-06T15:11:35Z
  name: default-token-17rhr
  namespace: kube-system
  resourceVersion: "275"
  selfLink: /api/v1/namespaces/kube-system/secrets/default-token-17rhr
  uid: e780df51-699b-11e8-aac2-080027d04b6c
type: kubernetes.io/service-account-token
```

当用户在该 *namespace* 下创建 pod 时，会默认使用这个 sa ，k8s会默认把 sa 挂载到容器内。
```
[root@node1 ~]# kubectl get pods busybox1 -n kube-system -o yaml
apiVersion: v1
kind: Pod
metadata:
spec:
  containers:
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  volumes:
  - name: default-token-17rhr
    secret:
      defaultMode: 420
      secretName: default-token-17rhr

  ...
  volumes:
  - name: default-token-17rhr
    secret:
      defaultMode: 420
      secretName: default-token-17rhr
```

pod的容器里面可以查询到token和crt
```
/ # ls -l /var/run/secrets/kubernetes.io/serviceaccount/
total 0
lrwxrwxrwx    1 root     root            13 Jun 15 08:22 ca.crt -> ..data/ca.crt
lrwxrwxrwx    1 root     root            16 Jun 15 08:22 namespace -> ..data/namespace
lrwxrwxrwx    1 root     root            12 Jun 15 08:22 token -> ..data/token
```

## 创建sa

创建一个sa，会自动创建一个secret token，被sa引用。
```
[root@node1 ~]# cat sa-test.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-test

[root@node1 ~]# kubectl create -f sa-test.yaml 
serviceaccount "sa-test" created

[root@node1 ~]# kubectl get sa/sa-test -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2018-06-15T09:29:48Z
  name: sa-test
  namespace: default
  resourceVersion: "345447"
  selfLink: /api/v1/namespaces/default/serviceaccounts/sa-test
  uid: a5b2b982-707e-11e8-ac37-0800277b75c4
secrets:
- name: sa-test-token-xqb7q

[root@node1 ~]# kubectl get secrets
NAME                  TYPE                                  DATA      AGE
default-token-xz2pl   kubernetes.io/service-account-token   3         8d
sa-test-token-xqb7q   kubernetes.io/service-account-token   3         43s

[root@node1 ~]# kubectl get secrets/sa-test-token-xqb7q -o yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM5ekNDQWQrZ0F3SUJBZ0lKQVArdldaWFZMQ0JxTUEwR0NTcUdTSWIzRFFFQkN3VUFNQkl4RURBT0JnTlYKQkFNTUIydDFZbVV0WTJFd0hoY05NVGd3TmpBMk1UUTBOVFEwV2hjTk5EVXhNREl5TVRRME5UUTBXakFTTVJBdwpEZ1lEVlFRRERBZHJkV0psTFdOaE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBCnNFMjBnbkJlQjNuVTF1WTJmR1FKUk96UVlpbXJvN0hFbUlYQVFVaVFiR2VacE5HMWc1MHYyU0RYdlFzS0dMUXIKakJRNVVMd2U2SHhXbEo3NGszQVVpMURLcVBTcEorc3R3TlpIY2xGRWtDbjA3alNPTGxrZjZLUlVkcTlNMHZFTgpreEprRHYrVkxON2dnNzhISVZRMGxkUkh5TWxVcEpTbG9kbkQ5YzVuSG9QWTFKeHo4OE42QWhuTng2RnlnRVVaCkN6QzZXZ202MktPckE1YmNRdGw2eURQbnRTYVV6S3Z6VE94S0tLYXp5cm52cWlDVmdjQzFRRFFKUFRCZFdnNUYKREV0Ukp6RlFtcDJHL3NEcENLOVRqVGZWc0h0R3NyVitUY1VGWWVWM2Eyam9FNC9vSEJINVdoSjRCcC95YUxtNwpuSlpJVmpPNEVmUEtjekhxZmZ6QWt3SURBUUFCbzFBd1RqQWRCZ05WSFE0RUZnUVVHVDNackF2WEFRcStZL2h4CnZTcitGQ3lpMXNNd0h3WURWUjBqQkJnd0ZvQVVHVDNackF2WEFRcStZL2h4dlNyK0ZDeWkxc013REFZRFZSMFQKQkFVd0F3RUIvekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBcG9PbExXL0dnZEtIRUsxSkFyQmtOclRjMlNFLwp2NjFZNWdSZndQdkFMbEwwOHQ1d0hlUjRXMC9BQm9PQUdjL0t5M21OMzFWRmF1cTVZVW8rVFZRdnNsTk5ZZG9ICnUzbVg3a1BlYjIxZkh5WHhubU9tLzdkdTVDcFk5OHBLdnVNaWI5U3Via05nSWpCOVhZN3ZQZFVNQ2M5dnZVaUcKUmVYeGZiczl4UE9FVEJjYTl5a0t3VlVndWtnSTQ3MVViSlRBYzdpbkJjeWJTbWlsT0ppODRrRGdhTzJpc0MvSQp3QVpUVWZiQlZJd0s3MkFmTlhYcWszR2ovMWwzRVRTeXd6VGczc3J2emlYS0NUamFZRDZicWRKZ1gwMkdOeWZFCnprZHVWZEQwKzVtaDhCNEhnVENuemRrQUVQTFJZWVpuVUFMd09VTFRBbExmaHJrRDNoTXJlcUN2T0E9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  namespace: ZGVmYXVsdA==
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJblI1Y0NJNklrcFhWQ0o5LmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbk5oTFhSbGMzUXRkRzlyWlc0dGVIRmlOM0VpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9pYzJFdGRHVnpkQ0lzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVnlkbWxqWlMxaFkyTnZkVzUwTG5WcFpDSTZJbUUxWWpKaU9UZ3lMVGN3TjJVdE1URmxPQzFoWXpNM0xUQTRNREF5TnpkaU56VmpOQ0lzSW5OMVlpSTZJbk41YzNSbGJUcHpaWEoyYVdObFlXTmpiM1Z1ZERwa1pXWmhkV3gwT25OaExYUmxjM1FpZlEucWVmZVZXT2xnTThKYUZ6eG5IaDJQMUMzSXZ0SU9fSGlmRUVBay1vVVZ0cEcxMFJkQWVfemZndmpyWmVCOTVaTlFlREZrZDd2dVpkbXNDY05mMk1mZzhOV0xsZHo3ZDJwUG5PQ0NaUk9xMklxY1R3dzFwc1RFNnhIdWE5RDlEUE9fTzd6Vkx5Y29qV0xDbUN1Zk9qbHpoSGlvR2M3YzhSZE5rRE9zMXhuSEFPY05DcTFkRTFHLTdhVlFCbmxNdG9pTWt3VWJ5SFR3a3dBakpxNHkwbVNXTlg5NllsSWxCVGE2dExJUkNMSTVDRjE4TUYybUV4LURqUGIybkdxMDQ3dHV0aDdqbk9nWW1yaTZWZ3kybWE2Y2tMZWRvOXNzLXJtQm9QbUt1NmJhcjZiSXR0ZkptR2dvUHAtQjQtNVB2N0RqY19FbHdCeHNrQ25GdHVON2oxX0pR
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: sa-test
    kubernetes.io/service-account.uid: a5b2b982-707e-11e8-ac37-0800277b75c4
  creationTimestamp: 2018-06-15T09:29:48Z
  name: sa-test-token-xqb7q
  namespace: default
  resourceVersion: "345446"
  selfLink: /api/v1/namespaces/default/secrets/sa-test-token-xqb7q
  uid: a5ec7e4a-707e-11e8-9d66-080027d04b6c
type: kubernetes.io/service-account-token
```

创建sa 的默认 secret token
```
[root@node1 ~]# cat sa-test-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: sa-test-secret
  annotations: 
    kubernetes.io/service-account.name: sa-test
type: kubernetes.io/service-account-token

```

-  *pod* 和 *service account* 中可以设置 *automountServiceAccountToken*来取消自动挂载API凭证,如果都设置了，pod设置的优先级更高。
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
...
```

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: build-robot
  automountServiceAccountToken: false
  ...
```

## 参考

* https://jimmysong.io/kubernetes-handbook/concepts/serviceaccount.html
* https://blog.csdn.net/u010278923/article/details/72857928
* https://www.jianshu.com/p/415c5fc6ddcf
* https://kubernetes.io/docs/reference/access-authn-authz/authorization/#a-quick-note-on-service-accounts
* https://github.com/kubernetes/kubernetes/blob/release-1.0/examples/cassandra/README.md