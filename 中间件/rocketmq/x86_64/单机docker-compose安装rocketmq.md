## 一、部署 Docker 27.3.1

### 1. 解压 Docker 安装包

假设 Docker 安装包 `docker-27.3.1-x86.tgz` 已上传至服务器 `/opt` 目录，执行以下命令解压并移动二进制文件：

```bash
# 解压安装包到 /opt 目录
tar -xvf /opt/docker-27.3.1-x86.tgz -C /opt/

# 移动 Docker 核心二进制文件到系统可执行路径
mv /opt/docker/* /usr/local/bin/

# 清理临时目录（可选）
rm -rf /opt/docker
```

### 2. 创建 Docker 配置文件（daemon.json）

通过 `cat` 命令生成 Docker 核心配置文件，定义存储驱动、日志策略等：

```bash
# 先创建 Docker 配置目录
mkdir -p /etc/docker

# 生成 daemon.json 配置
cat > /etc/docker/daemon.json <<EOF
{
    "storage-driver": "overlay2",          # 推荐存储驱动（性能优、兼容性好）
    "max-concurrent-downloads": 20,        # 最大并发镜像下载数（加速拉取）
    "live-restore": true,                  # Docker 重启时保持容器运行
    "max-concurrent-uploads": 10,          # 最大并发镜像上传数
    "debug": true,                         # 开启调试模式（便于问题排查）
    "log-opts": {
        "max-size": "100m",                # 单个容器日志最大容量
        "max-file": "5"                    # 单个容器日志保留文件数
    }
}
EOF
```

### 3. 创建 Docker Systemd 服务文件

生成 `docker.service` 让系统通过 Systemd 管理 Docker 服务：

```bash
cat > /etc/systemd/system/docker.service <<EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
# Docker 启动命令（二进制文件路径）
ExecStart=/usr/local/bin/dockerd
# 重载配置命令（发送 HUP 信号）
ExecReload=/bin/kill -s HUP $MAINPID

# 关闭资源限制（避免容器性能损耗）
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

TimeoutStartSec=0     # 启动无超时
Delegate=yes          # 允许 Docker 管理容器 cgroups
KillMode=process      # 仅杀死 Docker 主进程（不影响容器）
Restart=on-failure    # 失败时自动重启
StartLimitBurst=3     # 60秒内最多重启3次
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target  # 多用户模式下开机自启
EOF
```

### 4. 安装依赖并启动 Docker

```bash
# 安装 Docker 网络依赖（iptables）
apt-get update && apt-get install -y iptables --no-install-recommends

# 重载 Systemd 配置（识别新创建的 docker.service）
systemctl daemon-reload

# 设置 Docker 开机自启并立即启动
systemctl enable --now docker

# 验证 Docker 状态（确保显示 "active (running)"）
systemctl status docker --no-pager

# 验证 Docker 版本
docker --version
```

## 二、部署 Docker Compose 2.35.1

假设 Docker Compose 二进制文件 `docker-compose-linux_2.35.1-x86_64` 已上传至 `/opt` 目录，执行以下命令部署：

```bash
# 移动 Compose 二进制文件到系统路径
mv /opt/docker-compose-linux_2.35.1-x86_64 /usr/local/bin/docker-compose

# 赋予可执行权限
chmod +x /usr/local/bin/docker-compose

# 验证 Compose 版本
docker-compose --version
```

## 三、部署 RocketMQ 4.9.7（Namesrv + Broker）

### 1. 创建 RocketMQ 工作目录

```bash
mkdir -p /opt/rocketmq
cd /opt/rocketmq
```

### 2. 生成 Broker 配置文件（broker.conf）

通过 `cat` 生成 Broker 核心配置，绑定宿主机 IP 确保远程可访问：

```bash
cat > /opt/rocketmq/broker.conf <<EOF
# RocketMQ Broker 基础配置
brokerIP1=172.16.9.47  # 必须配置：绑定宿主机 IP（远程客户端连接用）
brokerName=broker-a     # Broker 名称（集群标识，主从需一致）
brokerId=0              # 0=Master 节点，>0=Slave 节点
deleteWhen=04           # 凌晨 4 点清理过期日志
fileReservedTime=48     # 日志文件保留时间（单位：小时）
brokerRole=ASYNC_MASTER # Broker 角色：ASYNC_MASTER=异步复制，SYNC_MASTER=同步复制
flushDiskType=ASYNC_FLUSH # 刷盘策略：异步刷盘（性能优先）
EOF
```

### 3. 生成 Docker Compose 配置（docker-compose.yml）

定义 Namesrv 和 Broker 服务，通过 Compose 一键启动：

```bash
cat > /opt/rocketmq/docker-compose.yml <<EOF
version: '3.8'  # 兼容 Docker Compose 2.x 版本

# 定义 RocketMQ 专属网络（隔离容器网络，避免冲突）
networks:
  rocketmq:
    driver: bridge

services:
  # RocketMQ Namesrv（注册中心：管理 Broker 节点，提供路由信息）
  namesrv:
    image: harbor.ciicsh.com/library/rocketmq:4.9.7  # 私有仓库镜像地址
    container_name: rmqnamesrv                       # 固定容器名，便于管理
    ports:
      - "9876:9876"  # 暴露 Namesrv 端口（客户端连接端口）
    networks:
      - rocketmq     # 加入专属网络
    command: sh mqnamesrv  # 启动 Namesrv 命令
    restart: always        # 容器异常自动重启
    deploy:
      resources:
        limits:
          cpus: '1'        # CPU 限制（根据服务器配置调整）
          memory: 1G       # 内存限制（避免内存溢出）

  # RocketMQ Broker（消息存储节点：接收、存储、转发消息）
  broker:
    image: harbor.ciicsh.com/library/rocketmq:4.9.7
    container_name: rmqbroker
    ports:
      - "10909:10909"  # 客户端 TCP 通信端口
      - "10911:10911"  # Broker 主从同步端口
      - "10912:10912"  # 控制台/旧客户端访问端口
    environment:
      - NAMESRV_ADDR=rmqnamesrv:9876  # 指向 Namesrv 服务（容器内直接通过容器名访问）
      - JAVA_OPTS=-Xms512m -Xmx512m   # JVM 内存配置（根据需求调整）
    volumes:
      # 挂载宿主机配置文件到容器（覆盖容器默认配置）
      - ./broker.conf:/home/rocketmq/rocketmq-4.9.7/conf/broker.conf
      # 可选：挂载日志和数据目录（持久化数据，需提前创建目录并授权）
      # - ./logs:/home/rocketmq/logs
      # - ./data:/home/rocketmq/store
    depends_on:
      - namesrv  # 依赖 Namesrv 服务，确保 Namesrv 先启动
    networks:
      - rocketmq
    command: sh mqbroker -c /home/rocketmq/rocketmq-4.9.7/conf/broker.conf
    restart: always
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
EOF
```

### 4. 启动 RocketMQ 服务

```bash
# 进入工作目录
cd /opt/rocketmq

# 后台启动 Namesrv 和 Broker（-d 表示后台运行）
docker-compose up -d

# 查看容器状态（确保 STATUS 为 Up）
docker-compose ps

# 查看 Broker 日志（排查启动异常，按 Ctrl+C 退出）
docker logs -f rmqbroker
```

## 四、部署 RocketMQ Dashboard（可视化控制台）

### 1. 生成 Dashboard 启动命令

```bash
# 生成启动脚本
cat > /opt/rocketmq/rocketmq_dashboard.sh <<EOF
#!/bin/bash
# 停止并删除旧容器（避免端口冲突）
if docker ps -a | grep -q "rocketmq-dashboard"; then
    docker stop rocketmq-dashboard
    docker rm rocketmq-dashboard
fi

# 启动 Dashboard 容器
docker run -d \
  --name rocketmq-dashboard \
  # 指定 Namesrv 地址 + JVM 内存配置
  -e "JAVA_OPTS=-Drocketmq.namesrv.addr=172.16.9.47:9876 -Xms256m -Xmx256m" \
  -p 8082:8080 \  # 宿主机 8082 端口映射容器 8080 端口（避免冲突）
  --network=rocketmq_rocketmq \  # 加入 RocketMQ 专属网络（与 Broker/Namesrv 互通）
  --restart always \  # 容器异常自动重启
  harbor.ciicsh.com/library/rocketmq-dashboard:latest
EOF
```

### 2. 启动 Dashboard

```bash
# 赋予脚本可执行权限
chmod +x /opt/rocketmq/rocketmq_dashboard.sh

# 执行脚本启动 Dashboard
/opt/rocketmq/rocketmq_dashboard.sh

# 验证 Dashboard 容器状态（确保 STATUS 为 Up）
docker ps | grep rocketmq-dashboard
```

## 五、验证部署结果

1. **访问 Dashboard**：
   浏览器打开 `http://172.16.9.47:8082`，若能看到控制台页面且显示 `Namesrv Address: 172.16.9.47:9876`，说明连接成功。
2. **检查 Broker 状态**：
   在 Dashboard 左侧菜单栏点击「Broker 集群」，若能看到 `broker-a` 且状态为「在线」，说明 RocketMQ 服务正常。

## 六、关键注意事项

1. **私有仓库登录**：
   若拉取 `harbor.ciicsh.com` 镜像失败，需先登录私有仓库：

   ```bash
   docker login harbor.ciicsh.com
   # 输入仓库用户名和密码
   ```

2. **端口冲突处理**：
   若 9876/10909/10911/8082 端口已被占用，需修改对应配置文件中的端口映射（如 `8083:8080`）。

3. **数据持久化（可选）**：
   若需持久化 RocketMQ 日志和数据，需：

   ```bash
   # 创建日志和数据目录
   mkdir -p /opt/rocketmq/logs /opt/rocketmq/data
   # 赋予容器内用户权限（默认容器内用户 UID 为 1000）
   chown -R 1000:1000 /opt/rocketmq/logs /opt/rocketmq/data
   ```

   再取消 `docker-compose.yml` 中 `volumes` 下日志 / 数据目录的注释，重启服务：`docker-compose down && docker-compose up -d`。

