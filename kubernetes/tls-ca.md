# ca-bundle.crt

## What is ca-bundle.crt?

ca-bundle.crt is a file that contains well known root CAcertificates.

```
[root@node3 certs]# curl -v https://www.baidu.com/
* About to connect() to www.baidu.com port 443 (#0)
*   Trying 61.135.169.121...
* Connected to www.baidu.com (61.135.169.121) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: /etc/pki/tls/certs/ca-bundle.crt
  CApath: none
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
* 	subject: CN=baidu.com,OU=service operation department.,O="BeiJing Baidu Netcom Science Technology Co., Ltd",L=beijing,ST=beijing,C=CN
* 	start date: Jun 29 00:00:00 2017 GMT
* 	expire date: Aug 17 23:59:59 2018 GMT
* 	common name: baidu.com
* 	issuer: CN=Symantec Class 3 Secure Server CA - G4,OU=Symantec Trust Network,O=Symantec Corporation,C=US
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: www.baidu.com
> Accept: */*
> 

```

centos 更新ca-bundle.

```
cp foo.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust extract
```