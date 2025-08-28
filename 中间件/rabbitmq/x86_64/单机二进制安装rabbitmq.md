## 一、基础信息

### 1. 安装包下载地址

| 软件     | 版本      | 官方下载地址                                                 |
| -------- | --------- | ------------------------------------------------------------ |
| RabbitMQ | 3.8.35    | https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.8.35 |
| Erlang   | 24.3.4.17 | https://github.com/erlang/otp/releases/tag/OTP-24.3.4.17     |

### 2. 版本适配说明

RabbitMQ 对 Erlang 版本有严格依赖，需确保版本匹配，避免启动失败：

| 服务     | 推荐版本 | 兼容 Erlang 版本 |
| -------- | -------- | ---------------- |
| RabbitMQ | 3.8.35   | 23.2 ~ 24.x      |

## 二、安装 Erlang（24.3.4.17）

RabbitMQ 基于 Erlang 开发，需先安装 Erlang 运行环境。

### 1. 安装系统依赖

执行以下命令安装 Erlang 编译所需的依赖包：

```bash
yum install -y \
  ncurses-devel \
  libsctp-devel \
  lksctp-tools \
  make \
  gcc \
  gcc-c++ \
  kernel-devel \
  m4 \
  openssl-devel \
  unixODBC-devel \
  wxBase3 \
  wxGTK3 \
  wxGTK3-devel \
  mesa-libGL-devel \
  mesa-libGLU-devel \
  fop.noarch
```

### 2. 解压 Erlang 安装包

假设安装包已下载至 `/opt` 目录，执行解压命令：

```bash
# 进入安装包目录
cd /opt

# 解压安装包（文件名需与实际下载的一致）
tar -xf otp-OTP-24.3.4.17.tar.gz

# 创建 Erlang 安装目录（统一路径，便于管理）
mkdir -p /usr/local/erlang
```

### 3. 配置与编译安装

进入解压后的 Erlang 源码目录，执行配置、编译和安装命令：

```bash
# 进入 Erlang 源码目录
cd /opt/otp-OTP-24.3.4.17

# 配置安装参数（指定安装路径，启用多线程支持）
./configure \
  --prefix=/usr/local/erlang \  # 安装路径
  --enable-smp-support \        # 启用对称多处理（提升性能）
  --enable-threads              # 启用线程支持

# 编译并安装（-j 后接 CPU 核心数，加速编译，如 -j4）
make -j4 && make install
```

### 4. 配置 Erlang 环境变量

将 Erlang 的 `bin` 目录添加到系统环境变量，确保全局可调用：

```bash
# 编辑系统环境变量配置文件
cat >> /etc/profile <<EOF
# Erlang 环境变量（2025-08-27 添加）
export PATH=\$PATH:/usr/local/erlang/bin
EOF

# 生效环境变量（无需重启系统）
source /etc/profile
```

### 5. 验证 Erlang 安装

执行以下命令验证 Erlang 是否安装成功：

```bash
# 查看 Erlang 版本（若输出版本信息，说明安装成功）
erl --version
```

## 三、安装 RabbitMQ（3.8.35）

### 1. 解压 RabbitMQ 安装包

假设 RabbitMQ 安装包已下载至 `/opt` 目录，执行以下命令：

```bash
# 进入安装包目录
cd /opt

# 解压 RabbitMQ 安装包（通用 Unix 版本，无需编译）
tar -xf rabbitmq-server-generic-unix-3.8.35.tar.xz

# 创建软链接（简化路径，避免版本号变更导致命令失效）
ln -s /opt/rabbitmq_server-3.8.35 /opt/rabbitmq
```

### 2. 配置 RabbitMQ 环境变量

将 RabbitMQ 的 `sbin` 目录添加到系统环境变量：

```bash
# 编辑系统环境变量配置文件
cat >> /etc/profile <<EOF

# RabbitMQ 环境变量（2025-08-27 添加）
export PATH=\$PATH:/opt/rabbitmq/sbin
EOF

# 生效环境变量
source /etc/profile
```

### 3. 创建 systemctl 服务配置文件

```bash
cat > /etc/systemd/system/rabbitmq.service <<EOF
[Unit]
# 服务描述
Description=RabbitMQ 3.8.35 Service
# 服务依赖：网络启动后、Erlang 环境就绪后启动
After=network.target
# 服务文档（可选）
Documentation=https://www.rabbitmq.com/

[Service]
# 启动类型：forking（后台运行，需指定 PID 文件）
Type=forking
# 启动用户：建议用普通用户（如 rabbitmq），避免 root 权限过高（需提前创建用户）
User=root
# 启动组（与用户一致）
Group=root

# RabbitMQ 启动命令（指定 PID 文件路径，便于 systemd 管理）
ExecStart=/opt/rabbitmq/sbin/rabbitmq-server -detached

# RabbitMQ 停止命令（通过官方工具停止，避免强制 kill）
ExecStop=/opt/rabbitmq/sbin/rabbitmqctl stop

# RabbitMQ 重启命令（先停止再启动）
ExecReload=/opt/rabbitmq/sbin/rabbitmqctl stop && /opt/rabbitmq/sbin/rabbitmq-server -detached

# 关键配置：确保服务稳定运行
PIDFile=/opt/rabbitmq/var/lib/rabbitmq/mnesia/rabbit@$(hostname).pid  # PID 文件路径（动态匹配主机名）
Restart=on-failure  # 服务失败时自动重启
RestartSec=5        # 重启间隔（5秒）
TimeoutStartSec=60  # 启动超时时间（60秒，避免长时间阻塞）
TimeoutStopSec=30   # 停止超时时间（30秒）

# 资源限制（可选，根据服务器配置调整）
LimitNOFILE=65535  # 最大文件句柄数（避免高并发时文件句柄不足）
LimitNPROC=65535   # 最大进程数

[Install]
# 多用户模式下开机自启（加入 multi-user.target 目标）
WantedBy=multi-user.target
EOF
```

### 4.创建用户和赋权

```bash
# 1. 创建 rabbitmq 用户（若未创建）
useradd -m rabbitmq

# 2. 授予目录权限（关键！避免启动时权限不足）
chown -R rabbitmq:rabbitmq /opt/rabbitmq
chmod -R 755 /opt/rabbitmq  # 目录权限：所有者读写执行，其他读执行
```

### 5. 启动rabbitMQ 服务

```bash
# 重新加载配置文件
systemctl daemon-reload
# 启动服务
systemctl start rabbitmq
# 查看状态
systemctl status rabbitmq
# 启用开机自启
systemctl enable rabbitmq
```

### 6. 启用 RabbitMQ 管理界面（可视化操作）

RabbitMQ 提供 Web 管理界面，需手动启用对应插件：

```bash
# 启用管理界面插件
rabbitmq-plugins enable rabbitmq_management

# 查看已安装的插件列表（确认 rabbitmq_management 状态为 [E*]）
rabbitmq-plugins list
```

**成功标识**：列表中 `rabbitmq_management` 行显示 `[E*]`（E = 启用，*= 自动启用的依赖插件）。

## 四、RabbitMQ 用户配置（管理员权限）

默认情况下，RabbitMQ 有一个 `guest` 用户，但仅允许本地登录，需创建新用户用于远程访问。

### 1. 创建管理员用户

```bash
# 创建用户（用户名：rabbitmq，密码：123456，可根据需求修改）
rabbitmqctl add_user vhr VHRrabbitmq@admin
```

### 2. 为用户添加管理员标签

```bash
# 设置用户标签为 administrator（最高权限，可管理所有资源）
rabbitmqctl set_user_tags vhr administrator
```

### 3. 授予用户全局权限

```bash
# 授予用户在根虚拟主机（/）下的所有权限（配置、写、读）
rabbitmqctl set_permissions -p "/" rabbitmq ".*" ".*" ".*"
```

- 权限说明：`"."` 表示匹配所有资源，三个 `"."` 分别对应 **配置权限**、**写权限**、**读权限**。

### 4. 验证用户配置

```bash
# 查看所有用户列表（确认 rabbitmq 用户存在且标签为 administrator）
rabbitmqctl list_users
```

**成功标识**：输出类似 `rabbitmq [administrator]`。

## 五、最终验证

### 1. 检查 RabbitMQ 端口监听

RabbitMQ 主要使用以下端口，确保端口正常监听：

```bash
# 查看 5672（AMQP 协议端口，客户端连接用）和 15672（管理界面端口）
netstat -nptl | grep -E "5672|15672"
```

**成功标识**：输出中包含 `LISTEN` 状态的 5672 和 15672 端口。

### 2. 访问 Web 管理界面

通过浏览器访问 RabbitMQ 管理界面，地址格式：
`http://服务器IP:15672`

- 示例：`http://172.16.6.154:15672`
- 登录账号：`rabbitmq`，密码：`123456`

**成功标识**：登录后可看到 RabbitMQ 控制台首页，显示「Nodes」为 `rabbit@主机名`（状态为 `running`）。

## 六、常用命令（后续维护用）

| 功能               | 命令                                                 |
| ------------------ | ---------------------------------------------------- |
| 启动 RabbitMQ 服务 | `rabbitmq-server start` 或 `nohup rabbitmq-server &` |
| 停止 RabbitMQ 服务 | `rabbitmqctl stop`                                   |
| 重启 RabbitMQ 服务 | `rabbitmqctl stop && nohup rabbitmq-server &`        |
| 查看服务状态       | `rabbitmqctl status`                                 |
| 查看队列列表       | `rabbitmqctl list_queues`                            |
| 查看虚拟主机列表   | `rabbitmqctl list_vhosts`                            |
| 禁用管理界面插件   | `rabbitmq-plugins disable rabbitmq_management`       |

## 七、注意事项

1. **防火墙配置**：
   若服务器启用防火墙（firewalld），需开放 5672 和 15672 端口：

   ```bash
   # 开放端口（永久生效）
   firewall-cmd --zone=public --add-port=5672/tcp --permanent
   firewall-cmd --zone=public --add-port=15672/tcp --permanent
   # 重启防火墙
   firewall-cmd --reload
   ```

2. **日志查看**：
   RabbitMQ 日志默认存储在 `/opt/rabbitmq/var/log/rabbitmq/` 目录，若服务启动失败，可查看日志排查问题：

   ```bash
   cat /opt/rabbitmq/var/log/rabbitmq/rabbit@主机名.log
   ```

3. **密码安全**：
   生产环境中需修改默认密码（`123456`），建议使用包含大小写字母、数字和特殊字符的复杂密码。