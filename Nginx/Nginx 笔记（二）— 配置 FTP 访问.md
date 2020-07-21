## Nginx 笔记（二）— 配置 FTP 访问

安装好 FTP 后，要在浏览器或者富文本上通过 URL 显示服务器上的图片，ftp:// + IP 的方式是无法显示的，这样会直接下载文件。在链接中加上登录信息可以起作用，但安全性低且 Chrome 不支持这种方式，这时需要配置 Nginx 使用反向代理解决这些问题。

~~~shell
cd /usr/local/nginx/conf/

# 复制配置文件
cp nginx.conf nginx_ftp.conf

# 配置如下
# 加上 FTP 配置的用户名
user slj;

http {
    server {
        listen       80;
        server_name  IP地址;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   FTP 配置的文件地址;
            index  index.html index.htm;
        }
        
        # 希望当访问 IP/file/* 的时候，请求转发到 FTP 文件地址，此时可以如下配置：
        location /file/ {
            root   /;
	        rewrite ^/file/(.*)$ FTP 配置的文件地址/$1 break;
        }
}
# 启动（这里指定了配置文件）
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx_ftp.conf
~~~

配置完成后开放对应的端口，访问路径：ip + nginx_ftp 端口 + 文件目录 + 文件名。
