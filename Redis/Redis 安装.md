## Redis 安装

### 安装 Redis

~~~shell
# 下载安装最新版的 gcc 编译器
yum install centos-release-scl scl-utils-build
yum install -y devtoolset-8-toolchain
scl enable devtoolset-8 bash
# 测试 gcc 版本
gcc --version

# 下载 Redis
wget http://download.redis.io/releases/redis-6.2.1.tar.gz
# 移动到 /usr/local 目录下并解压
tar xzf redis-6.2.1.tar.gz
cd redis-6.2.1
make
~~~

若编译的过程报错 jemalloc/jemalloc.h: No such file or directory，原因是 jemalloc 重载了 Linux 下的 ANSI C 的 malloc 和 free 函数，解决办法就是 make 时添加参数。

```shell
make MALLOC=libc
```



### 配置

Redis 的配置文件 redis.conf 在解压路径的根目录下，需要修改几个地方。

~~~shell
# bind  127.0.0.1 代表本地回环地址，访问 Redis 服务只能通过本机的客户端连接，而无法通过远程连接
bind  0.0.0.0

# 保护模式，默认只允许本地链接
protected-mode no

# 代表开启守护进程模式。此时是单进程多线程的模式，Redis 将在后台运行，并将 pid 写入 redis.conf--pidfile 文件中。默认为 no，当前界面将进入 Redis 的命令行界面，exit 强制退出或者关闭连接工具（xshell 等）都会导致 Redis 进程退出
daemonize yes

# 数据淘汰策略
# volatile-lru：从设置了过期时间的数据集中，选择最近最少使用的数据释放；
# allkeys-lru：从数据集中（所有数据），选择最近最少使用的数据释放；
# volatile-random：从设置了过期时间的数据集中，随机选择一个数据进行释放；
# allkeys-random：从数据集中（所有数据）随机选择一个数据进行释放；
# volatile-ttl：从设置了过期时间的数据集中，选择马上就要过期的数据释放；
# noeviction：不删除任意数据（但 Redis 还会根据引用计数器进行释放）,这时如果内存不够时，会直接返回错误
maxmemory-policy noeviction
~~~

持久化配置项：

~~~shell
# RDB 文件名称
dbfilename dump.rdb

# RDB 文件保存路径，默认为 Redis 启动时命令行所在的目录下
dir ./

# 快照策略，save seconds nums 表示在 seconds 时间内有 nums 个 key 发生改变时会触发 RDB，给 save 传入空字符串会禁用
save 3600 1
save 30 10
save 60 10000

# 当 Redis 无法写入磁盘时，直接关掉 Redis 的写操作
stop-writes-on-bgsave-error yes

# 压缩 RDB 文件，Redis 会采用 LZF 算法进行压缩，会消耗一定性能
rdbcompression yes

# 检查 RDB 文件的完整性，会消耗一定性能
rdbchecksum yes



# AOF 默认关闭
appendonly no

# AOF 文件名称
appendfilename "appendonly.aof"


# 缓冲区写入磁盘的频率 always 每次写入都会立即同步；everysec 每秒同步；no 由操作系统决定合适的时间来同步磁盘
appendfsync everysec

# 为了尽量防止 Redis 在某些情况下，在 fsync 方法上阻塞时间过久，可以使用下面的选项。yes 为不写入 AOF 文件只写入缓存，用户请求不会阻塞，但是在这段时间如果宕机会丢失缓存数据。no 会把数据往磁盘里刷，但是遇到重写操作，可能会发生阻塞
no-appendfsync-on-rewrite no

# AOF 日志重写的时机，系统载入时或上一次重写完毕时 AOF 文件大小记为 base_Size，当前 AOF 文件大小 >= base_size * (1 + percentage) 且大于 min-size 的情况下，Redis 会自动进行重写
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 混合持久化
aof-use-rdb-preamble yes
~~~

其他配置项：

~~~shell
########## 网络相关配置 ##########

# backlog 是一个连接队列，队列总和 = 未完成三次握手队列 + 已经完成三次握手队列。在高并发环境下需要一个高 backlog 值来避免慢客户端连接问题。注意 Linux 内核会将这个值减小到 /proc/sys/net/core/somaxconn 的值（128），所以需要确认增大 /proc/sys/net/core/somaxconn 和 /proc/sys/net/ipv4/tcp_max_syn_backlog（128）两个值来达到想要的效果
tcp-backlog 511

# 一个空闲的客户端维持多少秒会关闭，0 表示永不关闭
timeout 0

# 对访问客户端每隔 n 秒进行心跳检测，0 表示不进行检测，建议设置成 60
tcp-keepalive 300

########## 通用配置 ##########

# 存放 pid 文件的位置，每个实例会产生一个不同的 pid 文件
pidfile /var/run/redis_6379.pid

# 日志级别：debug、verbose、notice、warning
loglevel notice

# 日志文件名称
logfile ""

# 设定库的数量
database 16

########## 安全配置 ##########

# 设置访问密码
requirepass foobared

########## Limits 配置 ##########

# 最大客户端连接数量
maxclients 10000

# 最大占用内存，建议必须设置，否则内存占满会造成服务器宕机。一旦到达内存使用上限，Redis 将会根据移除规则移除内部数据。如果 Redis 无法根据移除规则来移除数据，或者设置了不允许移除，那么 Redis 会对那些需要申请内存的指令返回错误信息，如 SET、LPUSH 等。对于无内存申请的指令，仍然会正常响应，如 GET 等。如果 Redis 是主节点（主从模式），那么在设置内存使用上限时，需要在系统中留出一些内存空间给同步队列缓存。只有在设置的是不移除的情况下，才不用考虑这个因素。
maxmemory <bytes>

# 样本数量，LRU 算法和最小 TTL 算法并非是精确的算法，而是估算值。可以设置样本的大小，Redis 默认会检查这么多个 key 并选择其中LRU 的那些。一般设置 3 到 7 的数字，数值越小样本越不准确，但性能消耗越小。
maxmemory-samples 5
~~~



### 启动

~~~shell
# 开放端口
firewall-cmd --zone=public --permanent --add-port=6379/tcp
firewall-cmd --reload

cd /usr/local/redis-6.2.1/src
# 这种方式使用的是默认配置
./redis-server

# 指定配置文件
./redis-server ../redis.conf
~~~



### 设置开机自启动

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
ExecStart=/usr/local/redis-6.2.1/src/redis-server /usr/local/redis-6.2.1/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
 
[Install]
WantedBy=multi-user.target

# 开启服务
systemctl enable redis-server.service
~~~