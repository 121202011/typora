## 1. 前置准备：文件与环境确认

| 服务名 | 下载地址                            | 备注 |
| ------ | ----------------------------------- | ---- |
| nginx  | https://hub.docker.com/_/nginx/tags |      |

### 1.1 需上传的文件清单

将以下文件上传至服务器（建议统一存放于临时目录，如 `/tmp/nginx-files`，后续会迁移至正式路径）：

- `docker-compose.yml`：Docker 容器编排配置文件
- `nginx.conf`：Nginx 主配置文件（全局参数）
- `nginx_1.22.1-alpine-x86.tar.gz`：Nginx 镜像压缩包（本地导入用）
- `default.conf`：Nginx 业务路由配置文件（后端服务转发规则）

### 1.2 依赖环境要求

- 服务器已安装 Docker（建议 20.10+ 版本）
- 服务器已安装 Docker Compose（建议 2.10+ 版本）
- 确保 80 端口未被占用（可通过 `netstat -tuln | grep 80` 检查）

## 2. 初始化目录结构

创建 Nginx 正式工作目录，用于存储配置、日志等数据（目录路径可自定义，此处以 `/opt/nginx` 为例）：

```bash
# 创建主目录及子目录
mkdir -p /opt/nginx/{conf.d,logs}
# 说明：
# - /opt/nginx：主目录
# - /opt/nginx/conf.d：业务路由配置目录（存放 default.conf）
# - /opt/nginx/logs：日志存储目录（挂载容器内日志）
```

## 3. 生成核心配置文件

### 3.1 容器编排文件（docker-compose.yml）

```bash
cat > /opt/nginx/docker-compose.yml << EOF
version: '3.8'  # Docker Compose 版本（适配主流 Docker 环境）
services:
  nginx:
    image: harbor.ciicsh.com/library/nginx:1.22.1-alpine  # 镜像地址（私有仓库）
    container_name: nginx  # 容器固定名称（便于后续管理）
    privileged: true       # 授予容器特权（避免文件权限问题）
    network_mode: host     # 使用主机网络（端口直接映射，无需额外暴露）
    restart: always        # 容器异常退出时自动重启（高可用基础）
    volumes:               # 宿主机与容器目录挂载（实现数据持久化）
      - /opt/nginx/nginx.conf:/etc/nginx/nginx.conf  # 主配置文件挂载
      - /opt/nginx/conf.d:/etc/nginx/conf.d          # 业务配置目录挂载
      - /opt/nginx/logs:/var/log/nginx              # 日志目录挂载
#     - /opt/nginx/certs:/opt/ciic-ssl/ciicsh.com    # SSL 证书挂载（HTTPS 时启用）
    environment:
      - TZ=Asia/Shanghai  # 配置容器时区（与宿主机一致，避免日志时间偏差）
EOF
```

### 3.2 Nginx 主配置文件（nginx.conf）

```bash
cat > /opt/nginx/nginx.conf <<EOF
# 1. 基础运行参数
user  nginx;                  # Nginx 运行用户（容器内默认用户）
worker_processes  auto;       # 工作进程数（自动匹配 CPU 核心数，优化并发）
error_log  /var/log/nginx/error.log warn;  # 错误日志（warn 级别，减少冗余）
pid        /var/run/nginx.pid;              # 进程 PID 文件路径

# 2. 事件驱动配置
events {
    worker_connections  5000;  # 单个工作进程最大连接数（适配中高并发场景）
}

# 3. HTTP 核心配置
http {
    include       /etc/nginx/mime.types;  # 引入 MIME 类型映射（识别文件格式）
    default_type  application/octet-stream;  # 默认 MIME 类型（未知文件时使用）

    # 3.1 日志格式配置
    # 文本格式日志（便于人工阅读）
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    # JSON 格式日志（便于日志系统解析，如 ELK 栈）
    log_format access_json_log  '{"@timestamp":"$time_iso8601",'
    '"host":"$hostname",'
    '"server_ip":"$server_addr",'
    '"client_ip":"$remote_addr",'
    '"xff":"$http_x_forwarded_for",'  # 客户端真实 IP（反向代理场景）
    '"domain":"$host",'
    '"url":"$uri",'
    '"referer":"$http_referer",'
    '"args":"$args",'  # URL 请求参数
    '"upstreamtime":"$upstream_response_time",'  # 后端服务响应时间
    '"responsetime":"$request_time",'  # 总请求响应时间（含网络耗时）
    '"request_method":"$request_method",'  # 请求方法（GET/POST/PUT 等）
    '"status":"$status",'  # HTTP 状态码（200/404/500 等）
    '"size":"$body_bytes_sent",'  # 响应体大小（字节）
    '"request_length":"$request_length",'  # 请求总长度（字节）
    '"protocol":"$server_protocol",'  # HTTP 协议版本（HTTP/1.1/HTTP/2）
    '"upstreamhost":"$upstream_addr",'  # 后端服务地址（IP:端口）
    '"file_dir":"$request_filename",'  # 请求对应的本地文件路径
    '"http_user_agent":"$http_user_agent"'  # 客户端浏览器/设备信息
    '}';

    # 3.2 日志输出配置
    access_log  /var/log/nginx/access.log  main;  # 主访问日志（文本格式）

    # 3.3 性能优化参数
    sendfile        on;  # 启用零拷贝技术（提升文件传输效率，减少 CPU 占用）
    keepalive_timeout  1800;  # 长连接超时时间（30 分钟，适配后端长连接场景）
    server_names_hash_bucket_size 512;  # 服务器名哈希表大小（适配多域名场景）

    # 3.4 客户端请求缓冲配置（避免频繁读写磁盘）
    client_body_buffer_size  50m;    # 请求体缓冲区大小（适配大请求体）
    client_header_buffer_size 20m;   # 请求头缓冲区大小
    client_max_body_size 100m;       # 客户端最大请求体大小（限制大文件上传）
    large_client_header_buffers 2 20m;  # 大请求头缓冲区（2 个 20M 缓冲区）

    # 3.5 超时控制配置（避免连接占用资源）
    client_body_timeout   3m;   # 请求体接收超时时间
    client_header_timeout 3m;   # 请求头接收超时时间
    send_timeout          3m;   # 响应发送超时时间

    # 3.6 后端代理缓冲配置（优化后端响应传输）
    fastcgi_buffers 8 128k;     # FastCGI 缓冲区（8 个 128K 缓冲区）
    fastcgi_buffer_size 128K;   # FastCGI 主缓冲区大小
    proxy_buffer_size 128k;     # 反向代理主缓冲区大小
    proxy_buffers 16 32k;       # 反向代理缓冲区（16 个 32K 缓冲区）
    proxy_busy_buffers_size 128k;  # 反向代理忙缓冲区大小

    # 3.7 引入业务配置（加载 conf.d 目录下所有 .conf 文件）
    include /etc/nginx/conf.d/*.conf;
}
EOF
```

### 3.3 业务路由配置文件（default.conf）

```bash
cat > /opt/nginx/conf.d/default.conf <<EOF
# 1. 后端服务集群定义（upstream 实现负载均衡）
upstream vhr-gateway {          # 网关服务集群（API 入口）
    server 10.172.131.164:9999; # 集群节点 1
    server 10.172.131.98:9999;  # 集群节点 2（默认轮询负载均衡）
}

upstream vhr-pc-base {          # PC 端基础服务集群
    server 10.172.131.214:80;
    server 10.172.131.58:80;
}

upstream vhr-fe-flow {          # 前端流程模块集群
    server 10.172.131.164:8081;
    server 10.172.131.98:8081;
}

upstream vhr-fe-perm {          # 前端权限模块集群
    server 10.172.131.164:8082;
    server 10.172.131.98:8082;
}

upstream vhr-fe-res {           # 前端资源模块集群
    server 10.172.131.164:8083;
    server 10.172.131.98:8083;
}

upstream vhr-fe-rule {          # 前端规则模块集群
    server 10.172.131.164:8084;
    server 10.172.131.98:8084;
}

upstream vhr-fe-sys {           # 前端系统模块集群
    server 10.172.131.164:8085;
    server 10.172.131.98:8085;
}

upstream hrzx-comp-sv-service { # 综合监控服务集群
    server 10.172.131.164:8086;
    server 10.172.131.98:8086;
}

# 2. 虚拟主机配置（监听 80 端口，处理业务请求）
server {
    listen       80;  # 监听端口（HTTP 协议默认端口）
    server_name localhost zhvhr.ciicsh.com;  # 绑定域名（多个域名用空格分隔）
    access_log /var/log/nginx/zhvhr.ciicsh.com.log access_json_log;  # 访问日志（JSON 格式）
    error_log /var/log/nginx/zhvhr.ciicsh.com.log error;  # 错误日志（error 级别）
    ssl_protocols   TLSv1.2;  # SSL 协议版本（HTTPS 时生效，预留配置）

    # 2.1 根路径路由（PC 端基础服务）
    location / {
        proxy_pass http://vhr-pc-base/;  # 转发至 vhr-pc-base 集群
        # 传递客户端真实信息至后端（避免后端获取到容器 IP）
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 2.2 前端流程模块路由（正则匹配：/mfe-flow/ 开头的请求）
    location ~ ^/mfe-flow/(.*)$ {
        proxy_pass http://vhr-fe-flow/$1;  # $1 保留请求路径后缀（如 /mfe-flow/a → /a）
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 2.3 前端权限模块路由
    location ~ ^/mfe-perm/(.*)$ {
        proxy_pass http://vhr-fe-perm/$1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 2.4 前端资源模块路由（~* 表示不区分大小写）
    location ~* ^/mfe-res/(.*)$ {
        proxy_pass http://vhr-fe-res/$1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 2.5 前端规则模块路由
    location ~ ^/mfe-rule/(.*)$ {
        proxy_pass http://vhr-fe-rule/$1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 2.6 前端系统模块路由
    location ~ ^/mfe-sys/(.*)$ {
        proxy_pass http://vhr-fe-sys/$1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 2.7 API 接口路由（后端网关入口）
    location ~ ^/api/(.*)$ {
        proxy_pass http://vhr-gateway/$1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 2.8 综合监控模块路由
    location ~ ^/mfe-supervise/(.*)$ {
        proxy_pass http://hrzx-comp-sv-service/$1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF
```

## 4. 导入 Nginx 镜像

若服务器无法直接拉取私有仓库镜像，需通过本地文件导入（假设镜像包在 `/tmp/nginx-files` 目录）：

```bash
# 进入镜像包所在目录
cd /tmp/nginx-files
# 导入镜像到 Docker 本地仓库
docker load < nginx_1.22.1-alpine-x86.tar.gz
# 验证导入结果（查看镜像是否存在）
docker images | grep nginx:1.22.1-alpine
```

## 5. 启动与管理 Nginx 容器

### 5.1 启动容器

```bash
# 进入 Docker Compose 配置文件所在目录
cd /opt/nginx
# 后台启动容器（-d 表示 detached 模式）
docker-compose -f docker-compose.yml up -d
```

### 5.2 查看容器状态

```bash
# 方式 1：通过 Docker Compose 查看
docker-compose ps -a
# 方式 2：通过 Docker 直接查看
docker ps | grep nginx  # 查看运行中的容器
docker inspect nginx | grep "Status"  # 查看容器详细状态（如 Up、Exited）
```

### 5.3 查看容器日志

```bash
# 实时查看日志（排查启动异常）
docker logs -f nginx
# 查看指定时间段日志（如最近 30 分钟）
docker logs --since 30m nginx
```

### 5.4 停止 / 重启容器

```bash
# 停止容器
docker-compose -f docker-compose.yml down
# 重启容器（配置修改后需执行）
docker-compose -f docker-compose.yml restart
```

## 6. 验证 Nginx 服务可用性

### 6.1 本地验证（服务器内部）

```bash
# 访问本地 Nginx（默认 80 端口）
curl http://localhost
# 若返回后端服务页面或 200 状态码，说明服务正常
```

### 6.2 远程验证（外部机器）

在外部机器浏览器或终端中访问：

```bash
# 替换为服务器 IP 或绑定的域名（如 zhvhr.ciicsh.com）
curl http://服务器IP
# 或访问具体模块路径（如流程模块）
curl http://服务器IP/mfe-flow/
```