## 1. 前置准备：文件与环境确认

### 1.1 需上传的文件清单

将以下文件上传至服务器（建议统一存放于临时目录，如 `/tmp/kibana-files`，后续迁移至正式路径）：

- `docker-compose.yml`：Kibana 容器编排配置文件
- `kibana.tar.gz`：Kibana 镜像压缩包（本地导入用）
- `kibana.yml`：Kibana 核心配置文件（关联 Elasticsearch 集群）
- 依赖证书文件：`elastic-stack-ca.p12`（从 Elasticsearch 集群节点复制，路径参考 `/opt/elasticsearch/elastic-stack-ca.p12`）

### 1.2 环境依赖要求

- 已部署 Elasticsearch 集群（版本需与 Kibana 匹配，此处为 8.17.6），且集群节点（10.172.131.206、10.172.131.68、10.172.131.201）可正常访问
- 服务器已安装 Docker 与 Docker Compose（建议版本：Docker 20.10+、Docker Compose 2.10+）
- 开放 5601 端口（Kibana 默认访问端口），确保外部可访问

## 2. 初始化工作目录与权限

创建 Kibana 正式工作目录，用于存储配置、数据及证书，并配置正确权限（避免容器内用户无读写权限）：

```bash
# 1. 创建主目录及数据子目录
mkdir -p /opt/kibana/data
# 2. 配置目录权限（Kibana 容器内默认用户 UID/GID 为 1000，需与宿主机目录权限匹配）
chown -R 1000:1000 /opt/kibana/
# 3. 复制 Elasticsearch 证书至 Kibana 配置目录（确保证书权限一致）
cp /opt/elasticsearch/elastic-stack-ca.p12 /opt/kibana/
chown 1000:1000 /opt/kibana/elastic-stack-ca.p12
```

## 3. 生成核心配置文件

### 3.1 容器编排文件（docker-compose.yml）

定义 Kibana 容器的运行参数（镜像、挂载目录、网络、重启策略等）：

```bash
cat > /opt/kibana/docker-compose.yml <<EOF
version: '3.8'  # Docker Compose 版本（适配主流 Docker 环境）

services:
  kibana:
    container_name: kibana  # 容器固定名称（便于管理与排查）
    image: "harbor.ciicsh.com/library/kibana:8.17.6"  # 私有仓库镜像（版本与 ES 集群一致）
    environment:
      TZ: 'Asia/Shanghai'  # 配置容器时区（与宿主机一致，避免日志时间偏差）
    volumes:  # 宿主机与容器目录挂载（实现配置持久化与数据共享）
      - /opt/kibana/kibana.yml:/usr/share/kibana/config/kibana.yml  # 核心配置文件挂载
      - /opt/kibana/data:/usr/share/kibana/data  # Kibana 数据目录挂载（缓存、索引等）
      - /opt/kibana/elastic-stack-ca.p12:/usr/share/kibana/config/elastic-stack-ca.p12  # CA 证书挂载
#      - /opt/kibana/elastic-certificates.p12:/usr/share/kibana/config/elastic-certificates.p12  # 可选：节点证书（按需启用）
    ports:
      - 5601:5601  # 端口映射（宿主机端口:容器内端口，Kibana 默认端口 5601）
    restart: always  # 重启策略：容器异常退出时自动重启（高可用基础）
    network_mode: "host"  # 使用主机网络（减少网络转发开销，适合生产环境）
EOF
```

### 3.2 Kibana 核心配置文件（kibana.yml）

配置 Kibana 访问端口、关联 Elasticsearch 集群、认证信息及本地化设置：

```bash
cat > /opt/kibana/kibana.yml <<EOF
# 1. 服务基础配置
server.port: 5601  # Kibana 监听端口（需与 docker-compose 端口映射一致）
server.host: "0.0.0.0"  # 允许所有地址访问（生产环境可改为指定 IP，如服务器内网 IP）

# 2. Elasticsearch 集群关联配置
elasticsearch.hosts: ["https://10.172.131.206:9200","https://10.172.131.68:9200","https://10.172.131.201:9200"]  # ES 集群节点列表（HTTPS 协议，需与 ES 配置一致）
elasticsearch.requestTimeout: 90000  # 请求超时时间（90 秒，适配大查询场景）

# 3. 认证配置（对接 ES 内置用户）
elasticsearch.username: "kibana_system"  # Kibana 内置服务用户（ES 默认存在，用于 Kibana 与 ES 通信）
elasticsearch.password: "VHRkibana@admin"  # 用户密码（需与 ES 中配置的 kibana_system 密码一致）

# 4. SSL 证书配置（按需调整，当前为简化配置）
elasticsearch.ssl.verificationMode: none  # 禁用 SSL 证书校验（测试/内部环境用，生产环境建议改为 certificate 并启用 CA 证书）
#elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/config/elastic-stack-ca.p12"]  # 可选：启用 CA 证书校验（生产环境推荐）
#elasticsearch.ssl.verificationMode: certificate  # 可选：仅校验证书有效性（需启用上一行 CA 证书配置）

# 5. 本地化配置
i18n.locale: "zh-CN"  # 界面语言设置为中文（可选值：en、zh-CN 等）
EOF
```

## 4. 导入 Kibana 镜像

若服务器无法直接拉取私有仓库镜像，需通过本地压缩包导入：

```bash
# 进入镜像包所在目录（假设镜像包在 /tmp/kibana-files）
cd /tmp/kibana-files
# 导入镜像至 Docker 本地仓库
docker load < kibana.tar.gz
# 验证导入结果（查看镜像是否存在）
docker images | grep kibana:8.17.6
```

## 5. 启动与管理 Kibana 服务

### 5.1 启动 Kibana 容器

进入 `docker-compose.yml` 所在目录，执行启动命令：

```bash
cd /opt/kibana
# 后台启动容器（-d 表示 detached 模式，不占用终端）
docker-compose -f docker-compose.yml up -d
```

### 5.2 查看容器运行状态

```bash
# 方式 1：通过 Docker Compose 查看（显示容器状态、端口等信息）
docker-compose ps -a
# 方式 2：通过 Docker 直接查看（显示容器运行状态、ID 等）
docker ps | grep kibana
# 方式 3：查看容器详细日志（排查启动异常，如 ES 连接失败、权限问题）
docker logs -f kibana
```

### 5.3 停止 / 重启 Kibana 容器

```bash
# 停止容器
docker-compose -f docker-compose.yml down
# 重启容器（配置修改后需执行，如更新 kibana.yml 后）
docker-compose -f docker-compose.yml restart
```

## 6. 验证 Kibana 服务可用性

### 6.1 本地验证（服务器内部）

通过 `curl` 命令检查 Kibana 服务是否正常响应：

```bash
curl -I http://localhost:5601
# 若返回 "HTTP/1.1 200 OK"，说明服务正常启动
```

### 6.2 远程访问验证（外部浏览器）

在外部机器打开浏览器，访问 Kibana 地址：

```plaintext
http://Kibana服务器IP:5601
```

- 首次访问需使用 `kibana_system` 账号密码登录（即 `kibana.yml` 中配置的 `elasticsearch.username` 和 `elasticsearch.password`）
- 登录成功后，若能看到 Kibana 中文界面且「Elasticsearch 集群状态」显示为绿色，说明部署成功

## 7. 注意事项

1. **版本兼容性**：Kibana 版本必须与 Elasticsearch 集群版本完全一致（如均为 8.17.6），否则会出现通信异常

2. **证书配置**：生产环境不建议禁用 SSL 校验（`elasticsearch.ssl.verificationMode: none`），需启用 CA 证书（`elastic-stack-ca.p12`）并将 `verificationMode` 改为 `certificate` 或 `full`

3. **密码一致性**：`kibana.yml` 中 `elasticsearch.password` 必须与 Elasticsearch 集群中 `kibana_system` 用户的密码一致，可通过 ES 命令修改密码：

   ```bash
   # 在 ES 集群节点执行，修改 kibana_system 用户密码
   elasticsearch-reset-password -u kibana_system -i
   ```

   