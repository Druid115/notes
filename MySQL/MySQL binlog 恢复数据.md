### MySQL

create user 'slj-oa'@'%' identified by 'slj@1234';

grant create,insert,delete,update,select,ALTER,REFERENCES,drop,index on ruoyi.* to 'slj-oa'@'%';

grant select on `performance_schema`.user_variables_by_thread to 'slj-oa'@'%';

grant process on *.* to 'slj-robot'@'%';

管理员：root    密码：slj@123456

slj_boot库：slj-boot    密码：slj@1234

slj_oa库：slj-oa    密码：slj@1234

slj_robot库：slj-robot    密码：slj@1234



OA应用库：slj-oa root 密码：slj@123456

### Oracle

用户名：sys system





```
mysqlbinlog --no-defaults --database=slj_oa --start-datetime="2019-10-10 09:00:00" --stop-datetime="2019-10-11 11:00:00" -v /var/lib/mysql/binlog.000005   | grep t_slj_device  |more > /var/lib/mysql/test.txt
```

```
mysqlbinlog --no-defaults --database=slj_oa --start-datetime="2019-10-10 09:00:00" --stop-datetime="2019-10-11 11:00:00" /var/lib/mysql/binlog.000005   > /var/lib/mysql/test.txt
```

