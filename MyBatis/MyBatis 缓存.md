## MyBatis 缓存

### 一级缓存

缓存命中场景：

- 运行时参数相关
  - 同一个会话
  - SQL 和参数相同
  - 相同的 StatementId
  - RowBounds 相同

- 操作与配置相关
  - 未手动清空缓存（提交、回滚）
  - 未配置 flushCache = true
  - 未执行 Update 语句
  - 缓存作用域不是 STATEMENT

子查询依赖了一级缓存，不会清空。

Spring 中 mapper -> SqlSessionTemplate -> SqlSessionInterceptor -> SqlSessionFactory



### 二级缓存

二级缓存也被称作应用级缓存，它的作用范围是整个应用，而且可以跨线程使用。所以二级缓存拥有更高的命中率，适合缓存一些修改较少的数据。

使用了装饰者和责任链模式。

缓存命中场景：

- 运行时参数相关
  - 会话提交后（**手动提交**）
  - SQL 和参数相同
  - 相同的 StatementId
  - RowBounds 相同