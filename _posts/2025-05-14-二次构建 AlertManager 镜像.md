---
title: 二次构建 AlertManager 镜像
author: 刘Sir
date: 2024-05-014 10:10:00 +0800
categories: [技术]
tags: [容器]
render_with_liquid: false
---  

此文档是是描述在二次开发 AM 后，在 Ubuntu 主机中打包镜像的注意事项和过程：
## build 准备
1. 安装 golang 
2. 安装 nodejs
3. 安装 docker
4. 安装 make  sudo apt install make
以下是目前打包环境的各个组件的版本，可以按需调整：
```shell
Docker version 24.0.7, build afdd53b
go version go1.24.2 linux/amd64
node v22.15.0

```

另外在国内 golang 的环境在打包过程中，包的下载问题可以通过以下方式配置：
```shell
$ go env -w GO111MODULE=on
$ go env -w GOPROXY=https://goproxy.cn,direct
```
## 编译源代码
目前选择是前后端一起编译。进入 alertmanager 目录执行 **make build-all**
### 问题一 无法拉取镜像
docker: Error response from daemon: Get  "https://registry-1.docker.io/v2/": context deadline exceeded (Client.Timeout exceeded while awaiting headers 
解决此问题需要增加 docker 的代理， /etc/docker/daemon.json 加入
```text
Registry Mirrors:
  https://docker.registry.cyou/
  https://docker-cf.registry.cyou/
  https://dockercf.jsdelivr.fyi/
  https://docker.jsdelivr.fyi/
  https://dockertest.jsdelivr.fyi/
  https://mirror.aliyuncs.com/
  https://dockerproxy.com/
  https://mirror.baidubce.com/
  https://docker.m.daocloud.io/
  https://docker.nju.edu.cn/
  https://docker.mirrors.sjtug.sjtu.edu.cn/
  https://docker.mirrors.ustc.edu.cn/
  https://mirror.iscas.ac.cn/
  https://docker.rainbond.cc/
  https://do.nark.eu.org/
  https://dc.j8.work/
  https://gst6rzl9.mirror.aliyuncs.com/
  https://registry.docker-cn.com/
  http://hub-mirror.c.163.com/
  http://mirrors.ustc.edu.cn/
  https://mirrors.tuna.tsinghua.edu.cn/
  http://mirrors.sohu.com/
```
**重启 docker 服务： sudo systemctl restart docker**

### 问题二 文件权限
```text
docker run --user=0:0 --rm -t -v /home/ryze/alertmanager/template:/app -w /app alertmanager-template ./inline-css.js
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to line-css.js": permission denied: unknown.

```
修改文件执行权限：chmod +x inline-css.js 解决
### 问题三 webpack-cli 错误
```text
[webpack-cli] Failed to load '/home/ryze/alertmanager/ui/react-app/webpack.prod.ts' config
[webpack-cli] Error: Cannot find module '/home/ryze/alertmanager/ui/react-app/node_modules/ajv-keywords/index.js'. Please verify that the package.json has a v
    at tryPackage (node:internal/modules/cjs/loader:511:19)
    at Function._findPath (node:internal/modules/cjs/loader:796:18)
    at Function.<anonymous> (node:internal/modules/cjs/loader:1387:27)
    at Function.Module._resolveFilename.sharedData.moduleResolveFilenameHook.installedValue [as _resolveFilename] (/home/ryze/alertmanager/ui/react-app/node_m/source-map-support.js:811:30)
    at defaultResolveImpl (node:internal/modules/cjs/loader:1057:19)
    at resolveForCJSWithHooks (node:internal/modules/cjs/loader:1062:22)
    at Function._load (node:internal/modules/cjs/loader:1211:37)
    at TracingChannel.traceSync (node:diagnostics_channel:322:14)
    at wrapModuleLoad (node:internal/modules/cjs/loader:235:24)
    at Module.require (node:internal/modules/cjs/loader:1487:12) {
  code: 'MODULE_NOT_FOUND',
  path: '/home/ryze/alertmanager/ui/react-app/node_modules/ajv-keywords/package.json',
  requestPath: 'ajv-keywords'
}
make: *** [Makefile:37: build-react-app] Error 2

```
进入 /alertmanager/ui/react-app 下面执行： rm -rf node_modules/ package-lock.json  解决
### 问题4 eslint 问题
```text
ERROR in [eslint]
/home/ryze/alertmanager/ui/react-app/src/App.tsx
   1:45  error  Delete `␍`  prettier/prettier
   2:42  error  Delete `␍`  prettier/prettier
   3:31  error  Delete `␍`  prettier/prettier

```
因为代码先在 git clone 在 windows 中，然后 cp 到 ubuntu 可能导致前端代码 eslint 检测不合格 （但实际上我在 windows 上也出现这个错误）。 进入 /alertmanager/ui/react-app/ 下面执行
npm run lint:fix 解决

### 权限不足
```text
webpack 5.99.8 compiled with 2 warnings in 69029 ms
>> compressing assets
scripts/compress_assets.sh
make: scripts/compress_assets.sh: Permission denied
make: *** [Makefile:42: assets-compress] Error 127

```
执行 chmod +x 解决

### 问题5 脚本执行报错
```text
webpack 5.99.8 compiled with 2 warnings in 35635 ms
>> compressing assets
scripts/compress_assets.sh
/usr/bin/env: ‘bash\r’: No such file or directory
make: *** [Makefile:42: assets-compress] Error 127
```
原因是脚本中的头声明中换行导致，windows （dos） 和 unix 系统下换行的导致的， windows 换行是 \r\n , linux macos 这些类 unix 换行是 \n 。 
```shell
#!/usr/bin/env bash
.....
```
  
下载： sudo apt install dos2unix
执行： dos2unix scripts/compress_assets.sh 解决

## 构建镜像
通过上面步骤编译源码后，在 alertmanager 文件夹中会出现多两个可执行文件： alertmanager 和 amtool 。在修改源码中的 Dockerfile 文件后，执行 docker build 即可构建完整镜像。 修改后的 Dockerfile 内容如下：
```dockerfile
ARG ARCH="amd64"
ARG OS="linux"
FROM quay.io/prometheus/busybox-${OS}-${ARCH}:latest
LABEL maintainer="The Prometheus Authors <prometheus-developers@googlegroups.com>"
LABEL org.opencontainers.image.source="https://github.com/prometheus/alertmanager"

ARG ARCH="amd64"
ARG OS="linux"
COPY amtool       /bin/amtool
COPY alertmanager /bin/alertmanager
COPY examples/ha/alertmanager.yml      /etc/alertmanager/alertmanager.yml

RUN mkdir -p /alertmanager && \
    chown -R nobody:nobody /etc/alertmanager /alertmanager

USER       nobody
EXPOSE     9093
VOLUME     [ "/alertmanager" ]
WORKDIR    /alertmanager
ENTRYPOINT [ "/bin/alertmanager" ]
CMD        [ "--config.file=/etc/alertmanager/alertmanager.yml", \
             "--storage.path=/alertmanager" ]

```