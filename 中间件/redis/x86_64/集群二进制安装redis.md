#### 环境与基础配置

##### 下载地址

| 服务  | 地址                                | 备注      |
| ----- | ----------------------------------- | --------- |
| redis | https://download.redis.io/releases/ | 版本7.4.3 |

##### 集群节点规划

| 服务器 IP   | Redis 实例端口 | 初始角色（集群创建后） |
| ----------- | -------------- | ---------------------- |
| 172.16.8.94 | 6379           | 主节点（Master）       |
| 172.16.8.94 | 6380           | 从节点（Slave）        |
| 172.16.8.95 | 6379           | 主节点（Master）       |
| 172.16.8.95 | 6380           | 从节点（Slave）        |
| 172.16.8.96 | 6379           | 主节点（Master）       |
| 172.16.8.96 | 6380           | 从节点（Slave）        |

#### **一、安装部署 Redis**（所有节点）

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

#### **二、配置 systemd 服务管理**（所有节点）

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

#### **三、节点配置文件修改**（所有节点）

#####  1.核心配置项（关键部分）

```bash
# 网络配置
bind 0.0.0.0          # 允许所有IP访问（生产环境建议限定内网IP）
port 6379             # Redis实例端口（6380实例需修改此值）
daemonize yes         # 后台运行Redis
pidfile /opt/redis/redis.pid  # PID文件路径
loglevel notice       # 日志级别（notice：常规信息，warning：警告，debug：调试）
logfile /opt/redis/redis.log  # 日志文件路径

# 安全配置
requirepass J5rgv4gkoq+x  # 客户端连接密码（所有节点需保持一致）
masterauth J5rgv4gkoq+x   # 从节点连接主节点的密码（与requirepass一致）

# 数据持久化配置
dir /opt/redis              # 数据文件（RDB/AOF）存储目录
dbfilename dump.rdb         # RDB文件名
rdbcompression yes          # 开启RDB压缩
activerehashing yes         # 开启哈希表自动重哈希

# RDB持久化策略（满足任一条件即触发快照）
save 900 1    # 900秒内有1次写操作
save 300 10   # 300秒内有10次写操作
save 60 10000 # 60秒内有10000次写操作

# AOF持久化配置（建议开启，提升数据安全性）
appendonly yes               # 开启AOF持久化
appendfilename "appendonly.aof"  # AOF文件名
appendfsync everysec         # AOF刷盘策略：每秒刷盘（平衡性能与安全性）

# 集群核心配置（所有节点需开启）
cluster-enabled yes                # 启用Redis集群模式
cluster-config-file nodes.conf     # 集群节点信息文件（自动生成/更新）
cluster-node-timeout 15000         # 节点超时时间（15秒，超时则判定节点下线）
```

#### **四、启动并设置开机自启**（所有节点）

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

#### 五、Redis 集群初始化（所有节点）

通过 `redis-cli --cluster` 命令快速创建 3 主 3 从的 Redis 集群，`--cluster-replicas 1` 表示为每个主节点分配 1 个从节点。

```bash
./redis-cli --cluster create \
172.16.8.94:6379 172.16.8.95:6379 172.16.8.96:6379 \  # 3个主节点
172.16.8.94:6380 172.16.8.95:6380 172.16.8.96:6380 \  # 3个从节点
--cluster-replicas 1 \  # 主从比例 1:1
-a J5rgv4gkoq+x         # 集群密码（与redis.conf中requirepass一致）
```

#### 六、集群验证

```bash
./redis-cli -a 'J5rgv4gkoq+x' -p 6379 cluster nodes
```



#### 七、从节点操作

从节点（Slave）不负责哈希槽存储，主要用于数据备份和故障转移。管理流程包括「从集群中移除从节点」和「重新将从节点加入集群并指定主节点」两个场景

##### 1. 删除从节点（以 172.16.8.94:6380 为例）

###### 步骤 1：查询待删除从节点的 ID

通过任意集群节点执行`cluster nodes`命令，获取目标从节点的唯一标识（节点 ID）：

```bash
# 连接集群中任意节点（如172.16.8.94:6379）查询节点信息
./redis-cli -a 'J5rgv4gkoq+x' -p 6379 cluster nodes
```

###### **查询结果（关键信息提取）**：

```bash
# 待删除的从节点信息
b19126dff65ad2ad818435a2e5ace69ed250aad3 172.16.8.94:6380@16380 slave 38bad003b32aa137ecf94d0cb21ca565ddff3e8a 0 1740722693000 4 connected
```

- 节点 ID：`b19126dff65ad2ad818435a2e5ace69ed250aad3`（后续删除操作需使用）
- 节点地址：172.16.8.94:6380
- 角色：slave（从节点）

###### 步骤 2：执行删除操作（集群遗忘节点）

通过`CLUSTER FORGET`命令让集群移除目标从节点（需指定节点 ID）：

```bash
# 连接任意集群节点，发送遗忘命令
./redis-cli -a 'J5rgv4gkoq+x' -p 6379 \
CLUSTER FORGET b19126dff65ad2ad818435a2e5ace69ed250aad3
```

**命令说明**：

- `CLUSTER FORGET`：通知集群移除指定 ID 的节点
- 只需在一个节点执行该命令，集群会自动同步移除信息到所有节点

###### 步骤 3：验证删除结果

再次查询节点列表，确认目标从节点已被移除：

```bash
./redis-cli -a 'J5rgv4gkoq+x' -p 6379 cluster nodes
```

##### 2. 重新加入从节点（以 172.16.8.94:6380 为例）

###### 步骤 1：将从节点接入集群

通过从节点自身发送`CLUSTER MEET`命令，与集群中任意节点建立连接：

```bash
# 连接待加入的从节点（172.16.8.94:6380），并加入集群
./redis-cli -h 172.16.8.94 -p 6380 -a 'J5rgv4gkoq+x' \
CLUSTER MEET 172.16.8.95 6379  # 与集群中任意节点（如172.16.8.95:6379）建立连接
```

**命令说明**：

- `CLUSTER MEET`：让当前节点（172.16.8.94:6380）加入指定节点所在的集群
- 执行后，172.16.8.94:6380 会自动同步集群信息（如其他节点地址、哈希槽分布等）

###### **步骤 2：指定从节点的主节点**

通过`CLUSTER REPLICATE`命令，将重新加入的从节点绑定到目标主节点（需指定主节点 ID）：

```bash
# 连接从节点，指定其主节点
./redis-cli -h 172.16.8.94 -p 6380 -a 'J5rgv4gkoq+x' \
CLUSTER REPLICATE 38bad003b32aa137ecf94d0cb21ca565ddff3e8a
```

**参数说明**：

- 主节点 ID：`38bad003b32aa137ecf94d0cb21ca565ddff3e8a`（对应主节点 172.16.8.96:6379）
- 绑定后，从节点会自动从主节点同步数据

###### 步骤 3：验证绑定结果

查询集群节点列表，确认从节点的角色和关联主节点正确：

```bash
./redis-cli -a 'J5rgv4gkoq+x' -p 6379 cluster nodes
```

**预期结果（关键信息）**：

```bash
b19126dff65ad2ad818435a2e5ace69ed250aad3 172.16.8.94:6380@16380 slave 38bad003b32aa137ecf94d0cb21ca565ddff3e8a 0 1740726113000 7 connected
```

- 角色为`slave`
- 关联主节点 ID 为`38bad003b32aa137ecf94d0cb21ca565ddff3e8a`（正确绑定目标主节点）



#### 八、主节点管理（删除与重新加入）

主节点（Master）是 Redis 集群的核心数据存储节点，负责 **哈希槽分配** 与 **写操作处理**。由于主节点关联数据分片，管理流程需重点关注「哈希槽迁移」（避免数据丢失），整体分为「安全删除主节点」和「重新加入主节点并恢复数据分片」两大场景。以下以 **172.16.8.95:6379**（初始节点 ID：`878cca4742f0d0d6154df8a98fedd60dfbd7ab08`）为例，详细梳理操作步骤

##### 1. 删除主节点

主节点删除前必须完成 **哈希槽迁移**（将其负责的所有哈希槽转移到其他主节点），否则会导致对应槽位的数据无法访问。整体流程：`确认哈希槽范围 → 迁移哈希槽 → 从集群中遗忘节点 → 验证删除结果`。

###### 步骤 1：确认待删除主节点的关键信息

通过任意集群节点执行 `CLUSTER NODES` 命令，获取主节点的 **节点 ID** 和 **哈希槽范围**：

```bash
# 连接集群任意节点（如 172.16.8.94:6379）查询节点信息
./redis-cli -a 'J5rgv4gkoq+x' -p 6379 cluster nodes
```

**关键信息提取（待删除主节点）**

```bash
878cca4742f0d0d6154df8a98fedd60dfbd7ab08	172.16.8.95:6379@16379	master	5461-10922	172.16.8.96:6380（节点 ID：9354c600...）
```

- 核心数据：该主节点负责 **5461-10922 共 5462 个哈希槽**，需平均迁移到剩余 2 个主节点（各迁移 2731 个槽，确保总槽数 16384 不变）。

##### 步骤 2：迁移主节点的哈希槽

使用 `redis-cli --cluster reshard` 命令执行哈希槽迁移，`--cluster-yes` 用于自动确认迁移操作（避免交互确认），需分两次迁移（每次 2731 个槽）。

###### 子步骤 2.1：迁移 2731 个槽到主节点 172.16.8.94:6379

目标主节点信息：

- 地址：172.16.8.94:6379
- 节点 ID：`fb8a9680043623d4b188e83785db8961facb79f0`
- 原哈希槽范围：0-5460（迁移后新增 8192-10922）

执行迁移命令：

```bash
./redis-cli --cluster reshard 172.16.8.94:6379 \
--cluster-from 878cca4742f0d0d6154df8a98fedd60dfbd7ab08 \  # 源主节点 ID（待删除）
--cluster-to fb8a9680043623d4b188e83785db8961facb79f0 \    # 目标主节点 ID
--cluster-slots 2731 \  # 单次迁移槽数量（5462/2=2731）
--cluster-yes \         # 自动确认迁移（无需手动输入 yes）
-a 'J5rgv4gkoq+x'       # 集群密码（与 redis.conf 一致）
```

###### 子步骤 2.2：迁移剩余 2731 个槽到主节点 172.16.8.96:6379

目标主节点信息：

- 地址：172.16.8.96:6379
- 节点 ID：`38bad003b32aa137ecf94d0cb21ca565ddff3e8a`
- 原哈希槽范围：10923-16383（迁移后新增 5461-8191）

执行迁移命令：

```bash
./redis-cli --cluster reshard 172.16.8.94:6379 \
--cluster-from 878cca4742f0d0d6154df8a98fedd60dfbd7ab08 \  # 源主节点 ID（待删除）
--cluster-to 38bad003b32aa137ecf94d0cb21ca565ddff3e8a \    # 目标主节点 ID
--cluster-slots 2731 \  # 剩余槽数量
--cluster-yes \         # 自动确认迁移
-a 'J5rgv4gkoq+x'       # 集群密码
```

**迁移过程说明**：

- 迁移时集群仍可正常提供读写服务，但会对迁移的槽位加短暂锁（毫秒级），建议在业务低峰期操作；
- 命令执行后，Redis 会自动完成「槽位元数据更新」「数据迁移」「从节点跟随调整」（原关联从节点会自动绑定到新主节点）。

##### 步骤 3：从集群中遗忘已迁移完成的主节点

哈希槽迁移完成后，执行 `CLUSTER FORGET` 命令让集群移除该主节点（需通过任意集群节点发起）：

```bash
./redis-cli -c -h 172.16.8.94 -p 6379 -a 'J5rgv4gkoq+x' \
CLUSTER FORGET 878cca4742f0d0d6154df8a98fedd60dfbd7ab08
```

##### 步骤 4：验证主节点删除结果

执行 `CLUSTER NODES` 命令，确认以下信息：

```bash
./redis-cli -a 'J5rgv4gkoq+x' -p 6379 cluster nodes
```

#### 2. 重新加入主节点（172.16.8.95:6379）

若需将已删除的主节点重新加入集群，需先让节点接入集群，再从其他主节点迁移哈希槽回来，恢复其主节点角色。整体流程：`清理旧节点配置 → 加入集群 → 迁移哈希槽 → 验证节点状态`。

##### 步骤 1：清理待加入主节点的旧配置（关键！）

主节点删除后，其数据目录下的 `nodes.conf` 文件会保留旧节点 ID，若直接重启会导致集群中出现「节点 ID 冲突」，需先清理配置：

```bash
# 1. 停止待加入的 Redis 节点（172.16.8.95:6379）
kill $(cat /opt/redis/redis-6379.pid)  # PID 文件路径与 redis.conf 中 pidfile 一致
# 2. 删除旧节点配置文件（nodes.conf）和临时数据（避免残留哈希槽信息）
rm -f /opt/redis/nodes.conf  # 路径与 redis.conf 中 cluster-config-file 一致
# 3. 重启 Redis 节点（自动生成新节点 ID）
./redis-server /opt/redis/redis.conf  # 对应节点的配置文件路径
```

重启后，通过 `CLUSTER NODES` 在该节点本地查询新节点 ID（后续迁移需使用）：

```bash
./redis-cli -h 172.16.8.95 -p 6379 -a 'J5rgv4gkoq+x' cluster nodes
```

新节点 ID 示例：`878cca4742f0d0d6154df8a98fedd60dfbd7ab08`（与原 ID 可能不同，以实际为准）。

##### 步骤 2：将新主节点加入集群

通过待加入节点自身发送 `CLUSTER MEET` 命令，与集群中任意节点建立连接（如 172.16.8.94:6379）：

```bash
./redis-cli -h 172.16.8.95 -p 6379 -a 'J5rgv4gkoq+x' \
CLUSTER MEET 172.16.8.94 6379
```

- 执行后，新节点会自动同步集群信息（如其他节点地址、现有哈希槽分布），但此时角色为「无哈希槽的主节点」（需后续迁移槽位）。

##### 步骤 3：迁移哈希槽到新主节点

从现有 2 个主节点中各迁移 2731 个槽到新主节点，恢复其「5461-10922 哈希槽」的职责。

###### 子步骤 3.1：从 172.16.8.94:6379 迁移 2731 个槽

源主节点信息：

- 地址：172.16.8.94:6379
- 节点 ID：`fb8a9680043623d4b188e83785db8961facb79f0`
- 待迁移槽范围：8192-10922（共 2731 个槽）

执行迁移命令：

```bash
./redis-cli --cluster reshard 172.16.8.94:6379 \
--cluster-from fb8a9680043623d4b188e83785db8961facb79f0 \  # 源主节点 ID
--cluster-to 878cca4742f0d0d6154df8a98fedd60dfbd7ab08 \    # 新主节点 ID（步骤 1 中获取）
--cluster-slots 2731 \  # 迁移槽数量
--cluster-yes \         # 自动确认
-a 'J5rgv4gkoq+x'       # 集群密码
```

###### 子步骤 3.2：从 172.16.8.96:6379 迁移 2731 个槽

源主节点信息：

- 地址：172.16.8.96:6379
- 节点 ID：`38bad003b32aa137ecf94d0cb21ca565ddff3e8a`
- 待迁移槽范围：5461-8191（共 2731 个槽）

执行迁移命令：

```bash
./redis-cli --cluster reshard 172.16.8.94:6379 \
--cluster-from 38bad003b32aa137ecf94d0cb21ca565ddff3e8a \  # 源主节点 ID
--cluster-to 878cca4742f0d0d6154df8a98fedd60dfbd7ab08 \    # 新主节点 ID
--cluster-slots 2731 \  # 迁移槽数量
--cluster-yes \         # 自动确认
-a 'J5rgv4gkoq+x'       # 集群密码
```

##### 步骤 4：验证主节点重新加入结果

执行 `CLUSTER NODES` 和 `CLUSTER SLOTS` 命令，确认以下信息：

```bash
# 1. 查看节点角色与哈希槽分配
./redis-cli -a 'J5rgv4gkoq+x' -p 6379 cluster nodes
# 2. 查看哈希槽分布详情（更直观）
./redis-cli -a 'J5rgv4gkoq+x' -p 6379 cluster slots
```

##### 主节点管理核心注意事项

1. **哈希槽迁移优先级**：删除主节点前必须迁移所有槽位，否则会导致数据丢失；迁移时需确保「源节点槽范围明确」「目标节点可用」，避免中途中断。
2. **节点 ID 唯一性**：重新加入主节点前，务必删除旧的 `nodes.conf` 文件，否则会因 ID 冲突导致加入失败；若需保留原 ID，需提前备份 `nodes.conf` 并在重启时复用。
3. **业务影响控制**：哈希槽迁移会占用主节点 CPU/IO 资源，建议在业务低峰期执行；单次迁移槽数量不宜过大（建议 ≤ 1000 个），避免集群延迟升高。
4. **故障转移联动**：若删除主节点时未提前迁移槽位，集群会触发自动故障转移（其从节点升级为新主节点），但可能导致 15 秒左右的写操作延迟，需根据业务场景选择「手动迁移」或「自动故障转移」。

