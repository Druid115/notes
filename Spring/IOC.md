## IOC

### 通用职责

- 依赖处理
  - 依赖查找：主动或手动的依赖查找方式，通常需要依赖容器的 API 实现。
  - 依赖注入：手动或自动的依赖绑定方式，无需依赖特定的容器和 API。
- 生命周期管理
  - 容器
  - 托管的资源（Java Beans 或其他资源）
- 配置
  - 容器
  - 外部化配置
  - 托管的资源



### 依赖来源

- 自定义 Bean
- 容器默认创建的 Bean
- 容器构建的依赖



### BeanFactory 和 ApplicationContext 谁才是 IOC 容器

BeanFactory 是一个底层的 IOC 容器，ApplicationContext 是 BeanFactory 的一个子接口，并且提供了：

- 简化了与 Spring AOP 的整合（AOP 的特性）
- 资源的处理（用于国际化）
- 事件的发布
- 应用级别的上下文，例如 WebApplicationContext
- 配置元信息
- 注解



### BeanFactory 和 FactoryBean

BeanFactory 是一个底层的 IOC 容器，而 FactoryBean 是创建 Bean 的一种方式，帮助实现复杂的初始化逻辑
