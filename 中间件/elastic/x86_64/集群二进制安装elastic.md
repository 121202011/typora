| 服务名称 | 下载地址                                                     | 备注 |
| -------- | ------------------------------------------------------------ | ---- |
| elastic  | https://www.elastic.co/downloads/past-releases#elasticsearch |      |

# elastic安装部署

解压安装包（三台机器）

```
tar -xf elasticsearch-7.17.9-linux-x86_64.tar.gz
```

添加软连接（三台机器）

```
ln -s elasticsearch-7.17.9 elasticsearch
```

创建存储目录（三台机器）

```
mkdir /opt/elk/elasticsearch/data
mkdir /opt/elk/elasticsearch/logs
```

修改配置文件（跨站后面的相关配置需要证书文件再加入）三台机器

```
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

```
useradd es
chown -R es:es /opt/elk/elasticsearch/
```

添加证书存储路径（三台机器）

```
cd /opt/elk/elasticsearch/config/  && mkdir certs && cd certs
```

证书生成（主节点）

```
/opt/elk/elasticsearch/bin/elasticsearch-certutil ca 
```

添加证书权限（三台机器）

```
chmod +rx /etc/elasticsearch/config/elastic-certificates.p12
```

起服务（三台机器）

```
systemctl start elasticsearch
systemctl status elasticsearch
systemctl enable elasticsearch
```

手动生成密码（主节点）

```
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

验证密码

```bash
curl -u elastic:Belloai@123 '172.16.112.66:9200/_cat/health?v'
curl -u elastic:Belloai@123 '172.16.112.66:9200/_cat/nodes?v'
```

修改密码

```bash
curl -H "Content-Type: application/json" -X POST -u elastic:'Belloai@123' '172.16.112.66:9200/_xpack/security/user/elastic/_password' -d '{ "password" : "Belloai_123" }'
```