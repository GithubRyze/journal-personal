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
**使用以下方式创建 route**
```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: mdp-thirdparty-web-route
  namespace: platform
spec:
  host: his-uat2-web-sr-integrate.uat.cmhhk.org.hk
  to:
    kind: Service
    name: mdp-third-part-front-service
  port:
    targetPort: 80
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Allow
    certificate: |-
      -----BEGIN CERTIFICATE-----
      MIIDZzCCAk+gAwIBAgIURPYado89dZsuAGZYepFBtDawCJIwDQYJKoZIhvcNAQEL
      BQAwQjELMAkGA1UEBhMCKzAxFTATBgNVBAcMDERlZmF1bHQgQ2l0eTEcMBoGA1UE
      CgwTRGVmYXVsdCBDb21wYW55IEx0ZDAgFw0yNDExMTgxMDE4NTlaGA8yMTI0MTAy
      NTEwMTg1OVowQjELMAkGA1UEBhMCKzAxFTATBgNVBAcMDERlZmF1bHQgQ2l0eTEc
      MBoGA1UECgwTRGVmYXVsdCBDb21wYW55IEx0ZDCCASIwDQYJKoZIhvcNAQEBBQAD
      ggEPADCCAQoCggEBALgdKjjf9X9t7G0ZToA2gBfL50rUI6BKJ4V7qMZ8ltDwc/E8
      gbQ1or+NzrTbbsxbSGjngAT+UIQqqojYjnNMLXXatnt29O49qtW2YvEazMTp4pcy
      YMnAHb72/SEyXH1KQVGs5BJtyauS0gkF8+87ezs4ACG5rPNvIouPwm9XjrCOxtXb
      wqDiPA62xMhkERj2kUDiZmOeZB6sBGf4bTxwpq+varekL5FTdUGq8e6S3fgljEOe
      U9WMoqJSbN1AlTmwZhSbKZ3VAJfCAlZh/b8doqvdK16lXDvSBSPc7OxIkrmRjOQh
      g9G13vPliMkTcFlvORwZ98Owh3qUdVLu+OuF3rcCAwEAAaNTMFEwHQYDVR0OBBYE
      FKvy4jgrc8OKxAJ8+V9niXswLgT0MB8GA1UdIwQYMBaAFKvy4jgrc8OKxAJ8+V9n
      iXswLgT0MA8GA1UdEwEB/wQFMAMBAf8wDQYJKoZIhvcNAQELBQADggEBAGaZAJ5b
      LWAWYs2BQUHzMXS/RFIrKCMEUNhlxo3lSI2n/fCCOk+NS9cU5/SXtNImX2P1Qi7r
      AKvDFmsRrKELaKD/jxiiLa+q3zEB+EAS43oGuuSlkR6w7yG3Qwn+Eq0oZe1ZVI08
      2YAvCw1EIOL9hbZ4N7MZTszc2sHso4O3H0jxqHeZT2/KCP84CzlCUeMIHc5KsV0N
      K9APUl2VIG+B0K1kya+IcrBT95ozNKlWS/FnxxHH4DTkviBediy3Syk32U4qKfM+
      zYzEwAlVIMDj84RdcYia2XxtDBqYAmHf4rQoVjw1y0lt6p9wVXKFBOUEvmhPZoui
      +CgUTPhL/ndvUUc=
      -----END CERTIFICATE-----
    key: |-
      -----BEGIN PRIVATE KEY-----
      MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQC4HSo43/V/bext
      GU6ANoAXy+dK1COgSieFe6jGfJbQ8HPxPIG0NaK/jc60227MW0ho54AE/lCEKqqI
      2I5zTC112rZ7dvTuParVtmLxGszE6eKXMmDJwB2+9v0hMlx9SkFRrOQSbcmrktIJ
      BfPvO3s7OAAhuazzbyKLj8JvV46wjsbV28Kg4jwOtsTIZBEY9pFA4mZjnmQerARn
      +G08cKavr2q3pC+RU3VBqvHukt34JYxDnlPVjKKiUmzdQJU5sGYUmymd1QCXwgJW
      Yf2/HaKr3StepVw70gUj3OzsSJK5kYzkIYPRtd7z5YjJE3BZbzkcGffDsId6lHVS
      7vjrhd63AgMBAAECggEABbIlScAn/Kq+azuisZW+Db5pp+d7OKzCnz8NoCJfmhQu
      ShLIonYcVFfDQtYdDeZvDYvH6p+hhw043GXytj9vkptTFOu/tRUkUVtEoVfmaNsh
      fvL4ipVOdkd22k2QDfI7pha2sZlC6XNv2wabntWUwObBHkn7v0Y7Z9zpM3+ecvjS
      XvNJEvlOBKwAxHtedsx4u0FcORvIg+F7f2ERSq+s5h1heH0HQTibmeGyx40R67tk
      LYcyICTikYI9OLaswkWiLVHiJm1Aj88+dQTMeWqWDLOnviqzOycuur8+T8YdrwyV
      EGV+a1t9pvnhto8EiHNj89e+Ef2xl+Pzygi8/DUaPQKBgQDTmsoPzS9MQYCK/4YX
      p8TvTfKO9bfj0R0tqQo+/YUJKR8lmO5E5hBLwdr2HZQnKSRGfTxIWfrIXmV9oyP8
      Aip4pfyUNOQWUoA4q0MQ0N+f5EAke+2WoV8VORlNxQRJaQ2ayIC97BAgumfj+dLM
      Zdcu+f9hWYKNdEM36VG2WxFBewKBgQDevdvYCsPrkyC0lAIn4tbogHeT8fi7Dt5G
      MPA+D1Ky9DA8TErARqW7xvzJd/95qKT8LFtlh2/shhyyP11gFtWbPX7E2JIXp5Pz
      rl8RhR6zo5oxZVhUTM6+74ChnQRWzcx4VEAD2SidRhzm2mXQRi4dzQpfpl/G8Lhw
      G/Vjx05c9QKBgQCd9Ijq/LZWzPqAR7e4BsNKAzySHLgVNi76u5lrZeGd8fVPInaS
      Nc5qTp39ZB0IknyCpc+PDqPWWCiYfWRKJO/BOd5uX4D3bMRMCQO6FMEpYL/EvEGh
      VHpepE3QMMY/akz+grcfjoyHcb5FfmIts8PKTFGnQKLkpqnana8iWZ5q6wKBgQCe
      Iqyx9PKjjRDrIylkp+drqck7f87W/vdPXe8yIC7WXgbgyElJuN5xMhTF9Cusc86u
      Oc+phT4w7gXxhosEbTG5xS77EcamhJLFrkZQafIiF0ShTRIox18Ar4jjNFagSfug
      cjAAi2wKPHzIaMVf2HNlNOzoe3YEB2LqNO9Cp307AQKBgD0GqYxsOFn51Wpivj4U
      7Tsn0basA62FyInyl0M4A2x9P086K3tS8Rzv4ss+pO7UB/jqVt4upycHXDXtc+Bk
      T/YRKUIX1Vll2ZWZ8uhST9wIzyMtOwNNM2gtBUTMFMrOpJ6SfTbt9UiuLZ2shLOk
      eC9MmNeTrzP3pHiTMrmNpJGK
      -----END PRIVATE KEY-----
```
测试：
openssl  s_client --connect his-uat2-web-sr-integrate.uat.cmhhk.org.hk:443 --showcerts  
curl -k https://his-uat2-web-sr-integrate.uat.cmhhk.org.hk:443/servicer/user_login  
curl --resolve his-uat2-web-integrate.uat.cmhhk.org.hk:443:10.27.224.6 -k https://his-uat2-web-integrate.uat.cmhhk.org.hk:443/user_login
