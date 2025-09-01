### 3.1 基础环境准备

#### 3.1.1 Docker 环境安装

1. **上传安装包**
   将 `docker-27.3.1-x86.tgz` 上传至 `/opt/` 目录。

2. **解压安装包**

   ```bash
   tar -xvf /opt/docker-27.3.1-x86.tgz
   ```

3. **移动 Docker 文件到系统可执行目录**

   ```bash
   mv docker/* /usr/local/bin/
   ```

4. **创建 Docker 配置文件目录**

   ```bash
   mkdir /etc/docker/
   ```

5. **添加 Docker 配置文件（daemon.json）**

   ```bash
   cat > /etc/docker/daemon.json <<EOF
   {
       "storage-driver": "overlay2",
       "max-concurrent-downloads": 20,
       "live-restore": true,
       "max-concurrent-uploads": 10,
       "debug": true,
       "log-opts": {
           "max-size": "100m",
           "max-file": "5"
       },
       "bip": "11.16.0.1/16"
   }
   EOF
   ```

6. **创建 Docker 服务文件（docker.service）**

   ```bash
   cat > /etc/systemd/system/docker.service <<EOF
   [Unit]
   Description=Docker Application Container Engine
   Documentation=https://docs.docker.com
   After=network-online.target firewalld.service
   Wants=network-online.target
   
   [Service]
   Type=notify
   ExecStart=/usr/local/bin/dockerd
   ExecReload=/bin/kill -s HUP
   LimitNOFILE=infinity
   LimitNPROC=infinity
   LimitCORE=infinity
   TimeoutStartSec=0
   Delegate=yes
   KillMode=process
   Restart=on-failure
   StartLimitBurst=3
   StartLimitInterval=60s
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

7. **启动 Docker 服务并设置开机自启**

   ```bash
   systemctl enable --now docker
   ```

8. **验证 Docker 状态与版本**

   ```bash
   # 查看服务状态
   systemctl status docker
   # 查看版本
   docker --version 或 docker info
   ```

#### 3.1.2 Docker Compose 环境安装

1. **上传安装包**
   将 `docker-compose-linux_2.35.1-x86_64` 上传至 `/opt/` 目录。

2. **移动文件到系统可执行目录**

   ```bash
   mv /opt/docker-compose-linux_2.35.1-x86_64 /usr/local/bin/docker-compose
   ```

3. **添加执行权限**

   ```bash
   chmod +x /usr/local/bin/docker-compose
   ```

4. **验证安装**

   ```bash
   docker-compose --version
   ```

#### 3.1.3 防火墙与 SELinux 配置

1. **防火墙操作**

   ```bash
   # 查看防火墙状态
   systemctl status firewalld
   # 关闭防火墙
   systemctl stop firewalld
   # 禁用防火墙开机自启
   systemctl disable firewalld
   ```

   > 注：默认关闭防火墙；若需开启，需额外配置中间件端口的访问策略。

2. **SELinux 配置（设置为 disabled）**

   ```bash
   # 临时关闭
   setenforce 0
   # 永久关闭（编辑配置文件）
   vim /etc/selinux/config
   # 修改为：SELINUX=disabled
   ```

