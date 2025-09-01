### 部署 RabbitMQ 服务

| 服务名称 | 下载地址                                   | 备注 |
| -------- | ------------------------------------------ | ---- |
| rabbitmq | https://hub.docker.com/_/rabbitmq/tags     |      |
| Plugins  | https://www.rabbitmq.com/community-plugins |      |

**所有节点的 erlang.cookie 文件内容相同，erlang.cookie 是用于节点间认证的文件。**

#### 一、准备工作

1. 上传相关安装包至服务器

   - docker-compose.yml（编排文件）

   - rabbitmq-management_3.13.7-amd64.tar.gz（镜像包）

   - rabbitmq_delayed_message_exchange-3.13.0.ez（延迟消息插件）

     

#### 二、基础配置

1.**创建启动编排文件**

```bash
cat >/opt/rabbitmq/docker-compose.yml <<EOF
services:
  rabbitmq:
    container_name: rabbitmq
    image: harbor.ciicsh.com/base/rabbitmq:3.13.7-management
    privileged: true
    network_mode: host
    restart: always
    volumes:
      - /opt/rabbitmq/data:/var/lib/rabbitmq
      - /opt/rabbitmq/plugins/rabbitmq_delayed_message_exchange-3.13.0.ez:/plugins/rabbitmq_delayed_message_exchange-3.13.0.ez
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=VHRrabbitmq@admin
EOF
```

2.**配置主机映射（服务间通信）**

```bash
vim /etc/hosts

主机ip1    主机名1
主机ip2    主机名2
主机ip3    主机名3
```

3.**创建目录结构**

```bash
mkdir -p /opt/rabbitmq/{data,plugins}
```

#### 三、环境部署

1.**导入本地镜像**

```bash
docker load < rabbitmq-management_3.13.7-amd64.tar.gz
```

2.**设置目录权限**

```bash
chown -R 1000:1000 /opt/rabbitmq
```

3.**服务操作**

启动服务：

```bash
cd /opt/rabbitmq
docker-compose -f /opt/rabbitmq/docker-compose.yml up -d
```

停止服务：

```bash
docker-compose -f /opt/rabbitmq/docker-compose.yml down
```

查看状态：

```bash
docker-compose ps -a
```

#### 四、集群配置（从节点操作）

1. 停止应用服务

   ```bash
   rabbitmqctl stop_app
   ```

2. 重置节点配置

   ```bash
   rabbitmqctl reset
   ```

3. 加入集群

   ```bash
   rabbitmqctl join_cluster node_name
   ```

4. 启动应用服务

   ```bash
   rabbitmqctl start_app
   ```

5. 验证集群状态

   ```bash
   rabbitmqctl cluster_status
   ```

#### 五、业务账号与权限配置

1. 创建业务用户

   ```bash
   rabbitmqctl add_user vhr VHRrabbitmq@admin
   ```

2. 配置管理员权限（可选）

   ```bash
   rabbitmqctl set_user_tags vhr administrator
   ```

3. 创建专属虚拟主机

   ```bash
   rabbitmqctl add_vhost vhr
   ```

4. 分配虚拟主机权限

   ```bash
   rabbitmqctl set_permissions -p vhr vhr ".*" ".*" ".*"
   ```

5. 配置高可用策略

   ```bash
   rabbitmqctl set_policy -p vhr ha-mode ".*" '{"ha-mode":"all","ha-sync-mode":"automatic"}' --priority 0 --apply-to all
   ```

6. 验证配置

   - 查看策略：`rabbitmqctl list_policies -p vhr`
   - 查看虚拟主机：`rabbitmqctl list_vhosts`
   - 查看用户列表：`rabbitmqctl list_users`

#### 六、插件安装（延迟消息插件）

```bash
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```