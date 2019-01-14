ETCD
====
* kubernetes的所有云数据都保存在/registry目录下，下一层是API对象类型（复数），再下一层是namespace，最后一层是对象名字。
* calico网络存储是以v2的API来存储的，kubernetes是v3 API。
* 环境变量 ETCDCTL_API 默认是2

### v2 查询命令
```
[root@node1 test]# export ETCDCTL_API=2
[root@node1 test]# etcdctl --version
etcdctl version: 3.2.4
API version: 2
[root@node1 test]# etcdctl ls 
/calico
[root@node1 test]# etcdctl get /calico/bgp/v1/host/node1/ip_addr_v4
192.168.137.101

```

### v3 查询命令
* --prefix 查询所有子目录
* -w 输出格式
* -m 
```
[root@node1 test]# export ETCDCTL_API=3
[root@node1 test]# 
[root@node1 test]# etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/member-node1.pem --key=/etc/ssl/etcd/ssl/member-node1-key.pem get /registry/configmaps/default/game-config -w=json|python -m json.tool
{
    "count": 1,
    "header": {
        "cluster_id": 8635001981321661628,
        "member_id": 12035000773768423000,
        "raft_term": 1682,
        "revision": 416787
    },
    "kvs": [
        {
            "create_revision": 396096,
            "key": "L3JlZ2lzdHJ5L2NvbmZpZ21hcHMvZGVmYXVsdC9nYW1lLWNvbmZpZw==",
            "mod_revision": 396096,
            "value": "azhzAAoPCgJ2MRIJQ29uZmlnTWFwEu8CClMKC2dhbWUtY29uZmlnEgAaB2RlZmF1bHQiACokN2RkMmQ3N2MtNzg2Yi0xMWU4LWEwYjQtMDgwMDI3N2I3NWM0MgA4AEILCOKlw9kFEKypk3x6ABKxAQoPZ2FtZS5wcm9wZXJ0aWVzEp0BZW5lbWllcz1hbGllbnMKbGl2ZXM9MwplbmVtaWVzLmNoZWF0PXRydWUKZW5lbWllcy5jaGVhdC5sZXZlbD1ub0dvb2RSb3R0ZW4Kc2VjcmV0LmNvZGUucGFzc3BocmFzZT1VVURETFJMUkJBQkFTCnNlY3JldC5jb2RlLmFsbG93ZWQ9dHJ1ZQpzZWNyZXQuY29kZS5saXZlcz0zMBJkCg11aS5wcm9wZXJ0aWVzElNjb2xvci5nb29kPXB1cnBsZQpjb2xvci5iYWQ9eWVsbG93CmFsbG93LnRleHRtb2RlPXRydWUKaG93Lm5pY2UudG8ubG9vaz1mYWlybHlOaWNlChoAIgA=",
            "version": 1
        }
    ]
}

```
* 输出key base64编码了，需要解码
```
[root@node1 test]# echo L3JlZ2lzdHJ5L2NvbmZpZ21hcHMvZGVmYXVsdC9nYW1lLWNvbmZpZw==|base64 -d
/registry/configmaps/default/game-config
[root@node1 test]# 
```

### 遇到的问题
etcd容器内无法查询
```
/etc # etcdctl --ca-file=$ETCD_TRUSTED_CA_FILE --cert-file=$ETCD_CERT_FILE --key-file=ETCD_KEY_FILE cluster-health
Error:  open ETCD_KEY_FILE: no such file or directory
/etc # etcdctl --ca-file=$ETCD_TRUSTED_CA_FILE --cert-file=$ETCD_CERT_FILE --key-file=$ETCD_KEY_FILE cluster-health
cluster may be unhealthy: failed to list members
Error:  client: etcd cluster is unavailable or misconfigured; error #0: malformed HTTP response "\x15\x03\x01\x00\x02\x02"
; error #1: dial tcp 127.0.0.1:4001: getsockopt: connection refused

error #0: malformed HTTP response "\x15\x03\x01\x00\x02\x02"
error #1: dial tcp 127.0.0.1:4001: getsockopt: connection refused
```
解决办法

```
/etc # export ETCDCTL_ENDPOINT=https://127.0.0.1:2379
/etc # etcdctl --ca-file=$ETCD_TRUSTED_CA_FILE --cert-file=$ETCD_CERT_FILE --key-file=$ETCD_KEY_FILE cluster-health
member 82cf071e8608bc84 is healthy: got healthy result from https://192.168.137.102:2379
member a704e9708804a658 is healthy: got healthy result from https://192.168.137.101:2379
member e30c96cca4e3ace1 is healthy: got healthy result from https://192.168.137.103:2379
cluster is healthy
```


### 参考文档
* https://jimmysong.io/kubernetes-handbook/guide/using-etcdctl-to-access-kubernetes-data.html
* https://jimmysong.io/kubernetes-handbook/concepts/etcd.html
* https://zhangkesheng.github.io/2018/01/25/kubernetes-ha/