#### Docker 容器化部署 RocketMQ Dashboard

```bash
docker run -d \
  --name rocketmq-dashboard \
  -e "JAVA_OPTS=-Drocketmq.namesrv.addr=172.16.8.201:9876;172.16.8.202:9876;172.16.8.203:9876" \
  -p 8082:8080 \
  -t apacherocketmq/rocketmq-dashboard:latest

备注：多个 NameServer 地址必须严格用 分号 ; 分隔
```

