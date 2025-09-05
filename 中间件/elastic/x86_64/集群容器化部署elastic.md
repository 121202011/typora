#### 1. 准备工作

##### 设置主机名

```bash
# 在三台服务器上分别设置主机名
# 节点1 (10.172.131.68):
hostnamectl set-hostname ecs-zhrlythptjsxm-xa-app-0101

# 节点2 (10.172.131.201):
hostnamectl set-hostname ecs-zhrlythptjsxm-xa-app-0100

# 节点3 (10.172.131.206):
hostnamectl set-hostname ecs-zhrlythptjsxm-xa-app-0102
```

##### 创建服务目录

```bash
# 在三台服务器上执行
mkdir -p /opt/elasticsearch/data
```

##### 设置目录权限

```bash
chown -R 1000:1000 /opt/elasticsearch/
```

#### 2. 系统配置优化

##### 配置用户组资源限制

```bash
vim /etc/security/limits.conf

# 添加以下内容
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096
```

##### 内核参数优化

```bash
vim /etc/sysctl.conf

# 添加以下内容
vm.swappiness=1
vm.max_map_count=655360
fs.file-max=204800

# 使配置生效
sysctl -p
```

#### 3. 证书生成（在一个节点执行即可）

##### 导入镜像

```bash
docker load < elasticsearch-8.17.6-x86.tar
```

##### 预运行容器进行证书签发

```bash
# 运行临时容器
docker run -it --name elasticsearch-container -w /usr/share/elasticsearch/bin \
  harbor.ciicsh.com/library/elasticsearch:8.17.6 /bin/bash

# 生成CA根证书
./elasticsearch-certutil ca --days 36500 --out ca.p12

# 生成节点证书（包含所有节点信息）
./elasticsearch-certutil cert \
  --ca /usr/share/elasticsearch/ca.p12 \
  --dns ecs-zhrlythptjsxm-xa-app-0101,ecs-zhrlythptjsxm-xa-app-0100,ecs-zhrlythptjsxm-xa-app-0102 \
  --ip 10.172.131.68,10.172.131.201,10.172.131.206 \
  --days 36500 \
  --out /usr/share/elasticsearch/elastic-certificates.p12 \
  --pass ""

# 退出容器
exit
```

##### 复制证书到宿主机

```bash
# 将证书复制到本地目录
docker cp elasticsearch-container:/usr/share/elasticsearch/ca.p12 /opt/elasticsearch/
docker cp elasticsearch-container:/usr/share/elasticsearch/elastic-certificates.p12 /opt/elasticsearch/

# 删除临时容器
docker rm elasticsearch-container

# 设置证书权限
chmod +rx /opt/elasticsearch/elastic-certificates.p12
chmod +rx /opt/elasticsearch/ca.p12
```

#### 4. 分发证书到所有节点

```bash
# 将证书分发到其他两个节点
scp /opt/elasticsearch/elastic-certificates.p12 root@10.172.131.201:/opt/elasticsearch/
scp /opt/elasticsearch/elastic-certificates.p12 root@10.172.131.206:/opt/elasticsearch/

scp /opt/elasticsearch/ca.p12 root@10.172.131.201:/opt/elasticsearch/
scp /opt/elasticsearch/ca.p12 root@10.172.131.206:/opt/elasticsearch/
```

#### 5. 配置文件设置

##### 节点1 (10.172.131.68) 配置

```bash
cat > /opt/elasticsearch/elasticsearch.yml <<EOF
# 集群配置
cluster.name: vhrelastic
node.name: ecs-zhrlythptjsxm-xa-app-0101

# 网络配置
network.host: 10.172.131.68
http.port: 9200
transport.port: 9300

# 集群发现
discovery.seed_hosts: ["10.172.131.68", "10.172.131.201", "10.172.131.206"]
cluster.initial_master_nodes: ["ecs-zhrlythptjsxm-xa-app-0101", "ecs-zhrlythptjsxm-xa-app-0100", "ecs-zhrlythptjsxm-xa-app-0102"]

# 跨域访问
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization

# X-Pack 安全配置
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.verification_mode: certificate
xpack.security.http.ssl.keystore.path: elastic-certificates.p12
xpack.security.http.ssl.keystore.password: ""
xpack.security.http.ssl.truststore.path: elastic-certificates.p12
xpack.security.http.ssl.truststore.password: ""
EOF
```

##### 节点2 (10.172.131.201) 配置

```bash
cat > /opt/elasticsearch/elasticsearch.yml <<EOF
# 集群配置
cluster.name: vhrelastic
node.name: ecs-zhrlythptjsxm-xa-app-0100

# 网络配置
network.host: 10.172.131.201
http.port: 9200
transport.port: 9300

# 集群发现
discovery.seed_hosts: ["10.172.131.68", "10.172.131.201", "10.172.131.206"]
cluster.initial_master_nodes: ["ecs-zhrlythptjsxm-xa-app-0101", "ecs-zhrlythptjsxm-xa-app-0100", "ecs-zhrlythptjsxm-xa-app-0102"]

# 跨域访问
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization

# X-Pack 安全配置
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.verification_mode: certificate
xpack.security.http.ssl.keystore.path: elastic-certificates.p12
xpack.security.http.ssl.keystore.password: ""
xpack.security.http.ssl.truststore.path: elastic-certificates.p12
xpack.security.http.ssl.truststore.password: ""
EOF
```

##### 节点3 (10.172.131.206) 配置

```bash
cat > /opt/elasticsearch/elasticsearch.yml <<EOF
# 集群配置
cluster.name: vhrelastic
node.name: ecs-zhrlythptjsxm-xa-app-0102

# 网络配置
network.host: 10.172.131.206
http.port: 9200
transport.port: 9300

# 集群发现
discovery.seed_hosts: ["10.172.131.68", "10.172.131.201", "10.172.131.206"]
cluster.initial_master_nodes: ["ecs-zhrlythptjsxm-xa-app-0101", "ecs-zhrlythptjsxm-xa-app-0100", "ecs-zhrlythptjsxm-xa-app-0102"]

# 跨域访问
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization

# X-Pack 安全配置
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.verification_mode: certificate
xpack.security.http.ssl.keystore.path: elastic-certificates.p12
xpack.security.http.ssl.keystore.password: ""
xpack.security.http.ssl.truststore.path: elastic-certificates.p12
xpack.security.http.ssl.truststore.password: ""
EOF
```

#### 6. Docker Compose 配置

```bash
cat > /opt/elasticsearch/docker-compose.yml <<EOF
version: '3.8'

services:
  elasticsearch:
    image: "harbor.ciicsh.com/library/elasticsearch:8.17.6"
    container_name: es
    environment:
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms4096m -Xmx4096m"
      - "ELASTIC_PASSWORD=VHRelastic@admin"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - /opt/elasticsearch/data:/usr/share/elasticsearch/data
      - /opt/elasticsearch/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - /opt/elasticsearch/elastic-stack-ca.p12:/usr/share/elasticsearch/config/elastic-stack-ca.p12
      - /opt/elasticsearch/elastic-certificates.p12:/usr/share/elasticsearch/config/elastic-certificates.p12
    restart: always
    ports:
      - 9200:9200
      - 9300:9300
    network_mode: "host"
EOF
```

#### 7. 启动服务

```bash
# 在三台服务器上分别执行
cd /opt/elasticsearch/
docker-compose -f docker-compose.yml up -d
```

#### 8. 验证集群状态

```bash
# 查看服务状态
docker-compose ps -a

# 添加kibana账号
docker exec -it es /bin/bash
bin/elasticsearch-reset-password -u kibana_system -i

# 查看集群节点状态
curl --basic -u elastic:VHRelastic@admin -k "https://10.172.131.206:9200/_cat/nodes?v"

# 查看集群健康状态
curl --basic -u elastic:VHRelastic@admin -k "https://10.172.131.206:9200/_cat/health?v"
```

##### 注意事项

1. 确保所有节点的时间同步
2. 确认防火墙已开放9200和9300端口
3. 确保所有节点的证书文件相同
4. 首次启动可能需要较长时间初始化集群
5. 如果集群无法形成，检查日志文件排查问题