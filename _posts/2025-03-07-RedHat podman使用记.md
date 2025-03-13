---
title: RedHat podman使用记无法访问外网
author: 刘Sir
date: 2024-03-07 10:10:00 +0800
categories: [技术]
tags: [容器]
render_with_liquid: false
---  

有个应用需要使用 podman 部署到 Red Hat Enterprise Linux release 9.4 环境下。本次记录下遇到的问题。
## SSH 终端退出，容器服务也退出
发现是 Linux 系统中登录用户管理配置导致的。loginctl 作为 Systemd 的一部分，是 systemd 的登录管理器。其中有个 enable-linger 配置按照文档描述：启用/禁止用户逗留(相当于保持登录状态)。 如果指定了用户名或UID， 那么系统将会在启动时自动为这些用户派生出用户管理器， 并且在用户登出后继续保持运行。 这样就可以允许未登录的用户在后台运行持续时间很长的服务。 如果没有指定任何参数， 那么将作用于当前调用者的用户。使用以下命令检测或者开启 enable-linger。

```
loginctl show-user $(whoami) | grep Linger
loginctl enable-linger $(whoami)
```
## 服务无法访问
应用部署好后，在其他主机上面不能访问。
1. ping IP  - 网络层 （ICMP 协议）
2. tracert IP  - 网络层 （ICMP 协议）
3. traceroute -T -p 443 - 追踪 TCP 路由
4. telnet - 测试端口连通性
5. nmap  - 扫描端口
6. 防火墙 （iptables，firewall-cmd，ufw）  

在有些网络环境禁止了 ICMP 协议，那么就不能简单的使用 ping 和 tracert 或者 traceroute 等测试，需要通过判断是否能连接到目标机器的 IP + Port 来判断了。使用 telnet nmap 或者 traceroute 等工具访问目前机器的 443 80 22 端口。再一个就是查看防火墙是否开放了目标端口的访问。这次是 firewall-cmd 未开启 8443 端口导致的。最终通过以下命令解决
```
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```
网络访问测试：curl -v -L -k --resolve csms-pda-uat.uat.cmhhk.org.hk:8443:10.27.224.19 https://csms-pda-uat.uat.cmhhk.org.hk:8443
## 磁盘挂载问题
应用需要把日志和一些配置挂载到容器中去，总是遇到权限问题，正好这次回顾下 linux 文件权限系统。

### linux 文件权限基础
```
[user@svsycsma2001 ~]$ ls -l
total 0
drwxr-x---. 6 user users 56 Mar  4 19:05 test

d 表示这是一个目录，- 表示文件，l 表示文件链接
rwx 依次 读、写、可执行的权限
```
- rwx 表示文件或者目录所有者权限
- r-x 表示用户组权限
- ‘---’ 表示其他用户的权限
- . SELinux 扩展属性标志, 文件可能有 SELinux 的安全上下文（可以用 ls -Z 查看）

用户和用户组的关系，通过权限控制（rwx）来管理用户访问文件的权限
- 用户（User） 是系统中的独立账户。
- 用户组（Group） 用于管理多个用户的权限。
- 每个用户属于一个主组，还可以加入多个附加组。
### Docker 运行用户
- 容器中的进程默认以 root 用户运行
- 在构建镜像的时候 dockerfile 指定运行用户
```
# 创建一个新用户并使用它运行容器
RUN useradd -m appuser
USER appuser
```
- docker run 指令指定用户 
```
docker run --user 1001 myimage
docker run --user appuser myimage
docker run --user $(id -u user):$(id -g user) 当前宿主的用户和用户组

```
### --privileged=true

默认情况下，Docker 容器运行时是受限制的，不能访问某些设备和系统功能, 使用 --privileged=true 命令赋予容器几乎和宿主机 root 用户一样的权限。这样对有些特殊的容器需要访问宿主机硬件设备、执行系统级操作的容器提供了处理。可想而知如果使用不当也可以给宿主造成很大破坏，同时可以使用 --cap-add 开启特定的权限而不是整个 root 权限。
- --privileged=true 让容器获得宿主机几乎所有权限，可能容器中存在恶意代码，破坏系统安全
- 可以使用 --cap-add 代替 --privileged，只开启必要的能力。
- 在 Kubernetes 中，应使用 securityContext 控制容器权限。

### SELinux 的安全上下文
SELinux 是一个强制访问控制 (MAC) 系统，它通过标签控制对文件和资源的访问。当你挂载宿主机上的目录或文件到容器中时，SELinux 会使用安全上下文来控制哪些进程可以访问这些资源。
原文件夹内容
```
[ewelluser@svsycsma2001 test]$ ls -l
total 16
-rw-r--r--. 1 ewelluser users    0 Mar  7 14:39 access.log
-rw-r--r--. 1 ewelluser users 1380 Mar  7 14:39 error.log
-rw-r-----. 1 ewelluser users  181 Mar  7 11:58 nginx.conf
-rw-r-----. 1 ewelluser users 1448 Mar  7 12:02 nginx.crt
-rw-------. 1 ewelluser users 1704 Mar  7 12:03 nginx.key
```
测试挂载文件不加 Z
```
[ewelluser@svsycsma2001 test]$ podman run  --replace --name nginx-test -v /home/ewelluser/test/:/etc/nginx/certs/  localhost/nginx:1.27.4 sh -c "ls -l /etc/nginx/certs/"
ls: cannot open directory '/etc/nginx/certs/': Permission denied
```
挂载文件加 Z
```
[ewelluser@svsycsma2001 test]$ podman run  --replace --name nginx-test -v /home/ewelluser/test/:/etc/nginx/certs/:Z  localhost/nginx:1.27.4 sh -c "ls -l /etc/nginx/certs/"
total 16
-rw-r--r--. 1 root root    0 Mar  7 06:39 access.log
-rw-r--r--. 1 root root 1380 Mar  7 06:39 error.log
-rw-r-----. 1 root root  181 Mar  7 03:58 nginx.conf
-rw-r-----. 1 root root 1448 Mar  7 04:02 nginx.crt
-rw-------. 1 root root 1704 Mar  7 04:03 nginx.key
```
通过以上可以发现两个问题：
- 主机文件挂载到容器后 用户和用户组变成 root
- 挂载加 Z 后，权限问题解决了
### 加 Z 权限
检测主机是否开启了 SELinux
```
[ewelluser@svsycsma2001 test]$ getenforce
Enforcing
```
Enforcing，SELinux 正在强制执行安全策略，可能会阻止访问不符合要求的文件。如果是 Permissive，则会记录警告信息，但不会阻止操作；如果是 Disabled，SELinux 完全禁用，不会阻止任何访问。  

文件的 SELinux 上下文标签
```
[ewelluser@svsycsma2001 test]$ ls -lZ
total 16
-rw-r--r--. 1 ewelluser users system_u:object_r:container_file_t:s0:c421,c644    0 Mar  7 14:39 access.log
-rw-r--r--. 1 ewelluser users system_u:object_r:container_file_t:s0:c421,c644 1380 Mar  7 14:39 error.log
-rw-r-----. 1 ewelluser users system_u:object_r:container_file_t:s0:c421,c644  181 Mar  7 11:58 nginx.conf
-rw-r-----. 1 ewelluser users system_u:object_r:container_file_t:s0:c421,c644 1448 Mar  7 12:02 nginx.crt
-rw-------. 1 ewelluser users system_u:object_r:container_file_t:s0:c421,c644 1704 Mar  7 12:03 nginx.key
```
SELinux 上下文包含用户，角色，类型三个部分。system_u:object_r:container_file_t:s0:c421,c644
- system_u： 用户
- object_r： 角色
- container_file_t： 类型，容器文件类型

在 test 文件夹父目录新增的文件 test.conf  的 SELinux 上下文和 system_u:object_r:container_file_t 不一样，如下：
```
-rw-r-----. 1 ewelluser users unconfined_u:object_r:user_home_t:s0              0 Mar  7 15:12 test.conf  
```
test.conf 是在 用户的 home 目录（如 /home/ewelluser/） 下创建的，而 SELinux 规则会默认赋予 home 目录中的文件 user_home_t 类型, 这个时候在 SELinux下容器是没办法访问的，因为容器的 SELinux 上下文(system_u:object_r:container_file_t:s0:c421,c644) 不一样。

- :Z 标志用于告诉 Docker，将挂载的目录或文件标记为容器专用，并为容器分配一个新的 SELinux 上下文标签。这意味着，文件将被标记为仅对该容器可访问，而不会与其他容器共享。这有助于避免不同容器之间的安全冲突。
- :z 共享模式，如果多个容器要访问同一目录，可以使用

### 挂载权限问题总结
1. 区分文件系统，主要用户和用户组对文件的读写、执行权限
2. 在开启 SELinux 的主机上面，挂载文件操作需要假如 :z 或者 :Z 标识
3. 在开启 SELinux 的主机，SELinux 控制的优先级高于文件权限控制，即：文件可以授予用户操作，但是不符合 SELinux 规则，同样会失败
4. 主机文件挂载到容器后，用户和用户组会变成 root，跟使用 docker run --user 指定的用户没有关系
5. --privileged=true 让容器获得宿主机几乎所有权限，风险比较高，可使用 --cap-add 代替

## ngixn resolver
nginx 代理只会在启动的时候去 dns 解析服务 IP, 然后缓存。这样就会存在以下两种场景
1. 当代理的服务没有启动，nginx 无法解析到 IP，会报错
2. IP 变更后，nginx 不会更新

docker 和 podman 的网络中，容器重启后 IP 地址会变，因此会导致 nginx 502 和 499 错误。因此需要使用 resolver 间隔时间刷新 IP 地址，配置如下
```
server {
    listen 443 ssl;
    server_name csms-pda-uat.uat.cmhhk.org.hk;
    ssl_certificate /etc/nginx/certs/nginx.crt;
    ssl_certificate_key /etc/nginx/certs/nginx.key;
    location / {
        resolver 10.89.0.1 valid=10s; # 这里
        set $pda_backend "csms-pda-server:30029";
        proxy_pass http://$pda_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

}
```

## 反向代理 kecloak 记录
使用 https 代理 http 的 keycloak 服务配置如下。参考：https://www.keycloak.org/server/reverseproxy
- nginx 的 X-Forwarded 变量
- 以及 keycloak 启动命令中 --proxy-headers xforwarded --hostname https://keycloak.uat.cmhhk.org.hk:8443

nginx 代理配置
```
server {
    listen 443 ssl;
    server_name keycloak.uat.cmhhk.org.hk;
    ssl_certificate /etc/nginx/certs/nginx.crt;
    ssl_certificate_key /etc/nginx/certs/nginx.key;
    location / {
        resolver 10.89.0.1 valid=10s;
        set $backend "keycloak:8080";
        proxy_pass http://$backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host keycloak.uat.cmhhk.org.hk;
        proxy_set_header X-Forwarded-Port 8443;
    }
}
```
keycloak 启动命令
```
podman run -d -u root \
  --replace --name keycloak \
  --network csms-network \
  -v /home/ewelluser/csms/log/keycloak:/opt/keycloak/data/log:Z \
  -e KC_DB=mssql \
  -e KC_DB_URL="jdbc:sqlserver://10.27.224.15:1433;databaseName=TP_CSMS_K;encrypt=true;trustServerCertificate=true" \
  -e KC_DB_USERNAME=csms \
  -e KC_DB_PASSWORD=pb8_Sx1wdChG2K \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  -e KC_LOG=file,console \
  -e KC_LOG_FILE=/opt/keycloak/data/log/keycloak.log \
  -e PROXY_ADDRESS_FORWARDING=true \
  -e KEYCLOAK_FRONTEND_URL="https://keycloak.uat.cmhhk.org.hk:8443" \
  -p 8081:8080 \
  localhost/keycloak/keycloak:26.1.3 start-dev --proxy-headers xforwarded --hostname https://keycloak.uat.cmhhk.org.hk:8443
```