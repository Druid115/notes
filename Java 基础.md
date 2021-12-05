## Java 基础

### Exception 和 Error

Exception 和 Error 都继承了 Throwable 类，在 Java 中只有 Throwable 类型的实例才可以被抛出或者捕获，它是异常处理机制的基本组成类型。

Exception 是程序正常运行中，可以预料的意外情况，可能并且应该被捕获，进行相应处理。Error 是指在正常情况下，不大可能出现的情况，绝大部分的 Error 都会导致程序处于非正常的、不可恢复状态，所以不便于也不需要捕获。

Exception 又分为可检查异常和不检查异常，可检查异常在源代码里必须显式地进行捕获处理，这是编译期检查的一部分。不检查异常就是所谓的运行时异常，类似 NullPointerException、ArrayIndexOutOfBoundsException 之类，通常是可以编码避免的逻辑错误，具体根据需要来判断是否需要捕获，并不会在编译期强制要求。

- NoClassDefFoundError 和 ClassNotFoundException

  NoClassDefFoundError：如果 JVM 或者 ClassLoader 实例尝试加载类的时候却找不到类的定义。要查找的类在编译的时候是存在的，运行的时候却找不到了，这个时候就会导致 NoClassDefFoundError。原因可能是打包过程漏掉了部分类，或者 jar 包出现损坏或者篡改。

  ClassNotFoundException：Java 支持使用反射方式在运行时动态加载类，例如使用 Class.forName 方法，如果这个类在类路径中没 有被找到，那么此时就会在运行时抛出 ClassNotFoundException 异常。另外一个原因：当一个类已经被某个类加载器加载到内存中了，此时另一个类加载器又尝试着动态地从同一个包中加载这个类。

- TODO try-catch-finally 的执行顺序

- try-catch 代码段会产生额外的性能开销，往往会影响 JVM 对代码进行优化，尽量不要一个大的 try 包住整段的代码。 Java 每实例化一个 Exception，都会对当时的栈进行快照，这是一个相对比较重的操作。如果发生的非常频繁，这个开销可就不能被忽略了。



### String、StringBuffer、StringBuilder

- String 是典型的 Immutable 类，被声明成为 final class，所有属性也都是 final 的。在 Java 9 中，为了减少 char 类型占用两个 bytes 大小造成的内存浪费，将数据存储方式从 char 数组改变为一个 byte 数组加上一个标识编码的所谓 coder，并且将相关字符串操作类都进行了修改。在 JDK 8 中，字符串拼接操作会自动转换为StringBuilder 操作，而 JDK 9 为了更加统一字符串操作优化，提供了 StringConcatFactory 来作为一个统一的入口。
- StringBuffer 是为解决使用 String 拼接字符串产生太多中间对象的问题而提供的一个类，本质是一个线程安全的可修改字符序列，带来了额外的性能开销
- StringBuilder 在能力上和 StringBuffer 没有本质区别，但是它去掉了线程安全的部分

