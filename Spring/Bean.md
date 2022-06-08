## Bean

### Bean 概念

由 Spring 容器管理的对象统称为 bean 对象，普通的 Java 对象。



### 创建 Bean 实例对象

Spring 容器内部创建 Bean 实例对象常见的有 4 种方式：

1. 通过反射调用构造方法创建
2. 通过静态工厂方法创建
3. 通过实例工厂方法创建
4. 通过 FactoryBean 创建
   - 实现 FactoryBean 接口



### Bean 的作用域

- **singleton**

  整个 Spring 容器中只会存在一个 bean 实例，singleton 是 scope 的默认值。**通常 Spring 容器在启动的时候，会将 scope 为singleton 的 bean 创建好放在容器中（当 bean 的 lazy 被设置为 true 的时候，表示懒加载，使用的时候才会创建）**，用的时候直接返回。 

- **prototype**

  表示这个 bean 是多例的，只有在**每次获取的时候才会重新创建 bean实例**，且每次获取的 bean 都是不同的实例。

- **request**

  表示在一次 http 请求中，一个 bean 对应一个实例。对每个 http 请求都会创建一个 bean 实例，request 结束的时候，这个 bean 的生命周期也就结束了。

- **session**

  session 级别共享的 bean，每个会话会对应一个 bean 实例，不同的 session 对应不同的 bean 实例。

- **application**

  一个 web 应用程序对应一个 bean 实例。通常情况下和 singleton 效果类似，不过也有不一样的地方，singleton 是每个 Spring 容器中只有一个 bean 实例，一般只有一个 Spring 容器，但一个应用程序中可以创建多个 Spring 容器，不同的容器中可以存在同名的 bean。但是当 bean 的 sope 为 application 时，不管应用中有多少个 Spring 容器，这个应用中同名的 bean 只有一个。

- **自定义 scope**

  实现 Scope 接口，将实现类注册到 Spring 容器，使用自定义的 sope。

~~~java
/**
 * @ClassName ThreadScope
 * @Description 自定义本地线程级别的 scope
 * @Author Ding RD
 * @Date 2021/3/10 16:51
 */
public class ThreadScope implements Scope {

    private static final ThreadLocal<Map<String, Object>> THREAD_SCOPE = new NamedThreadLocal<Map<String, Object>>("SimpleThreadScope") {
        @Override
        protected Map<String, Object> initialValue() {
            return new HashMap(16);
        }
    };

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        Map<String, Object> scopeMap = THREAD_SCOPE.get();
        Object bean = scopeMap.get(name);
        if (Objects.isNull(bean)) {
            bean = objectFactory.getObject();
            scopeMap.put(name, bean);
        }
        return bean;
    }

    @Override
    public Object remove(String name) {
        return THREAD_SCOPE.get().remove(name);
    }

    @Override
    public void registerDestructionCallback(String s, Runnable runnable) {
        // bean 作用域范围结束的时候调用的方法
    }

    @Nullable
    @Override
    public Object resolveContextualObject(String s) {
        return null;
    }

    @Nullable
    @Override
    public String getConversationId() {
        return null;
    }
}



/**
 * @ClassName BeanScopeConfig
 * @Description Bean scope 配置类
 * @Author Ding RD
 * @Date 2021/3/10 17:08
 */
@Configuration
public class BeanScopeConfig {

    @Bean
    public CustomScopeConfigurer customScopeConfigurer() {
        CustomScopeConfigurer customScopeConfigurer = new CustomScopeConfigurer();
        Map<String, Object> map = new HashMap<>(1);
        map.put("threadScope", new ThreadScope());
        customScopeConfigurer.setScopes(map);
        return customScopeConfigurer;
    }
}



/**
 * @ClassName MyScope
 * @Description 自定义 scope 测试类
 * @Author Ding RD
 * @Date 2021/3/10 17:10
 */
@Component
@Scope("threadScope")
public class MyScope {

    public MyScope() {
        System.out.println("创建 MyScope 实例");
    }
}



/**
 * @ClassName DemoController
 * @Description
 * @Author Ding RD
 * @Date 2021/2/1 16:22
 */
@Controller
public class DemoController {

    ApplicationContext applicationContext;

    @Autowired
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @GetMapping("/scopeTest")
    public void scopeTest() {

        ExecutorService executorService = Executors.newFixedThreadPool(2);

        executorService.submit(() -> {
            System.out.println(Thread.currentThread().getName() + " " + applicationContext.getBean("myScope"));
            System.out.println(Thread.currentThread().getName() + " " + applicationContext.getBean("myScope"));
        });

        executorService.submit(() -> {
            System.out.println(Thread.currentThread().getName() + " " + applicationContext.getBean("myScope"));
            System.out.println(Thread.currentThread().getName() + " " + applicationContext.getBean("myScope"));
        });
    }
}
~~~



### 单例 bean 中使用多例 bean

~~~java
public class ServiceA {
}

public class ServiceB {
    private ServiceA serviceA;
    
    public ServiceA getServiceA() {
        return serviceA;    
    }
    public void setServiceA(ServiceA serviceA) {
        this.serviceA = serviceA;    
    }
}
~~~

当 bean 中存在依赖关系时，被依赖的 bean 在当前 bean 中自始至终都是同一个实例。如果需要每次获取不同的被依赖的 bean 的实例，可以在 serviceB 中加个方法去获取 Spring 容器，再从容器中获取 ServiceA，每次获取到的都是不同的 ServiceA 实例。

~~~java
public class ServiceB implements ApplicationContextAware {
    @Autowired
    private ApplicationContext context;
    
    public void say() {        
        ServiceA serviceA = this.getServiceA();        
    }
    
    public ServiceA getServiceA() {        
        return this.context.getBean(ServiceA.class);
    }
}
~~~

也可以使用 Lookup 注解。

~~~java
public class ServiceB implements ApplicationContextAware {
    @Lookup
    public abstract ServiceA getServiceA();
    
    public void say(){        
        ServiceA serviceA = getServiceA();        
    }
}
~~~