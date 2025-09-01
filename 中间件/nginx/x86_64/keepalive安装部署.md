# Keepalived 安装部署（高可用 VIP 配置）

（承接 Nginx 容器化部署，为 Nginx 服务配置双节点高可用，解决单点故障问题）

## 1. 前置准备：上传 ISO 镜像

### 1.1 用途说明

因无外网环境时需通过 **本地 YUM 源** 安装 Keepalived，需先上传麒麟系统 ISO 镜像（内含系统依赖包）。

### 1.2 操作步骤

1. 将镜像文件 `Kylin-Server-V10-SP3-2403-Release-20240426-x86_64.iso` 上传至服务器；
2. 建议存放路径：`/opt/iso`（后续步骤默认此路径，若修改需同步调整命令）。

## 2. 搭建本地 YUM 源（基于 ISO 镜像）

### 2.1 创建目录（ISO 存放与挂载点）

先创建镜像存放目录和挂载目录，确保权限正常：

```bash
# 1. 创建 ISO 存放目录（若已上传，确认目录存在）
mkdir -p /opt/iso
chmod 0755 /opt/iso  # 只读+执行权限，防止误删镜像

# 2. 创建 ISO 挂载点目录（镜像内容将挂载至此，供 YUM 源读取）
mkdir -p /opt/kylin
chmod 0755 /opt/kylin
```

### 2.2 临时挂载 ISO 镜像

将 ISO 镜像模拟为 “块设备” 挂载到指定目录，使系统能读取镜像内的软件包：

```bash
# -t iso9660：指定 ISO 标准文件系统类型
# -o ro,loop：ro=只读挂载（保护镜像），loop=将文件模拟为块设备
mount -t iso9660 -o ro,loop /opt/iso/Kylin-Server-V10-SP3-2403-Release-20240426-x86_64.iso /opt/kylin

# 验证挂载：若能列出 /opt/kylin 下的 Packages、repodata 目录，说明挂载成功
ls /opt/kylin
```

### 2.3 配置本地 YUM 源文件

创建 YUM 源配置文件，让系统识别本地挂载目录为软件源：

```bash
cat > /etc/yum.repos.d/kylin-media.repo << EOF
[kylin-media]                # YUM 源唯一标识（不可与其他源重复）
name=Kylin Server Local Media  # 源名称（自定义，便于识别）
baseurl=file:///opt/kylin     # 源地址（本地挂载点绝对路径）
enabled=1                     # 启用该源（1=启用，0=禁用）
gpgcheck=0                    # 禁用 GPG 签名校验（本地源无需校验，加速安装）
EOF
```

### 2.4 刷新 YUM 缓存

清除旧缓存并生成新缓存，确保系统优先使用本地源：

```bash
yum clean all  # 清除旧缓存（避免读取外网源残留信息）
yum makecache  # 生成新缓存（扫描本地源软件包信息）
```

## 3. 安装 Keepalived

主从节点均需执行此步骤，通过本地 YUM 源安装 Keepalived 服务：

```bash
yum -y install keepalived  # -y 自动确认安装，无需手动输入 y

# 验证安装：查看版本号，确认安装成功
keepalived -v
```

## 4. 配置 Keepalived（主从节点区分）

Keepalived 基于 **VRRP 协议** 实现主从通信与 VIP 漂移，核心是 “主节点持 VIP，从节点备用，异常时自动切换”。需分别配置主、从节点，关键参数（如虚拟路由 ID、认证密码）必须一致。

### 4.1 主节点配置（MASTER，IP：10.172.131.66）

主节点默认持有 VIP，优先级高于从节点，配置如下：

```bash
cat > /etc/keepalived/keepalived.conf <<EOF
# 全局配置
global_defs {
    router_id RABBITMQ_CLUSTER  # 路由标识（集群内唯一，可自定义为 Nginx_CLUSTER）
    script_user root            # 执行健康检查脚本的用户（需 root 权限操作容器）
    enable_script_security      # 启用脚本安全检查（防止低权限用户调用危险脚本）
}

# Nginx 健康检查脚本（监控 Nginx 状态，触发主从切换）
vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"  # 脚本路径（后续创建）
    interval 30                               # 检查间隔（30秒/次，平衡实时性与资源）
    weight -31                                # 脚本异常时，节点权重降低31（主节点100-31=69 < 从节点90）
}

# VRRP 实例（核心配置，定义 VIP 与主从规则）
vrrp_instance VI_1 {
    state MASTER                # 节点角色（MASTER=主节点）
    interface enp4s3            # 绑定网卡（需与服务器实际网卡一致，用 ip addr 查看）
    virtual_router_id 12        # 虚拟路由 ID（主从必须相同，范围 0-255）
    priority 100                # 节点优先级（主节点 > 从节点）
    advert_int 1                # VRRP 心跳间隔（1秒/次，主节点向从节点发状态）
    authentication {            # 主从认证（防止非法节点加入）
        auth_type PASS          # 认证类型（PASS=简单密码）
        auth_pass 1111          # 认证密码（主从必须相同，4-8位）
    }
    track_script {              # 关联健康检查脚本
        chk_nginx
    }
    virtual_ipaddress {         # 共享 VIP（主从相同，需与网卡同网段）
        10.172.133.184/24
    }
    unicast_src_ip 10.172.131.66  # 主节点自身 IP（单播模式，避免广播风暴）
    unicast_peer {                # 从节点 IP（单播通信目标）
        10.172.131.174
    }
}
EOF
```

### 4.2 从节点配置（BACKUP，IP：10.172.131.174）

从节点默认不持 VIP，仅主节点异常时接管，配置与主节点差异仅 3 处：

```bash
cat > /etc/keepalived/keepalived.conf <<EOF
global_defs {
    router_id RABBITMQ_CLUSTER  # 与主节点一致
    script_user root
    enable_script_security
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 30
    weight -31
}

vrrp_instance VI_1 {
    state BACKUP                # 角色改为 BACKUP（从节点）
    interface enp4s3            # 与主节点一致（确认从节点网卡名相同）
    virtual_router_id 12        # 与主节点一致
    priority 90                 # 优先级低于主节点（90 < 100）
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111          # 与主节点一致
    }
    track_script {
        chk_nginx
    }
    virtual_ipaddress {
        10.172.133.184/24       # 与主节点共享 VIP
    }
    unicast_src_ip 10.172.131.174  # 从节点自身 IP
    unicast_peer {
        10.172.131.66             # 主节点 IP（与主节点 unicast_peer 对应）
    }
}
EOF
```

### 4.3 生成 Nginx 健康检查脚本（主从均需配置）

脚本作用：检查 Nginx 容器状态（配置语法 + 进程存活），异常时返回非 0，触发 Keepalived 权重调整：

```bash
# 1. 创建脚本文件
cat > /etc/keepalived/check_nginx.sh <<EOF
#!/bin/bash
# Nginx 容器健康检查脚本（Keepalived 调用）

# 定义 Nginx 容器名称（需与 Nginx 部署的 docker-compose.yml 中 container_name 一致）
NGINX_CONTAINER="nginx"

# 步骤1：检查 Nginx 配置语法（优先验证配置，避免配置错误导致服务异常）
docker exec "$NGINX_CONTAINER" nginx -t >/dev/null 2>&1
exit_status=$?  # 保存命令结果（0=正常，非0=异常）

# 步骤2（可选）：额外检查 Nginx 进程存活（双重保障，按需启用）
# if [ $exit_status -eq 0 ]; then
#     docker exec "$NGINX_CONTAINER" pgrep -x nginx >/dev/null 2>&1
#     exit_status=$?
# fi

# 步骤3：返回结果（Keepalived 根据结果调整权重）
if [ $exit_status -eq 0 ]; then
    exit 0  # 服务正常：维持权重
else
    exit 1  # 服务异常：降低权重，触发切换
fi
EOF

# 2. 配置执行权限（必须，否则 Keepalived 无法调用）
chmod +x /etc/keepalived/check_nginx.sh
```

## 5. Keepalived 服务操作（主从均需执行）

### 5.1 启动服务

```bash
systemctl start keepalived  # 启动 Keepalived
```

### 5.2 配置开机自启

避免服务器重启后服务失效，需设置开机自启：

```bash
systemctl enable keepalived  # 配置自启

# 验证自启：返回 enabled 说明成功
systemctl is-enabled keepalived
```

### 5.3 查看状态与 VIP 绑定

#### 5.3.1 查看服务状态

```bash
systemctl status keepalived
# 显示 "active (running)" 说明服务正常
```

#### 5.3.2 查看 VIP 绑定

- **主节点**：正常时会绑定 VIP，执行命令可见结果：

  ```bash
  ip addr | grep 10.172.133.184
  # 示例输出：inet 10.172.133.184/24 scope global secondary enp4s3
  ```

- **从节点**：正常时无 VIP 绑定，仅主节点异常时接管。

#### 5.3.3 查看日志（排查问题）

Keepalived 日志默认存于 `/var/log/messages`，可查看通信与切换记录：

```bash
# 实时查看 VRRP 相关日志
tail -f /var/log/messages | grep VRRP
```

## 6. 高可用验证（关键步骤）

模拟主节点故障，验证从节点是否自动接管 VIP：

### 6.1 主节点故障模拟

在主节点（10.172.131.66）停止 Nginx 容器：

```bash
docker stop nginx  # 模拟 Nginx 服务异常
```

### 6.2 观察从节点接管

等待 30 秒（健康检查间隔），在从节点执行：

```bash
ip addr | grep 10.172.133.184
# 若能看到 VIP 绑定，说明从节点成功接管
```

### 6.3 主节点恢复验证

在主节点重启 Nginx 容器：

```bash
docker start nginx  # 恢复 Nginx 服务
```

等待 30 秒后，在主节点执行：

```bash
ip addr | grep 10.172.133.184
# 若主节点重新绑定 VIP，从节点 VIP 消失，说明切换正常
```

## 7. 注意事项

1. **网卡名一致性**：主从节点 `interface` 参数（如 `enp4s3`）需与实际网卡一致，用 `ip addr` 确认；

2. **VIP 网段**：VIP 需与主从节点网卡同网段，否则无法通信；

3. **脚本权限**：`check_nginx.sh` 必须有执行权限（`chmod +x`），否则检查失效；

4. **防火墙配置**：若启用防火墙，需开放 VRRP 协议（默认多播地址 224.0.0.18）：

   ```bash
   # 适用于 firewalld，永久开放 VRRP
   firewall-cmd --add-protocol=vrrp --permanent
   firewall-cmd --reload
   ```