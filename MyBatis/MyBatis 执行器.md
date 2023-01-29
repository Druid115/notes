## MyBatis 执行器

### BaseExecutor

执行器抽象类。



### 简单执行器 SimpleExecutor

无论 SQL 是否一样，每次都会创建一个新的预处理器 PrepareStatement 进行预编译。



### 可重用执行器 ReuseExecutor

相同的 SQL 只进行一次预处理。



### 批处理执行器 BatchExecutor

批量执行修改操作后，必须执行 `flushStatement()` 方法才会生效。

