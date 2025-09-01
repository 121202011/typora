| 安装包        | 下载地址                                                      |
| :--------- | :-------------------------------------------------------- |
| keepalived | <https://www.keepalived.org/download.html>                |
| rpm包下载地址   | <https://mirrors.aliyun.com/centos/7/os/x86_64/Packages/> |

解压安装

```shell
解压安装包
tar -xf keepalived-2.2.8.tar.gz

安装依赖
yum groupinstall "Development Tools" -y   （看情况）
rpm -ivh libnl-1.1.4-3.el7.x86_64.rpm
rpm -ivh libnl-devel-1.1.4-3.el7.x86_64.rpm

编译安装
cd keepalived-2.2.8/
./configure --prefix=/usr/local/keepalived
make && make install
```

修改配置文件

```shell
vrrp_script chk_rabbitmq {
  script "/usr/local/keepalived/etc/keepalived/check_rabbitmq.sh"
  interval 30
  weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface ens192
    virtual_router_id 11
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script {
        chk_rabbitmq
    }
    virtual_ipaddress {
        172.24.99.212
    }
}

```

健康检查脚本

```shell
[root@PRD-OPS-DEPLOY03 elk]# cat /usr/local/keepalived/etc/keepalived/check_rabbitmq.sh
#!/bin/bash
# 使用 rabbitmqctl 检查 RabbitMQ 状态
docker exec  rabbitmq3  rabbitmqctl -q status > /dev/null 2>&1
if [ $? -ne 0 ]; then
    # 尝试重启 RabbitMQ
    docker restart rabbitmq3
    sleep 30
    # 再次检查
    docker exec  rabbitmq3  rabbitmqctl -q status > /dev/null 2>&1
    if [ $? -ne 0 ]; then
        exit 1  # 检查失败，通知 Keepalived
    fi
fi
exit 0  # 检查成功
```

起服务

```shell
systemctl start keepalived
systemctl enable keepalived
```

