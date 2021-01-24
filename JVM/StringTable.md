## StringTable

String 声明为 final 的，不可被继承。实现了 Serializable 接口，支持序列化。实现了 Comparable 接口，可以比较大小。在 JDK8 及以前内部定义了 final char[] value 用于存储字符串数据，JDK9 时改为 byte[]：

1. String 类的当前实现将字符存储在 char 数组中，每个字符使用两个字节(16 位)。
2. 从许多不同的应用程序收集的数据表明，字符串是堆使用的主要组成部分，而且大多数字符串对象只包含拉丁字符。这些字符只需要一个字节的存储空间，因此这些字符串对象的内部 char 数组中有一半的空间将不会使用。
3. 之前 String 类使用 UTF-16 的 char[] 数组存储，现在改为 byte[] 数组外加一个编码标志位存储，该编码标志将指定 String 类中 byte[] 数组的编码方式。
4. 同时基于 String 的数据结构，例如 StringBuffer 和 StringBuilder 也同样做了修改。

Java 语言规范里要求完全相同的字符串字面量，应该包含同样的 Unicode 字符序列，并且必须是指向同一个 String 类实例。



### 底层结构说明

**字符串常量池不会存储相同内容的字符串。**字符串常量池是一个固定大小的 HashTable，默认值大小长度是 1009。如果存放的字符串非常多，就会造成 Hash 冲突严重，导致链表会很长，从而造成的影响就是当调用 String.intern() 方法时性能会大幅下降。使用 **-XX:StringTablesize** 可设置字符串常量池的长度。

- JDK6 中，StringTable 是固定 1009 的长度，所以如果常量池中的字符串过多就会导致效率下降很快，StringTablesize 设置没有要求。
- JDK7 中，StringTable 的长度默认值是 60013，StringTablesize 设置没有要求。
- JDK8 中，StringTable 的长度默认值是 60013，StringTable 可以设置的最小值为 1009。



### 内存分配演进

在 Java 语言中有 8 种基本数据类型和一种比较特殊的类型 String，为了使它们在运行过程中速度更快、更节省内存，都提供了一种常量池的概念。常量池就类似一个 Java 系统级别提供的缓存。8 种基本数据类型的常量池都是系统协调的，String 类型的常量池比较特殊，它的主要使用方法有两种：

- 直接使用双引号声明出来的 String 对象会直接存储在常量池中。
- 如果不是用双引号声明的 String 对象，可以使用 String 提供的 intern() 方法。

Java6 及以前，字符串常量池存放在永久代中。Java7 及 8 中字符串常量池的位置调整到 Java 堆内。



