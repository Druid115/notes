## Java 并发编程实践笔记（二）— 线程安全

编写线程安全的代码，本质上就是管理对状态的访问，而且通常是共享的、可变的状态。通俗的说，一个对象的状态就是它的数据。



### 线程安全

当多个线程访问一个类时，如果不用考虑这些线程在运行时环境下的调度和交替执行，并且不需要额外的同步及在调用方法代码不必做其他的协调，这个类的行为仍然是正确的，那么称这个类是线程安全的。



#### 无状态对象永远是线程安全的

无状态对象：不包含数据域或引用其他类的域。一次特定计算的瞬时状态，会唯一地存在本地变量中，这些本地变量存储在线程的栈中，只有执行线程才能访问。

~~~java
public class A {
    public A() {
    }
    
    public String hello() {
        return "Hello World!";
    }
}
~~~

有状态对象：有对应实例的对象，可以用来保存数据。有状态对象不一定是线程不安全的。

~~~java
/**
 * 成员变量的值是无法被改变的，此对象线程安全
 */
public class User {
    private String name;
    
    public String getName() {
        return name;
    }
}

/**
 * 引用了一个无状态对象，此对象线程安全
 */
public class User {
    private SayHello sayHello = new SayHello();
}
public class SayHello {
    public String hello() {
        return "Hello World!";
    }
}

/**
 * 成员变量的值可以被修改，此对象非线程安全
 */
public class User {
    private String name;
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
}
~~~



### 原子性

假设有操作 A 和 B，如果从执行 A 的线程的角度看，当其他线程执行 B 时，要么 B 全部执行完成，要么一点都没有执行，这样 A 和 B 互为原子操作。

原子性提供了互斥访问，同一时刻只能有一个线程来对它进行操作。

可以使用线程安全的对象来管理状态（atomic）。若一个不变约束涉及到多个状态变量时，变量间不是独立的，不能简单的引入更多的线程安全对象。为了保护状态的一致性，要在单一的原子操作中更新相互关联的状态变量。

~~~java
/**
 * 计算输入数字的因数，每次缓存上次的计算结果
 *
 * 非线程安全，无法保证同时更新和获得 lastNumber 和 lastFactors
 */
public class CachingFactors {
    private final AtomicReference<BigInteger> lastNumber = new AtomicReference<>();
    private final AtomicReference<BigInteger[]> lastFactors = new AtomicReference<>();

    public BigInteger[] service(BigInteger i) {
        if (i.equals(lastNumber.get())) {
            return lastFactors.get();
        } else {
            BigInteger[] factors = factor(i);
            lastNumber.set(i);
            lastFactors.set(factors);
            return factors;
        }
    }
}
~~~



### 有序性

一个线程观察其他线程中的指令执行顺序，由于指令重排序的存在，该观察结果一般杂乱无章。

#### happens-before 原则

程序次序规则：在一个线程内，按照控制流顺序（非程序代码顺序，因为要考虑分支、循环等结构），书写在前面的操作先行发生于书写在后面的操作。

管程锁定规则：一个 unlock 操作先行发生于后面对同一个锁的 lock 操作。

volatile 变量原则：对一个 volatile 变量的写操作先行发生于后面对这个变量的读操作。

线程启动原则：Thread 对象的 start() 方法先行发生于此线程的每一个动作。

线程终止原则：线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过 Thread.join() 方法结束、Thread.isAlive() 的返回值等手段检测到线程已经终止执行。

线程中断原则：对线程的 interrupt() 方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupted() 方法检测到是否有中断发生。

对象终结原则：一个对象的初始化完成（构造函数执行结束）先行发生于它的 finalize() 方法的开始。

传递性：如果操作 A 先行发生于操作 B，操作 B 先行发生于操作 C，那就可以得出操作 A 先行发生于操作 C 的结论。



### 锁

#### 重进入（Reentrancy）

线程在试图获得自己占有的锁时，请求会成功。重进入意味着锁的请求是基于“每线程”，而不是“每调用”的。重进入的实现是通过为每个锁关联一个请求计数和一个占有它的线程。

~~~java
public class Father {
    public synchronized void doSomething() {
    }
}

public class Son extends Father {
    @Override
    public synchronized void doSomething() {
        super.doSomething();
    }
}
~~~

有些耗时的计算或者操作，比如网络或者控制台 I/O，难以快速完成。执行这些操作期间不要占有锁。