---
title: docker 导致 Ping 失败
author: 刘Sir
date: 2025-01-15 10:10:00 +0800
categories: [技术]
tags: [网络]
render_with_liquid: false
---

排查总结：
1. 部分主机无法 ping 目标主机
2. 通过其他主机 ssh 到目标主机，查看目标主机路由规则 ip route
3. docker 自创路由规则导致，删除 docker 网桥和路由规则

## 过程
本地 (172.17.51.109) ssh 192.168.15.235 失败，但是可以通过其他的主机登录。查看 192.168.15.235 路由规则
ip route
```
ubuntu@techteam-dev-1:/etc/network$ ip route
default via 192.168.15.254 dev ens160 proto static
10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1
10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
10.244.3.0/24 via 10.244.3.0 dev flannel.1 onlink
172.17.0.0/16 dev br-b82bfb015996 proto kernel scope link src 172.17.0.1 linkdown
172.99.0.0/16 dev docker0 proto kernel scope link src 172.99.0.1 linkdown
192.168.15.0/24 dev ens160 proto kernel scope link src 192.168.15.235
```
按照规则应该是走 172.17.0.0/16 dev br-b82bfb015996 proto kernel scope link src 172.17.0.1 linkdown，但是网桥 br-b82bfb015996 的状态的 linkdown，重启试试，执行 sudo ip link set br-b82bfb015996 up 没有用。因此我直接删除此路由规则，走默认网桥 ens160。  

删除 sudo ip route del 172.17.0.0/16 dev br-b82bfb015996。 ssh 成功

br-b82bfb015996 这种网桥很大可能是 docker 创建的，使用 docker network ls 查看
```
ubuntu@techteam-dev-1:/etc/network$ docker network ls
NETWORK ID     NAME            DRIVER    SCOPE
45d9000290d3   bridge          bridge    local
dac827f889bb   host            host      local
7bf4d033a849   none            null      local
b82bfb015996   sonar_default   bridge    local
```
 docker network inspect sonar_default 查看创建时间

```
ubuntu@techteam-dev-1:~/sonar$ docker network inspect sonar_default
[
    {
        "Name": "sonar_default",
        "Id": "b82bfb015996f3f8353ff45958d10d87b7d954decb05ab7753d90ea49c0bca15",
        "Created": "2023-10-24T16:06:43.287583231+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": true,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {
            "com.docker.compose.network": "default",
            "com.docker.compose.project": "sonar",
            "com.docker.compose.version": "1.25.0"
        }
    }
]
```

docker 创建自定义网桥
```
docker network rm sonar_default
docker network create --driver bridge --subnet 172.17.0.0/16 --gateway 172.17.51.253 sonar_default
``` 
