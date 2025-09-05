

# window安装部署filebeat

```shell
以管理员身份运行 PowerShell，将解压缩的目录移动到指定路径中
mv filebeat-5.1.2-windows-x86_64 "C:\Program Files\Filebeat"

安装 Filebeat 为 Windows 服务
cd "C:\Program Files\Filebeat"
powershell.exe -ExecutionPolicy UnRestricted -File .\install-service-filebeat.ps1

编辑filebeat.yml配置文件并测试您的配置
.\filebeat.exe -e -configtest

（可选）在前台运行Filebeat以确保一切正常。 按Ctrl + C退出
.\filebeat.exe -c filebeat.yml -e -d "*"

启动服务
Start-Service filebeat
```

#### 环境准备（java安装部署）

```shell
解压安装包
tar -xf jdk-8u321-linux-x64.tar.gz  -C /usr/local/

环境变量添加
vim /etc/profile
##JAVA
export JAVA_HOME=/usr/local/jdk1.8.0_321/
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:$CLASSPATH

使其生效
source  /etc/profile
```

# kafka安装部署

解压文件（三台机器）

```shell
tar -xzf kafka_2.13-3.1.0.tgz
```

添加软连接（三台机器）

```shell
ln -s kafka_2.13-3.1.0  kafka
```

创建存储路径（三台机器）

```shell
cd kafka
mkdir /opt/zookeeper/data
```

修改zookeeper.properties配置文件（三台机器）

```shell
#复制 ZooKeeper 配置文件
cp  config/zookeeper.properties config/zookeeper.properties.bak

#编辑配置文件
vim config/zookeeper.properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/zookeeper/data
clientPort=2181
server.1=172.24.100.186:2888:3888
server.2=172.24.100.187:2888:3888
server.3=172.24.100.188:2888:3888

#在 dataDir 目录下创建 myid 文件，并写入当前服务器的编号（对应 server.x 中的 x）
echo "1" >/opt/zookeeper/data/myid
```

修改server.properties配置文件（三台机器）

```shell
#复制 Kafka 配置文件
cp config/server.properties config/server.properties.bak

#编辑 config/server.properties 文件 (broker.id 每台服务器的唯一编号)
vim config/server.properties
broker.id=1
host.name=172.24.100.186
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/opt/kafka/logs/
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.cleanup.policy=delete
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=172.24.100.186:2181,172.24.100.187:2181,172.24.100.188:2181
zookeeper.connection.timeout.ms=6000
group.initial.rebalance.delay.ms=0
#listeners=PLAINTEXT://172.24.100.186:9092

#开启plain认证
listeners=SASL_PLAINTEXT://172.24.100.186:9092
security.inter.broker.protocol= SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=PLAIN
sasl.enabled.mechanisms=PLAIN

#添加权限认证
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
```

为 Kafka 服务器配置基于 PLAIN 机制的身份验证（三台机器）

```shell
配置 PLAIN 认证
vim kafka_server_jaas.conf
KafkaServer {
   org.apache.kafka.common.security.plain.PlainLoginModule required
   username="admin"
   password="jZ1]uD3&uP4-wZ"
   user_admin="jZ1]uD3&uP4-wZ"
   user_log="cC4_zM0]nF3!nH";
};
```

添加 KAFKA\_OPTS 环境变量让 Kafka 在启动时加载 JAAS 配置文件，以支持 PLAIN 认证（三台机器）

```shell

vim kafka-run-class.sh
KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/kafka_server_jaas.conf
```

客户端配置文件（三台机器）

```shell
vim kafka_client_auth.conf
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="log" password="cC4_zM0]nF3!nH";
```

客户端配置文件（三台机器）

```shell
vim  kafka_client_auth_admin.conf
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="jZ1]uD3&uP4-wZ";
```

服务启动与停止（三台机器）

```shell
#zk服务启动
./bin/zookeeper-server-start.sh -daemon config/zookeeper.properties

#停止服务
./bin/zookeeper-server-stop.sh

#kafka服务启动
bin/kafka-server-start.sh -daemon config/server.properties

#停止服务
bin/kafka-server-stop.sh

#查看topic清单列表
./bin/kafka-topics.sh --bootstrap-server 172.24.100.186:9092 --list --command-config config/kafka_client_auth_admin.conf
./bin/kafka-topics.sh --bootstrap-server 172.24.100.186:9092 --list --command-config config/kafka_client_auth.conf
```

kafka-map安装部署

```shell
mkdir -p /opt/kafka-map/data
docker run -d  -p 9090:8080   -v /opt/kafka-map/data:/usr/local/kafka-map/data   -e DEFAULT_USERNAME=admin   -e DEFAULT_PASSWORD=CIIsh123,   --name kafka-map   --restart always harbor.ciicsh.com/library/kafka-map:latest
```

# logstash安装部署

创建安装目录（二台机器）

```shell
mkdir /opt/elk
cd /opt/elk
```

解压安装包（二台机器）

```shell
tar -xf logstash-7.17.9-linux-x86_64.tar.gz
```

添加软连接（二台机器）

```shell
ln -s logstash-7.17.9 logstash
```

修改配置文件（二台机器）

```shell
cd logstash/config/
vim pipelines.yml
- pipeline.id: main
  path.config: "/opt/elk/logstash/config/pipelines/main.conf"
```

添加配置文件，输入kafka相关数据（二台机器）

```shell
vim /opt/elk/logstash/config/pipelines/main.conf
input {
  kafka {
    bootstrap_servers => "172.24.100.186:9092,172.24.100.187:9092,172.24.100.188:9092"
    jaas_path => "/opt/elk/logstash/config/kafka_client_jaas.conf"
    security_protocol => "SASL_PLAINTEXT"
    sasl_mechanism => "PLAIN"
    topics => ["mssql-errorlog"]
    auto_offset_reset => "latest"
    group_id => "mssql"
    consumer_threads => 1
    decorate_events => true
    codec => "json"
  }
}

filter {

  grok {
    match => { "[host][ip]" => "%{IPV4:ipv4}" }
  }


  mutate {
    remove_field => ["@version","[fileset][name]","[agent][ephemeral_id]","[agent][hostname]","[agent][id]","[agent][name]","[ecs][version]","[host][architecture]","[host][id]","[host][mac]","[host][name]","[host][os][build]","[host][os][family]","[host][os][kernel]","[host][os][platform]","[host][os][type]","[host][os][version]","[input][type]","[log][offset]","[log][file][path]","[servie][type]"]
  }

  ruby {
    code => "
      original_time = event.get('@timestamp')
      if original_time
        # 将 @timestamp 字段的时间加 8 小时
        new_time = original_time + 8*3600  # 8 小时 = 8*3600 秒
        event.set('asi', new_time)
      end
    "
  }

}

output {
  elasticsearch {
    hosts => ["172.24.100.186:9200", "172.24.100.187:9200", "172.24.100.188:9200"]
    user => "elastic"
    password => "O1NgDycq%9ZL"
    index => "mssql-errorlog-%{+YYYY.MM.dd}"
  }
}

```

添加客户端文件（二台机器）

```shell
vim /opt/elk/logstash/config/kafka_client_jaas.conf
KafkaClient {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="log"
  password="cC4_zM0]nF3!nH";
};

```

添加service启动文件（二台机器）

```shell
vim /lib/systemd/system/logstash.service
[Unit]
Description=logstach

[Service]
ExecStart=/opt/elk/logstash/bin/logstash
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

修改logstash配置文件（二台机器）

```shell
vim /opt/elk/logstash/config/logstash.yml
http.host: 0.0.0.0
http.port: 9600-9700
log.level: info
path.logs: /opt/elk/logstash/logs
```

添加日志路径（二台机器）

```shell
 mkdir /opt/elk/logstash/logs
```

起服务，查看服务状态,开机自启（二台机器）

```shell
systemctl start logstash
systemctl status logstash
systemctl enable logstash
```

# elastic安装部署

解压安装包（三台机器）

```shell
tar -xf elasticsearch-7.17.9-linux-x86_64.tar.gz
```

添加软连接（三台机器）

```shell
ln -s elasticsearch-7.17.9 elasticsearch
```

创建存储目录（三台机器）

```shell
mkdir /opt/elk/elasticsearch/data
mkdir /opt/elk/elasticsearch/logs
```

修改配置文件（跨站后面的相关配置需要证书文件再加入）三台机器

```shell
vim /opt/elk/elasticsearch/config/elasticsearch.yml
#cluster
cluster.name: mssql-errorlog01
node.name: PRD-OPS-SQLSER-LOG03
path.data: /opt/elk/elasticsearch/data
path.logs: /opt/elk/elasticsearch/logs
network.host: 0.0.0.0
http.port: 9200
transport.tcp.port: 9300
discovery.seed_hosts: ["172.24.100.186", "172.24.100.187", "172.24.100.188"]
discovery.zen.minimum_master_nodes: 2
cluster.initial_master_nodes: ['PRD-OPS-SQLSER-LOG01']

#跨站访问
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization

#X-Pack
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /opt/elk/elasticsearch/config/certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /opt/elk/elasticsearch/config/certs/elastic-certificates.p12

#role
node.roles: [ master, data, ingest ]

```

添加用户，并赋权（三台机器）

```shell
useradd es
chown -R es:es /opt/elk/elasticsearch/
```

添加证书存储路径（三台机器）

```shell
cd /opt/elk/elasticsearch/config/  && mkdir certs && cd certs
```

证书生成（主节点）

```shell
/opt/elk/elasticsearch/bin/elasticsearch-certutil ca 
```

添加证书权限（三台机器）

```shell
chmod +rx /etc/elasticsearch/config/elastic-certificates.p12
```

起服务（三台机器）

```shell
systemctl start elasticsearch
systemctl status elasticsearch
systemctl enable elasticsearch
```

手动生成密码（主节点）

```shell
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

验证密码

```shell
curl -u elastic:Belloai@123 '172.16.112.66:9200/_cat/health?v'
curl -u elastic:Belloai@123 '172.16.112.66:9200/_cat/nodes?v'

```

修改密码

```shell
curl -H "Content-Type: application/json" -X POST -u elastic:'Belloai@123' '172.16.112.66:9200/_xpack/security/user/elastic/_password' -d '{ "password" : "Belloai_123" }'
```

