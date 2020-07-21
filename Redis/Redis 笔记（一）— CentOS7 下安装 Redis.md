## Redis 笔记（一）— CentOS7 下安装 Redis

### 1. 安装 Redis

~~~shell
wget http://download.redis.io/releases/redis-5.0.5.tar.gz
# 移动到 /usr/local 目录下并解压
tar xzf redis-5.0.5.tar.gz
cd redis-5.0.5
make
~~~

若编译的过程报错 jemalloc/jemalloc.h: No such file or directory，原因是 jemalloc 重载了 Linux 下的 ANSI C 的 malloc 和 free 函数，解决办法就是 make 时添加参数。

```shell
make MALLOC=libc
```



### 2. 配置

Redis 的配置文件 redis.conf 在解压路径的根目录下，需要修改几个地方。

~~~shell
# bind  127.0.0.1 代表本地回环地址，访问 redis 服务只能通过本机的客户端连接，而无法通过远程连接
bind  0.0.0.0

# 保护模式，默认只允许本地链接
protected-mode no

# 代表开启守护进程模式。此时是单进程多线程的模式，redis 将在后台运行，并将 pid 写入 redis.conf--pidfile 文件中，此时 redis将一直运行，除非手动 kill。默认为 no，当前界面将进入 redis 的命令行界面，exit 强制退出或者关闭连接工具（xshell等）都会导致redis 进程退出
daemonize yes

# 数据淘汰策略
# volatile-lru:从设置了过期时间的数据集中，选择最近最久未使用的数据释放；
# allkeys-lru:从数据集中(包括设置过期时间以及未设置过期时间的数据集中)，选择最近最久未使用的数据释放；
# volatile-random:从设置了过期时间的数据集中，随机选择一个数据进行释放；
# allkeys-random:从数据集中(包括了设置过期时间以及未设置过期时间)随机选择一个数据进行入释放；
# volatile-ttl：从设置了过期时间的数据集中，选择马上就要过期的数据进行释放操作；
# noeviction：不删除任意数据(但 redis 还会根据引用计数器进行释放),这时如果内存不够时，会直接返回错误。
maxmemory-policy noeviction
~~~



### 3. 启动

~~~shell
# 开放端口
firewall-cmd --zone=public --permanent --add-port=6379/tcp
firewall-cmd --reload

cd /usr/local/redis-5.0.5/src
# 这种方式使用的是默认配置
./redis-server

# 指定配置文件
./redis-server ../redis.conf
~~~



### 4. 设置开机自启动

~~~shell
# 创建 redis-server.service 文件
cd /usr/lib/systemd/system

# redis-server.service 内容如下
[Unit]
Description=The redis-server Process Manager
After=syslog.target network.target
 
[Service]
Type=forking
PIDFile=/var/run/redis_6379.pid
ExecStart=/usr/local/redis-5.0.5/src/redis-server /usr/local/redis-5.0.5/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
 
[Install]
WantedBy=multi-user.target

# 开启服务
systemctl enable redis-server.service
~~~