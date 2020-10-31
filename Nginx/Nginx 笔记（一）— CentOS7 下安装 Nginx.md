## Nginx 笔记（一）— CentOS7 下安装 Nginx

### 1. 安装所需环境

~~~shell
# 安装 nginx 需要先将官网下载的源码进行编译，编译依赖 gcc 环境，
yum install gcc-c++

# PCRE(Perl Compatible Regular Expressions) 是一个 Perl 库，包括 Perl 兼容的正则表达式库。nginx 的 http 模块使用 pcre 来解析正则表达式，所以需要安装 pcre 库。pcre-devel 是使用 pcre 开发的一个二次开发库。
yum install -y pcre pcre-devel

# zlib 库提供了很多种压缩和解压缩的方式，nginx 使用 zlib 对 http 包的内容进行 gzip
yum install -y zlib zlib-devel

# OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及 SSL 协议，并提供丰富的应用程序供测试或其它目的使用。nginx 不仅支持 http 协议，还支持 https（即在 SSL 协议上传输 http），所以需要安装 OpenSSL 库。
yum install -y openssl openssl-devel
~~~



### 2. 下载

直接下载 .tar.gz 安装包，地址：https://nginx.org/en/download.html

~~~shell
wget -c https://nginx.org/download/nginx-1.16.1.tar.gz
~~~



### 3. 安装

~~~shell
tar -zxvf nginx-1.16.1.tar.gz

cd nginx-1.16.1

# 使用默认配置
./configure

# 编译安装
make
make install

# 查找安装路径
whereis nginx
~~~



### 4. 启动

~~~shell
cd /usr/local/nginx/sbin/
./nginx 

# 此方式相当于先查出 nginx 进程 id 再使用 kill 命令强制杀掉进程。
./nginx -s stop
# 此方式是待 nginx 进程处理任务完毕后停止
./nginx -s quit

# 重新加载配置文件
./nginx -s reload

# 查看进程
ps aux|grep nginx
~~~



### 5. 设置开机自启动

~~~shell
# 创建 nginx.service 文件
cd /lib/systemd/system/

# nginx.service 内容如下
# 服务说明
[Unit]
# 描述服务
Description=nginx service
# 描述服务类别
After=network.target 

# 服务运行参数的设置
[Service] 
# 后台运行的形式
Type=forking 
# 服务的具体运行命令
ExecStart=/usr/local/nginx/sbin/nginx
# 重启命令
ExecReload=/usr/local/nginx/sbin/nginx -s reload
# 停止命令
ExecStop=/usr/local/nginx/sbin/nginx -s quit
# 给服务分配独立的临时空间
PrivateTmp=true 

# 运行级别下服务安装的相关设置
[Install] 
WantedBy=multi-user.target

# 加入开机自启动
systemctl enable nginx

# 取消
systemctl disable nginx
~~~
