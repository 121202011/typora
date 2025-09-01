## 1. 解压安装包至指定目录

将 Kibana 源码包解压到 `/usr/local/` 目录，并创建软链接便于管理：

```bash
# 解压源码包到 /usr/local 目录
tar -xf kibana-7.6.0-linux-x86_64.tar.gz -C /usr/local/

# 进入 /usr/local 目录
cd /usr/local/

# 创建软链接（简化路径引用）
ln -s kibana-7.6.0-linux-x86_64/ kibana

# 进入 Kibana 安装目录
cd kibana
```

## 2. 修改 Kibana 配置文件

编辑 `config/kibana.yml` 配置文件，设置基础参数：

```bash
vim config/kibana.yml
```

配置内容如下（根据实际环境调整）：

```yaml
# 服务端口（默认 5601）
server.port: 5601

# 允许访问的主机地址（0.0.0.0 表示所有地址）
server.host: "0.0.0.0"

# 关联的 Elasticsearch 地址（替换为实际 ES 地址）
elasticsearch.hosts: ["http://localhost:9200"]

# 界面语言设置为中文
i18n.locale: "zh-CN"
```

## 3. 配置目录权限

创建并授权 `es` 用户（非 root 用户运行更安全）：

```bash
# 创建 es 用户（若已存在可跳过）
useradd es -s /bin/bash

# 递归授权 Kibana 目录给 es 用户
chown -R es:es /usr/local/kibana-7.6.0-linux-x86_64/
```

## 4. 创建 systemd 服务文件

通过 systemd 管理 Kibana 服务（支持开机自启、状态监控等）：

```bash
# 创建服务配置文件
cat > /etc/systemd/system/kibana.service <<EOF
[Unit]
Description=Kibana Analytics Service
Documentation=https://www.elastic.co/guide/en/kibana/current/index.html
After=network.target elasticsearch.service  # 依赖网络和 Elasticsearch 服务
Wants=elasticsearch.service  # 启动 Kibana 时自动启动 Elasticsearch

[Service]
User=es  # 运行用户
Group=es  # 运行用户组
WorkingDirectory=/usr/local/kibana  # 工作目录
ExecStart=/usr/local/kibana/bin/kibana  # 启动命令
Restart=always  # 异常退出时自动重启
RestartSec=3  # 重启间隔 3 秒
LimitNOFILE=65536  # 提高文件描述符限制（避免高并发时受限）

[Install]
WantedBy=multi-user.target  # 多用户模式下启动
EOF
```

## 5. 管理 Kibana 服务

使用 systemctl 命令操作 Kibana 服务：

### 5.1 重载系统服务配置

```bash
systemctl daemon-reload
```

### 5.2 启动 Kibana 服务

```bash
systemctl start kibana
```

### 5.3 配置开机自启

```bash
systemctl enable kibana
```

### 5.4 查看服务状态

```bash
systemctl status kibana
# 若显示 "active (running)" 表示服务正常运行
```

### 5.5 停止 / 重启服务

```bash
# 停止服务
systemctl stop kibana

# 重启服务（配置修改后需执行）
systemctl restart kibana
```

## 6. 查看服务日志

通过 journalctl 查看 Kibana 运行日志（排查启动或运行异常）：

```bash
# 实时查看日志
journalctl -u kibana -f

# 查看最近 100 行日志
journalctl -u kibana -n 100
```

## 7. 访问 Kibana 界面

服务启动成功后，通过浏览器访问：

```plaintext
http://服务器IP:5601  # 例如 http://172.16.112.2:5601 或 http://172.24.100.185:5601
```

首次访问需等待初始化（约 1-2 分钟），成功后将显示中文界面。

## 注意事项

1. **版本兼容性**：Kibana 版本必须与 Elasticsearch 完全一致（均为 7.6.0），否则会出现通信失败

2. **端口占用**：确保 5601 端口未被占用，可通过 `netstat -tuln | grep 5601` 检查

3. **防火墙配置**：开放 5601 端口（示例为 firewalld）：

   ```bash
   firewall-cmd --add-port=5601/tcp --permanent
   firewall-cmd --reload
   ```

4. **资源调整**：若内存不足，可修改 `config/kibana.yml` 添加 `server.maxPayloadBytes: 10485760` 限制负载