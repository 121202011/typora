| 服务名称  | 下载地址                                           | 备注 |
| --------- | -------------------------------------------------- | ---- |
| kafka-map | https://hub.docker.com/r/dushixiang/kafka-map/tags |      |

#### 容器化部署kafka-map

```bash
#创建存储路径
mkdir -p /opt/kafka-map/data

#起服务
docker run -d  -p 9090:8080   -v /opt/kafka-map/data:/usr/local/kafka-map/data   -e DEFAULT_USERNAME=admin   -e DEFAULT_PASSWORD=CIIsh123,   --name kafka-map   --restart always harbor.ciicsh.com/library/kafka-map:latest
```

