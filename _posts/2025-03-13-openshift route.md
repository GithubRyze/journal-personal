---
title: openshift route
author: 刘Sir
date: 2024-03-14 10:10:00 +0800
categories: [技术]
tags: [K8S]
render_with_liquid: false
---  

在 K8S 的测试环境中我们使用 nginx ingress 配置可以支持同一个域名同时使用 HTTP 和 HTTPS 访问，配置如下：
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: test
  labels:
    app: ingress-api
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 100m
    nginx.ingress.kubernetes.io/proxy-connect-timeout: 900s
    nginx.ingress.kubernetes.io/proxy-read-timeout: 900s
    nginx.ingress.kubernetes.io/proxy-send-timeout: 900s
    # 指定 Nginx Ingress 和 后端服务沟通的协议
    #nginx.ingress.kubernetes.io/backend-protocol: "https"
    #ssl 客户端重定向（当客户端使用 http 请求，会返回重定向 308 状态）
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "false"
spec:
  rules:
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service:
            name: backend-service
            port:
              number: 8083  # 另一个服务的端口
  # tls 配置是为了 启用 HTTPS，确保 传入的 HTTPS 请求能够使用 TLS 证书进行加密，但它并不会强制客户端必须使用 HTTPS
  tls:
  - hosts:
    - www.example.com
    secretName: api-secret
```
实际就是使用 nginx annotations 中的 ssl-redirect 和 force-ssl-redirect 关掉 http 转发 https，以上部署到 openshift 后发现失效了。找资料后发现 openshift 平台负责网络配置是 route。   
```
[root@svsyhisb2001 nginx-ingress]# oc get route -n platform
NAME                                      HOST/PORT                                               PATH   SERVICES                       PORT                   TERMINATION          WILDCARD
mdp-api-route                             his-uat2-integrate.uat.cmhhk.org.hk                     /      mdp-server-service             8083                   edge/Allow           None
```
openshift 在使用 kubectl apply -f ingres.yaml 后。openshift 会把这个资源转换成 route。在一些其他的资料和我测试后发现要实现 http 和 https 同时访问  route 资源中的 TERMINATION 必须是 edge/Allow。 但是 openshift 在创建 https ingress 转换 route 后的默认值是 edge/Redirect。因此开始我想使用在 ingress 中使用 annotations 的方式去处理，在找到对应的 openshift 文档（https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/networking/configuring-routes#nw-route-specific-annotations_route-configuration） 发现了相关资料，配置 insecureEdgeTerminationPolicy 这个字段是控制 HTTPS 转发的，指是 Allow、None、Redirect（默认值）。但是没有 annotations 可以配置，还在开发的过程当中。目前只能使用 route 的方式解决
```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: api-route
  namespace: test
spec:
  host: www.example.com
  path: /
  to:
    kind: Service
    name: backend-service
  port:
    targetPort: 8083
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Allow
    secretName: api-secret  # 使用 Kubernetes Secret 存储证书
```
测试
```
curl -k -X GET https://10.27.224.6/test/test -H "Host: www.example.com"
curl -v GET http://10.27.224.6/test/test -H "Host: www.example.com"
```