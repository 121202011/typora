### Nginx 编译安装（含第三方模块）

| 服务名                      | 下载地址                                                     | 备注               |
| --------------------------- | ------------------------------------------------------------ | ------------------ |
| nginx                       | https://github.com/nginx/nginx/releases                      |                    |
| nginx-module-vts            | https://github.com/vozlt/nginx-module-vts/releases           | 兼容性要注意       |
| nginx_upstream_check_module | https://github.com/yaoweibin/nginx_upstream_check_module/tags | 最后一次更新2022年 |

#### 一、准备工作

```bash
# 进入源码目录
cd /usr/local/src/
```

#### 二、解压源码包

```bash
# 解压 Nginx 主程序源码
tar zxvf nginx-1.22.1.tar.gz

# 解压 vts 监控模块（用于流量统计）
tar zxvf nginx-module-vts-0.2.2.tar.gz

# 解压 upstream 健康检查模块
tar zxvf nginx_upstream_check_module-0.4.0.tar.gz
```

#### 三、安装依赖包

```bash
# 安装编译所需的依赖库
apt-get install libpcre3-dev libssl-dev libpcre2-dev -y
```

#### 四、创建专用用户

```bash
# 创建 nginx 系统用户（无登录权限）
useradd -r nginx
```

#### 五、应用模块补丁

```bash
# 进入 Nginx 源码目录
cd nginx-1.22.1/

# 为 upstream 健康检查模块打补丁（适配 Nginx 1.20.1+ 版本）
patch -p1 < /usr/local/src/nginx_upstream_check_module-0.4.0/check_1.20.1+.patch
```

#### 六、配置编译参数

```bash
./configure \
--prefix=/etc/nginx \                   # 安装目录
--sbin-path=/usr/sbin/nginx \           # 可执行文件路径
--modules-path=/usr/lib/nginx/modules \ # 模块存放路径
--conf-path=/etc/nginx/nginx.conf \     # 主配置文件路径
--error-log-path=/var/log/nginx/error.log \  # 错误日志路径
--http-log-path=/var/log/nginx/access.log \  # 访问日志路径
--pid-path=/var/run/nginx.pid \         # PID 文件路径
--lock-path=/var/run/nginx.lock \       # 锁文件路径
--http-client-body-temp-path=/var/cache/nginx/client_temp \  # 客户端临时文件路径
--http-proxy-temp-path=/var/cache/nginx/proxy_temp \          # 代理临时文件路径
--http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \      # FastCGI 临时文件路径
--http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \          # uWSGI 临时文件路径
--http-scgi-temp-path=/var/cache/nginx/scgi_temp \            # SCGI 临时文件路径
--user=nginx \                          # 运行用户
--group=nginx \                         # 运行组
--with-compat \                         # 兼容动态模块
--with-file-aio \                       # 启用文件异步 I/O
--with-threads \                        # 启用线程支持
--with-http_addition_module \           # 启用 HTTP 内容添加模块
--with-http_auth_request_module \       # 启用 HTTP 认证请求模块
--with-http_dav_module \                # 启用 WebDAV 模块
--with-http_flv_module \                # 启用 FLV 流媒体模块
--with-http_gunzip_module \             # 启用 gunzip 压缩模块
--with-http_gzip_static_module \        # 启用静态 gzip 压缩模块
--with-http_mp4_module \                # 启用 MP4 流媒体模块
--with-http_random_index_module \       # 启用随机索引模块
--with-http_realip_module \             # 启用真实 IP 模块
--with-http_secure_link_module \        # 启用安全链接模块
--with-http_slice_module \              # 启用切片模块
--with-http_ssl_module \                # 启用 SSL 模块
--with-http_stub_status_module \        # 启用状态监控模块
--with-http_sub_module \                # 启用内容替换模块
--with-http_v2_module \                 # 启用 HTTP/2 模块
--with-mail \                           # 启用邮件代理模块
--with-mail_ssl_module \                # 启用邮件 SSL 模块
--with-stream \                         # 启用流代理模块
--with-stream_realip_module \           # 启用流代理真实 IP 模块
--with-stream_ssl_module \              # 启用流代理 SSL 模块
--with-stream_ssl_preread_module \      # 启用流代理 SSL 预读模块
--with-cc-opt='-g -O2 -fdebug-prefix-map=/data/builder/debuild/nginx-1.19.6/debian/debuild-base/nginx-1.19.6=. -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' \
--with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie' \
--add-module=../nginx_upstream_check_module-0.4.0 \  # 添加 upstream 健康检查模块
--add-module=../nginx-module-vts-0.2.2              # 添加 vts 监控模块
```

#### 七、创建缓存目录并授权

```bash
# 创建各类临时缓存目录
mkdir -p /var/cache/nginx/{client_temp,fastcgi_temp,proxy_temp,scgi_temp,uwsgi_temp}

# 设置目录权限（nginx 用户可读写）
chown -R nginx.nginx /var/cache/nginx

# 创建日志目录
mkdir -p /var/log/nginx/

# 设置日志目录权限
chown -R nginx.nginx /var/log/nginx
```

#### 八、编译并安装

```bash
# 编译（多核心可加 -j 参数加速，如 make -j 4）
make && make install
```

#### 九、配置系统服务

```bash
# 创建 nginx 服务配置文件
cat > /lib/systemd/system/nginx.service <<EOF
[Unit]
Description=nginx - high performance web server  # 服务描述
Documentation=http://nginx.org/en/docs/          # 文档地址
After=network-online.target remote-fs.target nss-lookup.target  # 启动依赖
Wants=network-online.target

[Service]
Type=forking                               # 后台运行模式
PIDFile=/var/run/nginx.pid                 # PID 文件路径
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf  # 启动命令
ExecReload=/bin/sh -c "/bin/kill -s HUP \$(/bin/cat /var/run/nginx.pid)"  # 重载配置
ExecStop=/bin/sh -c "/bin/kill -s TERM \$(/bin/cat /var/run/nginx.pid)"   # 停止命令

[Install]
WantedBy=multi-user.target  # 多用户模式下启动
EOF
```

#### 十、启动 Nginx 服务

```bash
systemctl start nginx
```

#### 十一、Nginx 日志轮转配置文件

```bash
# 使用 cat 命令生成 Nginx 日志轮转配置文件（路径：/etc/logrotate.d/nginx）
# 若文件已存在会覆盖，执行前可先通过 cat /etc/logrotate.d/nginx 查看原文件内容
cat > /etc/logrotate.d/nginx << 'EOF'
# 1. 轮转 /var/log/nginx/ 目录下所有以 .log 结尾的日志（如 access.log、error.log 主日志）
/var/log/nginx/*log {
    create 0664 nginx root  # 轮转后新建日志文件：权限0664，属主nginx，属组root
    daily                   # 轮转频率：每天一次
    rotate 10               # 日志保留份数：最多保留10份历史日志
    missingok               # 若日志文件不存在，忽略错误（不中断轮转）
    notifempty              # 若日志文件为空，不执行轮转
    compress                # 对历史日志进行压缩（默认gzip）
    dateext                 # 日志文件名添加日期后缀（如 access.log-20240901）
    sharedscripts           # 所有匹配日志轮转完成后，仅执行一次后续脚本（避免重复操作）
    postrotate              # 轮转后执行的脚本（通知Nginx重新生成日志）
        /bin/kill -USR1 $(cat /run/nginx.pid 2>/dev/null) 2>/dev/null || true
    endscript               # 脚本结束标记
}

# 2. 轮转 /var/log/nginx/access_log/ 目录下的所有子日志（如按业务拆分的访问日志）
/var/log/nginx/access_log/*.log {
    create 0664 nginx root
    daily
    rotate 10
    missingok
    notifempty
    compress
    dateext
    sharedscripts
    postrotate
        /bin/kill -USR1 $(cat /run/nginx.pid 2>/dev/null) 2>/dev/null || true
    endscript
}

# 3. 轮转 /var/log/nginx/error_log/ 目录下的所有子日志（如按业务拆分的错误日志）
/var/log/nginx/error_log/*.log {
    create 0664 nginx root
    daily
    rotate 10
    missingok
    notifempty
    compress
    # 注：此处未配置 dateext，历史日志将以 .1、.2.gz 等后缀命名（而非日期）
    sharedscripts
    postrotate
        /bin/kill -USR1 $(cat /run/nginx.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
EOF
```