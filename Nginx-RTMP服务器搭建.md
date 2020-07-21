## Nginx-RTMP服务器搭建

### 1. 安装所需环境

~~~shell
yum install gcc-c++

yum install -y pcre pcre-devel

yum install -y zlib zlib-devel

yum install -y openssl openssl-devel

# 备份yum源
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup 
# 下载阿里源 
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo 
# 清空缓存 
yum makecache
~~~



### 2. 下载

- 可以直接下载 .tar.gz 安装包，地址：https://nginx.org/en/download.html

~~~shell
wget -c https://nginx.org/download/nginx-1.16.1.tar.gz

# 下载rtmp包
wget https://github.com/arut/nginx-rtmp-module/archive/master.zip
# 解压下载包
unzip -o master.zip
# 修改文件夹名
mv master nginx-rtmp-module
~~~



### 3. 安装 Nginx

~~~shell
tar -zxvf nginx-1.16.1.tar.gz

cd nginx-1.16.1

# 切换到nginx中 
cd nginx-1.16.1
# 生成配置文件，将上述下载的文件配置到configure中 
./configure --prefix=/usr/local/nginx --add-module=/nginx-rtmp-module 
# 编译程序 
make 
# 安装程序 
make install 
# 查看nginx模块 
nginx -V

# 查找安装路径
whereis nginx
~~~



### 4. 安装 FFmpeg

```shell
# 安装epel包
yum install -y epel-release 
# 导入签名 
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 
# 导入签名 
rpm --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro 
# 升级软件包 
rpm -Uvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-1.el7.nux.noarch.rpm 
# 更新软件包 
yum update -y 
# 安装ffmpeg 
yum install -y ffmpeg 
# 检查版本 
ffmpeg -version
```



### 5. 修改 Nginx 配置

~~~shell
http {
}

rtmp {
      server {
              listen 1935; 
              publish_time_fix on;
              application myapp {
                      live on; #stream on live allow
                      allow publish all; # control access privilege
                      allow play all; # control access privilege
              }
      }
}
~~~



### 6. 启动 Nginx

~~~shell
cd /usr/local/nginx/sbin/
./nginx 

# 此方式相当于先查出nginx进程id再使用kill命令强制杀掉进程。
./nginx -s stop
# 此方式是待nginx进程处理任务完毕后停止
./nginx -s quit

# 重新加载配置文件：
./nginx -s reload

# 查看进程
ps aux|grep nginx
~~~



### 7. 设置 Nginx 开机自启动

~~~shell
# 创建nginx.service文件
cd /lib/systemd/system/

# nginx.service内容如下
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

