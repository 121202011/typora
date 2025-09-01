### RabbitMQ 集群配置指南

#### 一、前提条件

1. 三台服务器已安装相同版本的 RabbitMQ
2. 服务器间可通过 IP 相互通信，防火墙开放以下端口：
   - 5672（AMQP 协议端口）
   - 15672（管理界面端口）
   - 25672（集群通信端口）
3. 所有节点的 `erlang.cookie` 文件内容必须一致（用于节点间认证）

#### 二、关键配置步骤

##### 1. 同步 erlang.cookie 文件

`erlang.cookie` 文件路径：`/var/lib/rabbitmq/.erlang.cookie`

- 选择一台节点的 cookie 文件作为标准
- 将其内容复制到其他所有节点，确保文件内容完全一致

##### 2. 配置主机名解析

```bash
cat << EOF >> /etc/hosts
172.24.23.211 rabbitmq-1
172.24.23.212 rabbitmq-2
172.24.23.213 rabbitmq-3
EOF
```

#### 三、集群搭建（以 rabbitmq-1 为基准节点）

在 rabbitmq-2 和 rabbitmq-3 节点执行以下操作：

1. 停止 RabbitMQ 应用

   ```bash
   rabbitmqctl stop_app
   ```

2. 加入集群

   ```bash
   rabbitmqctl join_cluster rabbit@rabbitmq-1
   ```

3. 启动 RabbitMQ 应用

   ```bash
   rabbitmqctl start_app
   ```

4. 验证集群状态

   ```bash
   rabbitmqctl cluster_status
   ```

#### 四、启用管理界面

1. 启用管理插件

   ```bash
   rabbitmq-plugins enable rabbitmq_management
   ```

2. 重启 RabbitMQ 服务

   ```bash
   systemctl restart rabbitmq-server
   ```

#### 五、用户与权限管理

1. 创建管理员用户

   ```bash
   rabbitmqctl add_user admin password
   ```

2. 设置管理员角色

   ```bash
   rabbitmqctl set_user_tags admin administrator
   ```

3. 授予全权限

   ```bash
   rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
   ```

4. 查看用户列表

   ```bash
   rabbitmqctl list_users
   ```

#### 六、虚拟主机管理

##### 虚拟主机适用场景

- **多租户隔离**：不同租户使用独立虚拟主机，资源互不干扰
- **业务隔离**：如电商系统中，订单业务与库存业务可使用不同虚拟主机

##### 常用命令

1. 创建虚拟主机

   ```bash
   rabbitmqctl add_vhost my_vhost
   ```

2. 删除虚拟主机

   ```bash
   rabbitmqctl delete_vhost my_vhost
   ```

3. 查看虚拟主机列表

   ```bash
   rabbitmqctl list_vhosts
   ```

4. 权限管理

   - 查看指定虚拟主机的用户权限

     ```bash
     rabbitmqctl list_permissions -p <vhost>
     ```

   - 清除用户在特定虚拟主机的权限

     ```bash
     rabbitmqctl clear_permissions -p <vhost> <username>
     ```