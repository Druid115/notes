### MySQL 常用语句

```mysql
# 用另外一张表中的某一字段更新目标表中的字段
UPDATE `sys_user` a LEFT JOIN `sys_dept` b ON a.SYS_DEPT = b.DEPT_NAME SET a.SYS_DEPT = b.ID
```

