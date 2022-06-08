### Java 代理

#### JDK 动态代理

~~~java
java.lang.reflect.Proxy;

/**
 * 为指定的接口创建代理类，返回代理类的 Class 对象
 *
 * loader：定义代理类的类加载器 
 * interfaces：指定需要实现的接口列表，创建的代理默认会按顺序实现 interfaces 指定的接口
 */
public static Class<?> getProxyClass(ClassLoader loader, Class<?>... interfaces);

/**
 * 创建代理类的实例对象
 *
 * h：当调用代理对象的任何方法的时候，会就被 InvocationHandler 接口的 invoke 方法处理
 */
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h);

/**
 * 判断指定的类是否是一个代理类
 */
public static boolean isProxyClass(Class<?> cl);

/**
 * 获取代理对象的 InvocationHandler 对象
 */
public static InvocationHandler getInvocationHandler(Object proxy) throws IllegalArgumentException;
~~~

~~~java
public interface IService {

    void m1();

    void m2();

    void m3();
}


@Slf4j
public class ProxyService {

    public static void main(String[] args) throws IllegalArgumentException, NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {

        // 创建代理类的处理器
        InvocationHandler invocationHandler = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                log.info("进入 IService 代理类，被调用的方法是：{}", method.getName());
                return null;
            }
        };

        // 创建代理实例
        IService proxyService = (IService) Proxy.newProxyInstance(IService.class.getClassLoader(), new Class[]{IService.class}, invocationHandler);
        
//        // 获取接口对应的代理类
//        Class<IService> proxyClass = (Class<IService>) Proxy.getProxyClass(IService.class.getClassLoader(), IService.class);
//        // 创建代理实例
//        IService proxyService = proxyClass.getConstructor(InvocationHandler.class).newInstance(invocationHandler);

        proxyService.m1();
        proxyService.m2();
        proxyService.m3();

    }
}
~~~

~~~java
@Slf4j
public class CostTimeInvocationHandler implements InvocationHandler {

    private Object target;

    public CostTimeInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Long startTime = System.currentTimeMillis();
        Object result = method.invoke(this.target, args);
        Long endTime = System.currentTimeMillis();
        log.info("{}.{} 方法执行耗时：{} 毫秒", this.target.getClass(), method.getName(), endTime - startTime);
        return result;
    }

    /**
     * 用来创建被代理接口的代理对象
     *
     * @param target          需要被代理的对象
     * @param targetInterface 被代理的接口
     * @param <T>
     * @return 代理对象
     */
    public static <T> T createProxy(Object target, Class<T> targetInterface) {
        if (!targetInterface.isInterface()) {
            throw new IllegalStateException("targetInterface 必须是接口类型!");
        } else if (!targetInterface.isAssignableFrom(target.getClass())) {
            throw new IllegalStateException("target 必须是 targetInterface 接口的实现类!");
        }
        return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(), new Class[]{targetInterface}, new CostTimeInvocationHandler(target));
    }
}
~~~

JDK 中的 Proxy 只能为接口生成代理类，如果你想给某个类创建代理类，需要用到 CGLIB。



#### CGLIB 代理

CGLIB 是一个强大、高性能的字节码生成库，它用于在运行时扩展 Java 类和实现接口；本质上它是通过动态的生成一个子类去覆盖所要代理的类（非 final 修饰的类和方法）。其被广泛应用于 AOP 框架（Spring、dynaop）中，用以提供方法拦截操作。

~~~java
@Slf4j
public class Service {

    public String m1() {
        this.m2();
        log.info("我是 m1 方法");
        return "Service : m1";
    }

    public String m2() {
        log.info("我是 m2 方法");
        return "Service : m2";
    }

    public String get() {
        log.info("我是 get 方法");
        return "Service : get";
    }

    public String insert() {
        log.info("我是 insert 方法");
        return "Service : insert";
    }
}


@Slf4j
public class CglibProxyService {

    public static void interceptor() {
        // 1. 创建 Enhancer 对象
        Enhancer enhancer = new Enhancer();
        // 2. 通过 setSuperclass 来设置父类型，即需要给哪个类创建代理类
        enhancer.setSuperclass(Service.class);
        // 3. 设置回调，需实现 org.springframework.cglib.proxy.Callback 接口，
        // 此处使用的是 org.springframework.cglib.proxy.MethodInterceptor，也是一个实现了 Callback 的接口，
        // 当调用代理对象的任何方法的时候，都会被 MethodInterceptor 接口的invoke方法处理
        /**
         * @param o 代理对象
         * @param method 被代理的类的方法
         * @param objects 调用方法传递的参数
         * @param methodProxy 方法代理对象
         */
        enhancer.setCallback((MethodInterceptor) (o, method, objects, methodProxy) -> {
            log.info("进入代理类，调用方法：{}", method.getName());
            // 可以调用 MethodProxy 的 invokeSuper 调用被代理类的方法
            return methodProxy.invokeSuper(o, objects);
        });

        Service proxyService = (Service) enhancer.create();
        System.out.println(proxyService.m1());
        System.out.println(proxyService.m2());
    }

    /**
     * 拦截所有方法并返回固定值
     */
    public static void fixedInterceptor() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Service.class);
        enhancer.setCallback((FixedValue) () -> "proxy");

        Service proxyService = (Service) enhancer.create();
        System.out.println(proxyService.m1());
        System.out.println(proxyService.m2());
    }

    /**
     * 直接放行，不做任何操作
     */
    public static void noInterceptor() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Service.class);
        enhancer.setCallback((NoOp.INSTANCE));

        Service proxyService = (Service) enhancer.create();
        System.out.println(proxyService.m1());
        System.out.println(proxyService.m2());
    }

    /**
     * 不同方法使用不同的拦截器
     */
    public static void twoInterceptors() {
        Enhancer enhancer = new Enhancer();

        Callback costTimeCallback = (MethodInterceptor) (o, method, objects, methodProxy) -> {
            Long startTime = System.currentTimeMillis();
            Object result = methodProxy.invokeSuper(o, objects);
            Long endTime = System.currentTimeMillis();
            log.info("{}.{} 方法执行耗时：{} 毫秒", o.getClass(), method.getName(), endTime - startTime);
            return result;
        };
        Callback fixedValueCallback = (FixedValue) () -> "proxy";

        Callback[] callbacks = {costTimeCallback, fixedValueCallback};

//        enhancer.setSuperclass(Service.class);
//        enhancer.setCallbacks(callbacks);
//        // 设置过滤器 CallbackFilter，CallbackFilter 用来判断调用方法的时候使用 callbacks 数组中的哪个 Callback 来处理当前方法
//        enhancer.setCallbackFilter(method -> method.getName().startsWith("insert") ? 0 : 1);

        CallbackHelper callbackHelper = new CallbackHelper(Service.class, null) {
            @Override
            protected Object getCallback(Method method) {
                return method.getName().startsWith("insert") ? costTimeCallback : fixedValueCallback;
            }
        };

        enhancer.setSuperclass(Service.class);
        enhancer.setCallbacks(callbackHelper.getCallbacks());
        enhancer.setCallbackFilter(callbackHelper);

        Service proxyService = (Service) enhancer.create();
        System.out.println(proxyService.get());
        System.out.println(proxyService.insert());
    }

    public static void main(String[] args) {
//        interceptor();
//        fixedInterceptor();
//        noInterceptor();
        twoInterceptors();
    }
}
~~~



#### CGLIB 和 JDK 动态代理的区别

1.  JDK 动态代理只能够对接口进行代理，不能对普通的类进行代理（因为所有生成的代理类的父类为 Proxy，Java 类继承机制不允许多重继承）；CGLIB 能够代理普通类。
2.  JDK 动态代理使用 Java 原生的反射 API 进行操作，在生成类上比较高效；CGLIB 使用 ASM 框架直接对字节码进行操作，在类的执行过程中比较高效。