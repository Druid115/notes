##  Jeecg-Boot前端笔记

### 前台全局配置文件
文件位置：src/public/index.html

内容：后台域名、图片服务器域名配置



### 路由

文件位置：src/router/index.js和/src/config/router.config

内容：通过不同的 URL 访问不同的内容

其中在配置文件中配置了基础路由constantRouterMap，指定了一些路径映射关系



### permission

文件位置：src/permission.js

内容：免登录白名单、路由前权限验证等配置



### store

### user.js

文件位置：src/store/modules

内容：账号密码登录、手机号登录







项目加载的过程是index.html->main.js->app.vue->index.js->helloworld.vue。