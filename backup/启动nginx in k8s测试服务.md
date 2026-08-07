# Nginx测试服务部署文档

## 1\. 概述

一键部署一套轻量化Nginx测试服务，集成Nginx状态监控、Pod全量信息查询、80M大文件下载、客户端IP/Header/网络类型探测、请求调试、故障模拟、跨域测试等运维常用能力，支持IPv4/IPv6双栈监听，通过NodePort对外暴露访问。

整套资源包含：ConfigMap（Nginx自定义配置）、Deployment（应用负载）、NodePort Service（外部访问），无PVC持久化依赖，采用稀疏文件实现零磁盘占用大文件，原生Nginx语法、无第三方模块依赖，适配标准Nginx镜像，集群兼容性极强。

## 2\. 资源清单组成

|资源类型|资源名称|作用说明|
|---|---|---|
|ConfigMap|nginx\-conf|存储自定义nginx\.conf配置，实现多调试接口、IPv4\+IPv6双栈监听、原生语法适配|
|Deployment|nginx\-demo|运行Nginx容器，启动脚本自动生成80M稀疏文件、Pod全量信息静态文件，配置资源限额、滚动更新策略|
|Service\(NodePort\)|nginx\-nodeport\-svc|节点端口30080暴露服务，开启真实客户端IP获取，支持集群内外、IPv4/IPv6访问|

## 3\. 核心功能说明（含访问示例）

### 3\.1 默认Nginx首页

- **访问地址**：`http://<节点IP>:30080`

- **功能**：Nginx原生默认欢迎静态页面，用于验证服务基础可用性、端口连通性。

- **访问示例**：`curl http://<节点IP>:30080`

### 3\.2 Nginx状态监控 /nginx\_status

- **访问地址**：`http://<节点IP>:30080/nginx_status`

- **功能**：启用Nginx原生`stub_status`模块，输出实时连接数、总请求数、读写等待状态，用于监控Nginx运行负载与并发状态。

- **访问示例**：`curl http://<节点IP>:30080/nginx_status`

- **返回内容示例**：

```Plain Text
Active connections: 1
server accepts handled requests
 10 10 24
Reading: 0 Writing: 1 Waiting: 0
```

### 3\.3 Pod全量信息查询 /podinfo/all

- **访问地址**：`http://<节点IP>:30080/podinfo/all`

- **功能**：统一查询当前Pod核心信息，包含Pod内网IP、Pod名称、所在节点、命名空间，替代分散的单字段接口，信息统一无重复。

- **实现原理**：

    1. K8s通过`fieldRef`自动注入POD\_IP、POD\_NAME、NODE\_NAME、NAMESPACE环境变量；

    2. 容器启动脚本读取所有环境变量，写入静态文件`/data/podinfo/all.txt`；

    3. Nginx精确匹配路径读取静态文件返回，规避Nginx无法直接读取系统环境变量的问题。

- **访问示例**：`curl http://<节点IP>:30080/podinfo/all`

- **返回内容示例**：

```Plain Text
POD_IP: 10.42.39.217
POD_NAME: nginx-demo-7f96588d9-2xqkz
NODE_NAME: m1
NAMESPACE: default
```

### 3\.4 80M测试文件下载 /download/80m

- **访问地址**：`http://<节点IP>:30080/download/80m`

- **功能**：提供80MB测试文件下载，用于压测带宽、验证文件传输、链路稳定性。

- **实现特点**：
        

    1. 使用`dd seek`创建**稀疏空洞文件**，逻辑大小80MB，实际不占用集群磁盘空间；

    2. 开启Nginx sendfile零拷贝传输，下载性能优异；

    3. 自动携带下载响应头，浏览器直接触发下载，文件名称`80m-test-file.dat`。

- **存储路径**：EmptyDir 临时卷 `/data/file/80m-test.dat`

- **访问示例**：`curl -OJ http://<节点IP>:30080/download/80m`

### 3\.5 客户端IP查询 /clientip

- **访问地址**：`http://<节点IP>:30080/clientip`

- **功能**：获取当前访问客户端真实IP，用于链路溯源、代理转发验证。

- **访问示例**：`curl http://<节点IP>:30080/clientip`

- **返回示例**：
```
remote_addr:          10.119.8.75
realip_remote_addr:   10.119.8.75
proxy_protocol_addr:  
http_x_real_ip:       10.42.45.26
http_x_forwarded_for: 10.42.45.26
remote_port:          54546
server_addr:          10.119.7.12
server_port:          80
```

### 3\.6 请求头查询 /headers

- **访问地址**：`http://<节点IP>:30080/headers`

- **功能**：输出完整请求头，包含Host、User\-Agent、自定义Header等，用于调试网关、代理、请求转发场景。

- **访问示例**：`curl -H "X-Token: test123" http://<节点IP>:30080/headers`

### 3\.7 网络类型识别 /ipversion

- **访问地址**：`http://<节点IP>:30080/ipversion`

- **功能**：自动识别客户端访问协议，区分IPv4/IPv6，验证双栈监听有效性。

- **访问示例**：`curl http://<节点IP>:30080/ipversion`

### 3\.8 请求元信息查询 /reqinfo

- **访问地址**：`http://<节点IP>:30080/reqinfo`

- **功能**：输出请求完整元数据，包含请求方法、URI、参数、协议、时间戳，用于路由调试、参数校验。

- **访问示例**：`curl http://<节点IP>:30080/reqinfo?a=1&b=2`

### 3\.9 服务器时间戳 /time

- **访问地址**：`http://<节点IP>:30080/time`

- **功能**：返回服务器标准时间、毫秒级时间戳，用于时钟同步、请求耗时排查。

- **访问示例**：`curl http://<节点IP>:30080/time`

### 3\.10 故障码模拟 /error/xxx

- **访问地址**：`http://<节点IP>:30080/error/404`、`/error/429`、`/error/500`、`/error/502`、`/error/503`

- **功能**：模拟常用HTTP异常状态码，用于测试熔断、重试、告警、负载均衡异常调度。

- **访问示例**：`curl -i http://<节点IP>:30080/error/503`

### 3\.11 响应头测试 /resp\-header

- **访问地址**：`http://<节点IP>:30080/resp-header`

- **功能**：返回自定义响应头，用于验证下游服务获取响应头、网关透传头部能力。

- **访问示例**：`curl -I http://<节点IP>:30080/resp-header`

### 3\.12 CORS跨域调试 /cors

- **访问地址**：`http://<节点IP>:30080/cors`

- **功能**：开启通用跨域配置，支持OPTIONS预检请求，用于前后端联调跨域问题排查。

- **访问示例**：`curl -X OPTIONS http://<节点IP>:30080/cors`

### 3\.13 IPv4\+IPv6双栈监听

Nginx原生配置开启双栈监听，兼容全网段访问：

- `listen 80;` 监听IPv4 0\.0\.0\.0:80

- `listen [::]:80;` 监听IPv6 \[::\]:80

容器可同时响应IPv4、IPv6客户端请求，适配双栈集群环境测试。

## 4\. 关键设计说明

1. **无第三方模块兼容**：镜像`nginx:1.23.1`无Lua、echo等第三方模块，全部采用Nginx官方原生指令实现所有功能，零依赖、零启动报错。

2. **规避配置兼容问题**：
       


    - 统一使用精确路径匹配\+alias规则，消除404路径冲突、root/alias共存冲突；

    - 废弃变量返回状态码，固定故障码接口，适配原生Nginx语法。

3. **轻量化存储设计**：80M文件为稀疏空洞文件，逻辑大容量、实际零磁盘占用；所有临时数据存储EmptyDir临时卷，无需PVC持久化，部署极简、无资源冗余。

4. **无重复接口设计**：精简冗余接口，仅保留Pod信息汇总入口，各接口职责单一、输出内容不重叠，调试更清晰。

5. **集群调度优化**：配置合理滚动更新策略，支持平稳重启不中断服务；Service开启真实客户端IP透传(svc中#externalTrafficPolicy: Local)，满足链路溯源需求。

6. **资源安全限制**：容器配置CPU/内存上下限，避免资源抢占，保障集群稳定性。

## 5\. 接口访问汇总清单

|接口路径|核心用途|访问示例|
|---|---|---|
|/|Nginx默认首页，验证服务连通性|curl http://\<节点IP\>:30080|
|/nginx\_status|Nginx连接、请求状态监控|curl http://\<节点IP\>:30080/nginx\_status|
|/podinfo/all|查询Pod、节点、命名空间全量信息|curl http://\<节点IP\>:30080/podinfo/all|
|/download/80m|80M大文件下载，带宽/链路压测|curl \-OJ http://\<节点IP\>:30080/download/80m|
|/clientip|获取客户端真实访问IP|curl http://\<节点IP\>:30080/clientip|
|/headers|查看完整请求头，代理调试|curl \-H "X\-Token: test" http://\<节点IP\>:30080/headers|
|/ipversion|识别客户端IPv4/IPv6访问类型|curl http://\<节点IP\>:30080/ipversion|
|/reqinfo|查询请求方法、参数、协议等元数据|curl http://\<节点IP\>:30080/reqinfo?a=1|
|/time|获取服务器标准时间戳|curl http://\<节点IP\>:30080/time|
|/error/xxx|模拟404/500/503等故障码|curl \-i http://\<节点IP\>:30080/error/503|
|/resp\-header|测试自定义响应头返回|curl \-I http://\<节点IP\>:30080/resp\-header|
|/cors|跨域调试、OPTIONS预检测试|curl \-X OPTIONS http://\<节点IP\>:30080/cors|

## 6\. 完整部署YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: default
data:
  nginx.conf: |
    worker_processes  auto;
    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;
        sendfile        on;
        keepalive_timeout  65;

        server {
            listen       80;
            listen       [::]:80;
            server_name  localhost;

            # 1. 默认nginx欢迎页面
            location / {
                root   /usr/share/nginx/html;
                index  index.html index.htm;
            }

            # 2. nginx_status 监控接口
            location /nginx_status {
                stub_status;
                allow all;
            }

            # 3. 汇总Pod全量信息
            location = /podinfo/all {
                default_type text/plain;
                alias /data/podinfo/all.txt;
            }

            # 4. 80M测试文件下载接口
            location /download {
                root /data/file;
                rewrite ^/download/80m$ /80m-test.dat break;
                add_header Content-Disposition "attachment; filename=\"80m-test-file.dat\"";
            }

            # 5. 客户端真实IP查询
            location = /clientip {
                default_type text/plain;
                # return 200 "$remote_addr\n";
                return 200 "remote_addr:          $remote_addr\nrealip_remote_addr:   $realip_remote_addr\nproxy_protocol_addr:  $proxy_protocol_addr\nhttp_x_real_ip:       $http_x_real_ip\nhttp_x_forwarded_for: $http_x_forwarded_for\nremote_port:          $remote_port\nserver_addr:          $server_addr\nserver_port:          $server_port\n";

            }

            # 6. 完整请求头查询
            location = /headers {
                default_type text/plain;
                return 200 "===== Request Headers =====\nHost: $host\nUser-Agent: $http_user_agent\nReferer: $http_referer\nX-Forwarded-For: $http_x_forwarded_for\nCookie: $http_cookie\nCustom X-Token: $http_x_token\n";
            }

            # 7. IPv4/IPv6网络类型识别
            location = /ipversion {
                default_type text/plain;
                if ($server_addr ~ ":") {
                    return 200 "Client Type: IPv6\n";
                }
                return 200 "Client Type: IPv4\n";
            }

            # 8. 请求元信息查询
            location = /reqinfo {
                default_type text/plain;
                return 200 "Method: $request_method\nURI: $request_uri\nArgs: $args\nProtocol: $server_protocol\nISO Time: $time_iso8601\nUnix Msec: $msec\n";
            }

            # 9. 固定故障码模拟接口
            location = /error/404 { return 404; }
            location = /error/429 { return 429; }
            location = /error/500 { return 500; }
            location = /error/502 { return 502; }
            location = /error/503 { return 503; }

            # 10. 自定义响应头测试
            location = /resp-header {
                default_type text/plain;
                add_header X-Request-Path $request_uri always;
                add_header X-Demo-Resp-Val test-demo always;
                return 200 "Use curl -I to view response headers\n";
            }

            # 11. 服务器时间戳查询
            location = /time {
                default_type text/plain;
                return 200 "ISO Time: $time_iso8601\nUnix Msec: $msec\n";
            }

            # 12. CORS跨域调试
            location = /cors {
                default_type text/plain;
                add_header Access-Control-Allow-Origin *;
                add_header Access-Control-Allow-Methods GET,POST,OPTIONS;
                if ($request_method = OPTIONS) {
                    return 204;
                }
                return 200 "CORS test ok\n";
            }
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: default
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  replicas: 3
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx-demo
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx
        image: registry.kylincloud.org:4001/kcc/images/nginx:1.23.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        command: ["/bin/sh","-c"]
        args:
        - |
          mkdir -p /data/file /data/podinfo
          dd if=/dev/zero of=/data/file/80m-test.dat bs=1M count=0 seek=80
          echo -n "$POD_IP" > /data/podinfo/ip.txt
          echo -n "$POD_NAME" > /data/podinfo/name.txt
          cat > /data/podinfo/all.txt <<EOF
          POD_IP: $POD_IP
          POD_NAME: $POD_NAME
          NODE_NAME: $NODE_NAME
          NAMESPACE: $NAMESPACE
          EOF
          exec nginx -g "daemon off;"
        volumeMounts:
        - name: nginx-conf-volume
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: data-empty-dir
          mountPath: /data
        resources:
          limits:
            cpu: "0.5"
            memory: 512Mi
          requests:
            cpu: "0.1"
            memory: 128Mi
      volumes:
      - name: nginx-conf-volume
        configMap:
          name: nginx-conf
      - name: data-empty-dir
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-svc
  namespace: default
spec:
  selector:
    app: nginx-demo
  type: NodePort
  #ipFamilyPolicy: PreferDualStack
  #externalTrafficPolicy: Local
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test
spec:
  ingressClassName: ingress-nginx
  rules:
  - host: test.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-nodeport-svc
            port:
              number: 80
```