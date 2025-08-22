| 服务  | 地址                                | 备注 |
| ----- | ----------------------------------- | ---- |
| redis | https://download.redis.io/releases/ | 版本 |

#### **一、安装部署 Redis**

##### 1. 解压安装包并创建软链接

```bash
# 解压安装包（版本号根据实际下载文件调整）
tar -xf redis-7.4.3.tar.gz

# 创建软链接（简化目录访问）
ln -s redis-7.4.3 redis
```

##### 2. 编译安装

```bash
# 进入 Redis 目录
cd redis

# 编译并指定安装路径（/opt/redis）
make && make PREFIX=/opt/redis install
```

#### **二、配置 systemd 服务管理**

##### 1. 创建服务配置文件

```bash
vim /etc/systemd/system/redis.service

[Unit]
Description=redis-server
After=network.target

[Service]
Type=forking
ExecStart=/opt/redis/bin/redis-server /opt/redis/redis.conf
ExecStop=/opt/redis/bin/redis-cli shutdown
ExecReload=/bin/kill -s HUP $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

#### **三、节点配置文件修改**

#####  1.核心配置项（关键部分）

```bash
vim /opt/redis/redis.conf

# 网络配置
bind 0.0.0.0          # 允许所有IP访问（生产环境建议指定具体IP）
port 6379             # 监听端口

# 运行模式
daemonize yes         # 后台运行
pidfile /opt/redis/redis.pid  # PID文件路径

# 日志配置
loglevel notice       # 日志级别（notice/warn/debug）
logfile /opt/redis/redis.log  # 日志文件路径

# 安全配置
requirepass jL7jnAEL,!Wq  # 客户端连接密码
masterauth jL7jnAEL,!Wq   # 主从同步密码（与requirepass保持一致）

# 数据存储
dir /opt/redis             # 数据文件存储目录
dbfilename dump.rdb        # RDB快照文件名
activerehashing yes        # 启用哈希表自动重哈希

# RDB持久化（按需调整）
save 900 1    # 900秒内有1次写入则触发快照
save 300 10   # 300秒内有10次写入则触发快照
save 60 10000 # 60秒内有10000次写入则触发快照
rdbcompression yes         # 启用RDB压缩

# AOF持久化（推荐开启）
appendonly yes              # 启用AOF模式
appendfilename "appendonly.aof"  # AOF文件名
appendfsync always          # 每次写入立即同步到磁盘（最安全，性能略低）
# appendfsync everysec      # 每秒同步一次（平衡安全与性能）
# appendfsync no            # 由操作系统决定同步时机（性能最高，安全性低）

# 主从复制（主节点无需配置，从节点需开启）
# replicaof <主节点IP> 6379  # 主节点地址（主节点注释此行，从节点取消注释）
```

#### **四、启动并设置开机自启**

```bash
# 重新加载systemd配置
systemctl daemon-reload

# 启动Redis
systemctl start redis

# 查看状态
systemctl status redis

# 设置开机自启
systemctl enable redis
```

