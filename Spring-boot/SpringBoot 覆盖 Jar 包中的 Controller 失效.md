## SpringBoot 覆盖 Jar 包中的 Controller 失效

### 起因

业务组同事需要重写平台依赖包中的登录方法，但又不想单独提取方法到一个新的类中重写，使用了同名包同名类覆盖的方式重写了整个类。起初打包方式为全量包，在此情况下没有出现问题。之后改成了瘦身包的形式（lib 和程序分开打包），重写的方法并不能生效。

推测是先加载了 lib 包中的类，根据双亲委派机制，重写的类并不会手动加载。为了解决这个问题，需要手动将重写的类注册到 Spring 中。



### 解决方案

#### 使用 PostProcessor 

使用 BeanFactoryPostProcessor 在 Bean 的实例化之前替换 BeanDefinition。

~~~java
/**
 * @ClassName MyBeanDefinitionRegistryPostProcessor
 * @Description
 * @Author Ding RD
 * @Date 2023/1/9 9:36
 */
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
        if (beanDefinitionRegistry.containsBeanDefinition("loginController") && beanDefinitionRegistry.containsBeanDefinition("loginControllerOuter")) {
            beanDefinitionRegistry.removeBeanDefinition("loginController");
            beanDefinitionRegistry.registerBeanDefinition("loginController", beanDefinitionRegistry.getBeanDefinition("loginControllerOuter"));
            // 这个地方移除是为了避免请求路径和方法重复的错误
            beanDefinitionRegistry.removeBeanDefinition("loginControllerOuter");
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {

    }
}
~~~



#### 手动注册 Controller

在系统启动成功后，手动注册重写的 Controller，并更新请求路径和方法的映射关系。

~~~java
/**
 * @ClassName MyApplicationReadyEvent
 * @Description
 * @Author Ding RD
 * @Date 2023/1/9 14:09
 */
@Component
public class MyApplicationReadyEvent implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        ApplicationContext applicationContext = SpringUtils.getApplicationContext();
        RequestMappingHandlerMapping requestMappingHandlerMapping = applicationContext.getBean(RequestMappingHandlerMapping.class);
        Object controller = applicationContext.getBean("loginController");
        Class<?> targetClass = controller.getClass();
        // 移除原有的映射关系
        ReflectionUtils.doWithMethods(targetClass, method -> {
            Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
            try {
                Method createMappingMethod = RequestMappingHandlerMapping.class.
                        getDeclaredMethod("getMappingForMethod", Method.class, Class.class);
                createMappingMethod.setAccessible(true);
                RequestMappingInfo requestMappingInfo = (RequestMappingInfo)
                        createMappingMethod.invoke(requestMappingHandlerMapping, specificMethod, targetClass);
                if (requestMappingInfo != null) {
                    // 此处移除
                    requestMappingHandlerMapping.unregisterMapping(requestMappingInfo);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, ReflectionUtils.USER_DECLARED_METHODS);

        // 先手动注册 Controller 到 Spring 容器中
        ManualRegisterBeanUtil.registerBean((ConfigurableApplicationContext) SpringUtils.getApplicationContext(), "loginController", LoginControllerOuter.class);

        // 注册 Controller
        Method method = RequestMappingHandlerMapping.class.
                getSuperclass().getSuperclass().
                getDeclaredMethod("detectHandlerMethods", Object.class);

        // 将 private 改为可使用
        method.setAccessible(true);
        method.invoke(requestMappingHandlerMapping, "loginController");
    }
}



/**
 * @ClassName ManualRegisterBeanUtil
 * @Description 手动注入 bean 工具类
 * @Author Ding RD
 * @Date 2021/4/6 11:44
 */
@Slf4j
public class ManualRegisterBeanUtil {

    /**
     * 主动向Spring容器中注册bean
     *
     * @param applicationContext Spring容器
     * @param name               BeanName
     * @param clazz              注册的bean的类性
     * @param args               构造方法的必要参数，顺序和类型要求和clazz中定义的一致
     * @param <T>
     * @return 返回注册到容器中的bean对象
     */
    public static <T> T registerBean(ConfigurableApplicationContext applicationContext, String name, Class<T> clazz,
                                     Object... args) {
        BeanDefinitionRegistry beanFactory = (BeanDefinitionRegistry) applicationContext.getBeanFactory();

        if (applicationContext.containsBean(name)) {
            log.info("BeanName 重复，移除原有 bean");
            beanFactory.removeBeanDefinition(name);
        }

        BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(clazz);
        for (Object arg : args) {
            beanDefinitionBuilder.addConstructorArgValue(arg);
        }
        BeanDefinition beanDefinition = beanDefinitionBuilder.getRawBeanDefinition();
        beanFactory.registerBeanDefinition(name, beanDefinition);
        return applicationContext.getBean(name, clazz);
    }
}
~~~