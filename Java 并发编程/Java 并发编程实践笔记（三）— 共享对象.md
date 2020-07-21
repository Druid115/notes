## Java 并发编程实践笔记（三）— 共享对象

### 可见性

当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看到修改的值。



### Volatile 变量

volatile 确保对一个变量的更新以可预见的方式告知其他线程。当一个域声明为 volatile 类型后，编译器在运行时会监视这个变量：它是共享的，而且对它的操作不会与求他的内存操作一起被重排序。volatile 变量不会缓存在寄存器或者缓存在对其他处理器隐藏的地方。所以，读一个 volatile 类型的变量时，总会返回由某一线程所写入的最新值。

volatile 通过加入内存屏障和禁止指令重排序优化来实现线程安全的。

内存屏障的作用：
1.在有内存屏障的地方，会禁止指令重排序，即屏障下面的代码不能跟屏障上面的代码交换执行顺序。
2.在有内存屏障的地方，线程修改完共享变量以后会马上把该变量从本地内存写回到主内存，并且让其他线程本地内存中该变量副本失效（使用 MESI 协议）。

volatile 变量通常被当作标识完成、中断、状态的标记使用。只有满足以下所有标准后，才能使用 volatile 变量：

- 写入变量时并不依赖变量当前值；或者能够确保只有单一的线程修改变量的值；
- 变量不需要与其他的状态变量共同参与不变约束；
- 访问变量时，没有其他的原因需要加锁。

加锁能保证可见性与原子性，volatile 变量只能保证可见性。



### 发布和逸出

发布（publishing）一个对象的意思是使它能够被当前范围之外的代码所使用。

逸出（escape），一个对象在尚未准备好的时候就将它发布。

~~~java
/**
 * 内部状态可变的数据逸出，任何一个调用者都能修改数组 states 的内容，已经逸出了它所属的范围
 */
public class UnsafeStates {
    private String[] states = {"AK", "AL"};

    public String[] getStates() {
        return states;
    }
}


/**
 * 隐式地允许 this 引用逸出，当 ThisEscape 发布 EvenListener 时，它也无条件地发布了封装
 * （enclosing）ThisEscape 的实例，因为内部类的实例包含了对封装实例隐含的引用：
 * 内部类自动拥有对外部类所有成员的访问权，是通过秘密捕获一个指向那个外部类对象的引用实现的。
 * 假如 ThisEscape 类中有个 i 变量，那么在 EventListener 内部类中就可以使用 ThisEscape.this.i 来访问外部类的变量 i。
 */
public class ThisEscape {
    public ThisEscape(EventSource source) {
        source.registerListener(
                new EventListener() {
                    public void onEvent(Event e) {
                        doSomething(e);
                        // ThisEscape 类的 this 引用在构造方法没有完成时，就可以被别的方法使用
                        ThisEscape.this.getClass();
                    }
                }
        );
    }
}
~~~

从构造函数内部发布的对象，只是一个未完成构造的对象，即使是在构造函数的最后一行发布的引用也是如此。

~~~java
/**
 * 使用工厂方法防止 this 引用在构造期间逸出
 */
public class SafeListener {
    private final EventListener listener;

    private SafeListener() {
        listener = new EventListener() {
          public void onEvent(Event e){
              doSomething(e);
          }
        };
    }

    public static SafeListener newInstance(EventSource source) {
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
~~~



### 线程封闭

把对象封闭在一个线程中，这种做法是线程安全的，即使被封闭的对象本身并不是。

- Ad-hoc 线程封闭：程序控制实现，最糟糕。
- 堆栈封闭：局部变量，无并发问题。
- ThreadLocal 线程封闭。



### 不可变性

不可变对象永远是线程安全的。只有满足如下状态，一个对象才是不可变的：

- 它的状态不能在创建后再被修改；
- 所有的域都是 final 类型；
- 它被正确的创建（创建期间没有发生 this 引用的逸出）。

~~~java
/**
 * 通过不可变容器对象的 volatile 类型引用，缓存最新的结果
 */
class OneValueCahe {
    private final BigInteger lastNumber;
    private final BigInteger[] lastFactors;

    public OneValueCahe(BigInteger lastNumber, BigInteger[] lastFactors) {
        this.lastNumber = lastNumber;
        this.lastFactors = Arrays.copyOf(lastFactors, lastFactors.length);
    }

    public BigInteger[] getFactors(BigInteger i) {
        if (lastNumber == null || !lastNumber.equals(i)) {
            return null;
        } else {
            // 返回拷贝
            return Arrays.copyOf(lastFactors, lastFactors.length);
        }
    }
}

public class VolatileCachedFactors {
    private volatile OneValueCahe cahe = new OneValueCahe(null, null);

    public BigInteger[] service(BigInteger i) {
        BigInteger[] factors = cahe.getFactors(i);
        if (factors == null) {
            factors = factors(i);
            cache = new OneValueCahe(i, factors);
        }
        return factors;
    }
}
~~~



### 安全发布

为了安全的发布对象，对象的引用以及对象的状态必须同时对其他线程可见。一个正确创建的对象可以通过下列条件安全地发布：

- 通过静态初始化器初始化对象的引用；
- 将它的引用存储到 volatile 域或 AtomicReference；
- 将它的引用存储到正确创建的对象的 final 域中；
- 或者将它的引用存储到正在由锁保护的域中。

~~~java
/**
 * 静态初始化器由 JVM 在类的初始化阶段执行，由于 JVM 内在的同步，该机制确保了以这种方式初始化对象可以被安全地发布
 */
public static Holder holder = new Holder()
~~~