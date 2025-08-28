# Redis 哨兵模式（Ubuntu 环境）安装部署

以下基于 **Ubuntu 20.04 LTS** 环境，梳理 Redis 哨兵模式的完整部署流程（含主从集群 + 哨兵集群）。Ubuntu 与 CentOS 的核心差异集中在「包管理工具（apt 替代 yum）」「防火墙配置（ufw 替代 firewalld）」，整体架构逻辑与主从 / 哨兵配置保持一致。

### 一、部署前准备

#### 1. 环境规划（3 台服务器，生产环境建议物理分离）

哨兵模式需 **1 主 2 从 Redis 节点**（存储数据）+ **3 个哨兵节点**（监控与故障转移），节点规划如下：

| 节点类型 | IP 地址      | Redis 实例端口 | 哨兵端口 | 角色说明               |
| -------- | ------------ | -------------- | -------- | ---------------------- |
| 主节点   | 192.168.1.10 | 6379           | -        | 写操作 + 数据同步源    |
| 从节点 1 | 192.168.1.11 | 6379           | -        | 读操作 + 主节点备份    |
| 从节点 2 | 192.168.1.12 | 6379           | -        | 读操作 + 主节点备份    |
| 哨兵 1   | 192.168.1.10 | -              | 26379    | 监控 + 投票 + 故障转移 |
| 哨兵 2   | 192.168.1.11 | -              | 26379    | 监控 + 投票 + 故障转移 |
| 哨兵 3   | 192.168.1.12 | -              | 26379    | 监控 + 投票 + 故障转移 |

#### 2. 基础环境配置（所有节点执行）

##### （1）更新系统与安装依赖

Ubuntu 用 `apt` 管理包，需先更新源并安装 Redis 编译依赖（GCC）：

```bash
# 更新系统软件源
apt update && apt upgrade -y

# 安装GCC（编译Redis源码）
apt install -y gcc make
```

##### （2）创建 /opt 核心目录结构

```bash
# 创建核心目录
mkdir -p /opt/{redis,redis-src}
mkdir -p /opt/redis/{master-6379,slave-6379,sentinel-26379}/{conf,data,logs}

# 说明：
# /opt/redis：Redis主目录
# /opt/redis-src：存储源码包
# 各节点目录包含conf（配置）、data（数据）、logs（日志）子目录
```

##### （3）下载 Redis 源码并解压

```bash
cd /opt/redis-src
# 下载稳定版（6.2.14）
wget https://download.redis.io/releases/redis-6.2.14.tar.gz
# 解压源码
tar -zxvf redis-6.2.14.tar.gz
```

##### （4）编译安装并创建软链接

```bash
# 进入源码目录编译
cd /opt/redis-src/redis-6.2.14
make && make install PREFIX=/opt/redis
```

**防火墙说明**：本次部署暂未开启防火墙，因此省略端口开放步骤。若后续启用防火墙（如 ufw），需执行以下命令开放端口：

```bash
ufw allow 6379/tcp   # Redis实例端口
ufw allow 26379/tcp  # 哨兵端口
ufw enable
```

### 二、部署 Redis 主从集群

### 1. 主节点配置（192.168.1.10）

#### （1）创建配置文件

```bash
# 复制基础配置文件
cp /opt/redis/src-link/redis.conf /opt/redis/master-6379/conf/redis.conf

# 编辑配置文件
vi /opt/redis/master-6379/conf/redis.conf
```

#### （2）核心配置内容

```bash
# 网络配置
bind 0.0.0.0          # 允许所有IP访问
port 6379             # 监听端口

# 运行模式
daemonize yes         # 后台运行
pidfile /opt/redis/master-6379/redis.pid  # PID文件路径

# 日志配置
loglevel notice       # 日志级别
logfile /opt/redis/master-6379/logs/redis.log  # 日志文件路径

# 安全配置
requirepass jL7jnAEL,!Wq  # 客户端连接密码
masterauth jL7jnAEL,!Wq   # 主从同步密码（与连接密码一致）

# 数据存储
dir /opt/redis/master-6379/data  # 数据文件存储目录
dbfilename dump.rdb              # RDB快照文件名
activerehashing yes              # 启用哈希表自动重哈希

# RDB持久化（按需调整）
save 900 1    # 900秒内有1次写入则触发快照
save 300 10   # 300秒内有10次写入则触发快照
save 60 10000 # 60秒内有10000次写入则触发快照
rdbcompression yes         # 启用RDB压缩

# AOF持久化（推荐开启）
appendonly yes              # 启用AOF模式
appendfilename "appendonly.aof"  # AOF文件名
appendfsync always          # 每次写入立即同步到磁盘（最安全）
# appendfsync everysec      # 每秒同步一次（平衡安全与性能）
# appendfsync no            # 由操作系统决定同步时机（性能最高）

# 主从复制（主节点无需配置此项）
# replicaof <主节点IP> 6379  # 主节点请保持注释状态
```

#### （3）创建 systemd 服务

```bash
vi /lib/systemd/system/redis6379.service

[Unit]
Description=redis-6379-server
After=network.target

[Service]
Type=forking
ExecStart=/opt/redis-6379/bin/redis-server /opt/redis-6379/conf/redis.conf
ExecStop=/opt/redis-6379/bin/redis-cli -a jL7jnAEL,!Wq shutdown
ExecReload=/bin/kill -s HUP $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

#### （4）起服务

```bash
systemctl daemon-reload
systemctl restart redis6379.service
systemctl enable redis6379.service
systemctl status redis6379.service
```



### 2. 从节点配置（192.168.1.11/12）

#### （1）创建配置文件

```bash
# 复制基础配置文件
cp /opt/redis/src-link/redis.conf /opt/redis/slave-6379/conf/redis.conf

# 编辑配置文件
vim /opt/redis/slave-6379/conf/redis.conf
# 网络配置（与主节点一致）
bind 0.0.0.0
port 6379

# 运行模式
daemonize yes
pidfile /opt/redis/slave-6379/redis.pid  # 从节点PID路径

# 日志配置
loglevel notice
logfile /opt/redis/slave-6379/logs/redis.log  # 从节点日志路径

# 安全配置（与主节点完全一致）
requirepass jL7jnAEL,!Wq
masterauth jL7jnAEL,!Wq

# 数据存储
dir /opt/redis/slave-6379/data  # 从节点数据目录
dbfilename dump.rdb
activerehashing yes

# RDB持久化（与主节点保持一致）
save 900 1
save 300 10
save 60 10000
rdbcompression yes

# AOF持久化（与主节点保持一致）
appendonly yes
appendfilename "appendonly.aof"
appendfsync always

# 主从复制（从节点核心配置）
replicaof 192.168.1.10 6379  # 指向主节点IP和端口
replica-read-only yes        # 从节点只读（必须开启）
```

#### （2）创建 systemd 服务

```bash
vi /lib/systemd/system/redis6379_slave.service

[Unit]
Description=redis-6379-server
After=network.target

[Service]
Type=forking
ExecStart=/opt/redis-6379/bin/redis-server /opt/redis-6379/conf/redis.conf
ExecStop=/opt/redis-6379/bin/redis-cli -a jL7jnAEL,!Wq shutdown
ExecReload=/bin/kill -s HUP $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

#### （3）启动节点(所有从节点)

```bash
systemctl daemon-reload
systemctl restart redis6379_slave.service
systemctl enable redis6379_slave.service
systemctl status redis6379_slave.service
```

#### （4）验证主从状态

```bash
# 连接主节点，查看从节点信息
/opt/redis/bin/redis-cli -h 192.168.1.10 -p 6379 -a jL7jnAEL,!Wq info replication

# 正常输出应包含：
# role:master
# connected_slaves:2
# 并显示两个从节点的IP和端口信息
```

## 三、部署哨兵集群（所有节点）

### 1. 创建哨兵配置文件

```bash
# 复制哨兵基础配置
cp /opt/redis/src-link/sentinel.conf /opt/redis/sentinel-26379/conf/sentinel.conf

# 编辑配置文件
vi /opt/redis/sentinel-26379/conf/sentinel.conf
```

### 2. 哨兵核心配置内容

```bash
# 网络配置
bind 0.0.0.0          # 允许所有IP访问
port 26379            # 哨兵端口
protected-mode no     # 关闭保护模式

# 运行模式
daemonize yes         # 后台运行
pidfile /opt/redis/sentinel-26379/sentinel.pid  # PID路径
logfile /opt/redis/sentinel-26379/logs/sentinel.log  # 日志路径

# 核心监控配置
sentinel monitor mymaster 192.168.1.10 6379 2  # 监控主节点，投票数2
sentinel auth-pass mymaster jL7jnAEL,!Wq        # 与主节点密码一致

# 故障检测配置
sentinel down-after-milliseconds mymaster 30000  # 30秒未响应视为下线
sentinel failover-timeout mymaster 180000        # 故障转移超时时间（3分钟）

# 同步配置
sentinel parallel-syncs mymaster 1  # 故障后同时同步新主节点的从节点数量
```

### 3. 启动哨兵并验证

#### （1）开机自启配置

```bash
vi /etc/systemd/system/redis-sentinel.service

[Unit]
Description=Redis Sentinel Service
After=network.target

[Service]
Type=forking
ExecStart=/opt/redis/bin/redis-sentinel /opt/redis/sentinel-26379/conf/sentinel.conf
ExecStop=/opt/redis/bin/redis-cli -h 127.0.0.1 -p 26379 shutdown
Restart=always
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

#### （2）起服务

```bash
systemctl daemon-reload
systemctl enable redis-sentinel
systemctl start redis-sentinel
```

#### （3）验证哨兵状态

```bash
# 连接任意哨兵节点
/opt/redis/bin/redis-cli -h 192.168.1.10 -p 26379

# 查看哨兵信息
192.168.1.10:26379> info sentinel

# 正常输出应包含：
# sentinel_masters:1
# sentinel_monitored_slaves:2
# sentinel_running_sentinels:3
```

