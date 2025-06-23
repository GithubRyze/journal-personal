---
title: K8S kubectl token 过期
author: 刘Sir
date: 2024-06-18 10:10:00 +0800
categories: [技术]
tags: [容器]
render_with_liquid: false
---  

kubectl get pods 报错
```
ubuntu@sz-mcr-dev-master-1:~/mdp-k8s-yaml/nginx-ingress$ kubectl get pods
E0618 16:03:06.026508  100793 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
E0618 16:03:06.029871  100793 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
E0618 16:03:06.033212  100793 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
E0618 16:03:06.036185  100793 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
E0618 16:03:06.039354  100793 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
error: You must be logged in to the server (the server has asked for the client to provide credentials)

```
使用 kubectl get pods --v=8 命令看看输出：
```
ubuntu@sz-mcr-dev-master-1:~/mdp-k8s-yaml/nginx-ingress$ kubectl get pods --v=8
I0618 16:08:38.729108  104483 loader.go:373] Config loaded from file:  /home/ubuntu/.kube/config
I0618 16:08:38.733231  104483 round_trippers.go:463] GET https://10.18.12.14:6443/api?timeout=32s
I0618 16:08:38.733291  104483 round_trippers.go:469] Request Headers:
I0618 16:08:38.733331  104483 round_trippers.go:473]     Accept: application/json;g=apidiscovery.k8s.io;v=v2beta1;as=APIGroupDiscoveryList,application/json
I0618 16:08:38.733374  104483 round_trippers.go:473]     User-Agent: kubectl/v1.27.1 (linux/amd64) kubernetes/4c94112
I0618 16:08:38.753335  104483 round_trippers.go:574] Response Status: 401 Unauthorized in 19 milliseconds
I0618 16:08:38.753401  104483 round_trippers.go:577] Response Headers:
I0618 16:08:38.753419  104483 round_trippers.go:580]     Audit-Id: c263f5ed-264d-495c-81b2-8b43b3a86d79
I0618 16:08:38.753433  104483 round_trippers.go:580]     Cache-Control: no-cache, private
I0618 16:08:38.753455  104483 round_trippers.go:580]     Content-Type: application/json
I0618 16:08:38.753473  104483 round_trippers.go:580]     Content-Length: 129
I0618 16:08:38.753492  104483 round_trippers.go:580]     Date: Wed, 18 Jun 2025 08:08:38 GMT
I0618 16:08:38.754075  104483 request.go:1188] Response Body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"Unauthorized","re
E0618 16:08:38.754906  104483 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
I0618 16:08:38.755019  104483 cached_discovery.go:120] skipped caching discovery info due to the server has asked for the client to provide credentials
I0618 16:08:38.755335  104483 round_trippers.go:463] GET https://10.18.12.14:6443/api?timeout=32s
I0618 16:08:38.755369  104483 round_trippers.go:469] Request Headers:
I0618 16:08:38.755394  104483 round_trippers.go:473]     Accept: application/json;g=apidiscovery.k8s.io;v=v2beta1;as=APIGroupDiscoveryList,application/json
I0618 16:08:38.755417  104483 round_trippers.go:473]     User-Agent: kubectl/v1.27.1 (linux/amd64) kubernetes/4c94112
I0618 16:08:38.757826  104483 round_trippers.go:574] Response Status: 401 Unauthorized in 2 milliseconds
I0618 16:08:38.757890  104483 round_trippers.go:577] Response Headers:
I0618 16:08:38.757966  104483 round_trippers.go:580]     Audit-Id: 1c193fca-b591-4531-9772-dd3db936ed03
I0618 16:08:38.758027  104483 round_trippers.go:580]     Cache-Control: no-cache, private
I0618 16:08:38.758066  104483 round_trippers.go:580]     Content-Type: application/json
I0618 16:08:38.758117  104483 round_trippers.go:580]     Content-Length: 129
I0618 16:08:38.758146  104483 round_trippers.go:580]     Date: Wed, 18 Jun 2025 08:08:38 GMT
I0618 16:08:38.758617  104483 request.go:1188] Response Body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"Unauthorized","re
E0618 16:08:38.759268  104483 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
I0618 16:08:38.759339  104483 cached_discovery.go:120] skipped caching discovery info due to the server has asked for the client to provide credentials
I0618 16:08:38.759420  104483 shortcut.go:100] Error loading discovery information: the server has asked for the client to provide credentials
I0618 16:08:38.759723  104483 round_trippers.go:463] GET https://10.18.12.14:6443/api?timeout=32s
I0618 16:08:38.759776  104483 round_trippers.go:469] Request Headers:
I0618 16:08:38.759816  104483 round_trippers.go:473]     Accept: application/json;g=apidiscovery.k8s.io;v=v2beta1;as=APIGroupDiscoveryList,application/json
I0618 16:08:38.759853  104483 round_trippers.go:473]     User-Agent: kubectl/v1.27.1 (linux/amd64) kubernetes/4c94112
I0618 16:08:38.762372  104483 round_trippers.go:574] Response Status: 401 Unauthorized in 2 milliseconds
I0618 16:08:38.762440  104483 round_trippers.go:577] Response Headers:
I0618 16:08:38.762463  104483 round_trippers.go:580]     Cache-Control: no-cache, private
I0618 16:08:38.762489  104483 round_trippers.go:580]     Content-Type: application/json
I0618 16:08:38.762509  104483 round_trippers.go:580]     Content-Length: 129
I0618 16:08:38.762527  104483 round_trippers.go:580]     Date: Wed, 18 Jun 2025 08:08:38 GMT
I0618 16:08:38.762549  104483 round_trippers.go:580]     Audit-Id: 851d6254-6977-4468-9a15-cdc6eaefec17
I0618 16:08:38.762920  104483 request.go:1188] Response Body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"Unauthorized","re
E0618 16:08:38.763535  104483 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
I0618 16:08:38.763600  104483 cached_discovery.go:120] skipped caching discovery info due to the server has asked for the client to provide credentials
I0618 16:08:38.763975  104483 round_trippers.go:463] GET https://10.18.12.14:6443/api?timeout=32s
I0618 16:08:38.764030  104483 round_trippers.go:469] Request Headers:
I0618 16:08:38.764088  104483 round_trippers.go:473]     User-Agent: kubectl/v1.27.1 (linux/amd64) kubernetes/4c94112
I0618 16:08:38.764142  104483 round_trippers.go:473]     Accept: application/json;g=apidiscovery.k8s.io;v=v2beta1;as=APIGroupDiscoveryList,application/json
I0618 16:08:38.765881  104483 round_trippers.go:574] Response Status: 401 Unauthorized in 1 milliseconds
I0618 16:08:38.765925  104483 round_trippers.go:577] Response Headers:
I0618 16:08:38.765939  104483 round_trippers.go:580]     Audit-Id: 5a1ba954-886e-4e40-95b8-6963e93cbc4c
I0618 16:08:38.765960  104483 round_trippers.go:580]     Cache-Control: no-cache, private
I0618 16:08:38.765994  104483 round_trippers.go:580]     Content-Type: application/json
I0618 16:08:38.766011  104483 round_trippers.go:580]     Content-Length: 129
I0618 16:08:38.766028  104483 round_trippers.go:580]     Date: Wed, 18 Jun 2025 08:08:38 GMT
I0618 16:08:38.766309  104483 request.go:1188] Response Body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"Unauthorized","re
E0618 16:08:38.766890  104483 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
I0618 16:08:38.766935  104483 cached_discovery.go:120] skipped caching discovery info due to the server has asked for the client to provide credentials
I0618 16:08:38.767136  104483 round_trippers.go:463] GET https://10.18.12.14:6443/api?timeout=32s
I0618 16:08:38.767161  104483 round_trippers.go:469] Request Headers:
I0618 16:08:38.767182  104483 round_trippers.go:473]     Accept: application/json;g=apidiscovery.k8s.io;v=v2beta1;as=APIGroupDiscoveryList,application/json
I0618 16:08:38.767205  104483 round_trippers.go:473]     User-Agent: kubectl/v1.27.1 (linux/amd64) kubernetes/4c94112
I0618 16:08:38.769248  104483 round_trippers.go:574] Response Status: 401 Unauthorized in 1 milliseconds
I0618 16:08:38.769346  104483 round_trippers.go:577] Response Headers:
I0618 16:08:38.769383  104483 round_trippers.go:580]     Content-Length: 129
I0618 16:08:38.769432  104483 round_trippers.go:580]     Date: Wed, 18 Jun 2025 08:08:38 GMT
I0618 16:08:38.769463  104483 round_trippers.go:580]     Audit-Id: a337ed15-5959-41ad-a14e-a8e0e2fe5596
I0618 16:08:38.769493  104483 round_trippers.go:580]     Cache-Control: no-cache, private
I0618 16:08:38.769525  104483 round_trippers.go:580]     Content-Type: application/json
I0618 16:08:38.769965  104483 request.go:1188] Response Body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"Unauthorized","re
E0618 16:08:38.770581  104483 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
I0618 16:08:38.770626  104483 cached_discovery.go:120] skipped caching discovery info due to the server has asked for the client to provide credentials
I0618 16:08:38.771101  104483 helpers.go:246] server response object: [{
  "metadata": {},
  "status": "Failure",
  "message": "the server has asked for the client to provide credentials",
  "reason": "Unauthorized",
  "details": {
    "causes": [
      {
        "reason": "UnexpectedServerResponse",
        "message": "unknown"
      }
    ]
  },
  "code": 401
}]
error: You must be logged in to the server (the server has asked for the client to provide credentials)
```

## 原因一： ~/.kube/config 缺失
```
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```
## 原因二： 证书~/.kube/config）时间过期
### 检查证书是否过期
```
kubectl config view --raw -o jsonpath='{.users[0].user.client-certificate-data}' | base64 -d > client.crt
openssl x509 -in client.crt -noout -dates

```
如果过期则在检查下 (*以下是未过期*)
```
sudo kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 May 09, 2034 03:03 UTC   8y              ca                      no
apiserver                  May 09, 2034 03:03 UTC   8y              ca                      no
apiserver-etcd-client      May 09, 2034 03:03 UTC   8y              etcd-ca                 no
apiserver-kubelet-client   May 09, 2034 03:03 UTC   8y              ca                      no
controller-manager.conf    May 09, 2034 03:03 UTC   8y              ca                      no
etcd-healthcheck-client    May 09, 2034 03:03 UTC   8y              etcd-ca                 no
etcd-peer                  May 09, 2034 03:03 UTC   8y              etcd-ca                 no
etcd-server                May 09, 2034 03:03 UTC   8y              etcd-ca                 no
front-proxy-client         May 09, 2034 03:03 UTC   8y              front-proxy-ca          no
scheduler.conf             May 09, 2034 03:03 UTC   8y              ca                      no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      May 08, 2033 09:01 UTC   7y              no
etcd-ca                 May 08, 2033 09:01 UTC   7y              no
front-proxy-ca          May 08, 2033 09:01 UTC   7y              no
```
如上都未过期，则直接 sudo cp /etc/kubernetes/admin.conf ~/.kube/config 即可。

如果过期则按照以下部署：
```
sudo kubeadm certs renew admin.conf
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $USER:$USER ~/.kube/config

```

有个更新脚本：https://github.com/yuyicai/update-kube-cert （使用前需要仔细阅读代码）