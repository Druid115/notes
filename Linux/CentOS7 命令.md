## CentOS7 命令

- 查看CentOS版本

~~~shell
cat /etc/redhat-release
~~~



- 防火墙配置

~~~shell
# Add
firewall-cmd --permanent --zone=public --add-port=80/tcp

# Remove
firewall-cmd --permanent --zone=public --remove-port=80/tcp

# Reload
firewall-cmd --reload

# 查看防火墙状态
systemctl status firewalld.service

# 启动防火墙
systemctl start firewalld.service

# 关闭防火墙
systemctl stop firewalld.service
~~~



- 更改主机名

~~~shell
vi /etc/hostname 
~~~



- 修改网络

~~~shell
vi /etc/sysconfig/network-scripts/ifcfg-ens32 

IPADDR=192.168.100.20
NETMASK=255.255.255.0
GATEWAY=192.168.100.1
DNS1=210.22.84.3

systemctl restart network
~~~



- 文件操作

~~~shell
# 移动文件
mv 文件名 移动目的地文件名

# 重命名
mv 文件名 修改后的文件名

# 删除文件
rm 文件名

# 不能删除非空的文件夹
rmdir 文件夹名 

# -r 就是向下递归，不管有多少级目录，一并删除；-f 就是直接强行删除，不作任何提示
rm -rf 非空文件夹名
~~~



- 内存占用

~~~shell
# 监控 linux 的系统状况，比如 cpu、内存的使用
top

# 查看内存占用
free -m

# 看硬盘占用率
df -h

# 查看各文件夹大小
du -h -x --max-depth=1
~~~



- 查看开机启动服务

~~~shell
chkconfig –list
~~~



- 修改时间

~~~shell
# 查看当前系统时间
date

# 修改当前系统时间 
date -s "2024-12-10 11:10:30"

# 查看硬件时间
hwclock --show

# 修改硬件时间
hwclock --set --date "2024-12-10 11:10:30"

# 同步系统时间和硬件时间
hwclock --hctosys

# 修改时区
mv /etc/localtime /etc/localtime.bak
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# 保存时钟
clock -w
~~~



