## StringTable

String 声明为 final 的，不可被继承。实现了 Serializable 接口，支持序列化。实现了 Comparable 接口，可以比较大小。在 JDK8 及以前内部定义了 final char[] value 用于存储字符串数据，JDK9 时改为 byte[]：

1. String 类的当前实现将字符存储在 char 数组中，每个字符使用两个字节(16 位)。
2. 从许多不同的应用程序收集的数据表明，字符串是堆使用的主要组成部分，而且大多数字符串对象只包含拉丁字符。这些字符只需要一个字节的存储空间，因此这些字符串对象的内部 char 数组中有一半的空间将不会使用。
3. 之前 String 类使用 UTF-16 的 char[] 数组存储，现在改为 byte[] 数组外加一个编码标志位存储，该编码标志将指定 String 类中 byte[] 数组的编码方式。
4. 同时基于 String 的数据结构，例如 StringBuffer 和 StringBuilder 也同样做了修改。

Java 语言规范里要求完全相同的字符串字面量，应该包含同样的 Unicode 字符序列，并且必须是指向同一个 String 类实例。



### 底层结构说明

**字符串常量池不会存储相同内容的字符串。**字符串常量池是一个固定大小的 HashTable，默认值大小长度是 1009。如果存放的字符串非常多，就会造成 Hash 冲突严重，导致链表会很长，造成的影响就是当调用 String.intern() 方法时性能会大幅下降。使用 **-XX:StringTablesize** 可设置字符串常量池的长度。

- JDK6 中，StringTable 是固定 1009 的长度，所以如果常量池中的字符串过多就会导致效率下降很快，StringTablesize 设置没有要求。
- JDK7 中，StringTable 的长度默认值是 60013，StringTablesize 设置没有要求。
- JDK8 中，StringTable 的长度默认值是 60013，StringTable 可以设置的最小值为 1009。



### 内存分配演进

在 Java 语言中有 8 种基本数据类型和一种比较特殊的类型 String，为了使它们在运行过程中速度更快、更节省内存，都提供了一种常量池的概念。常量池就类似一个 Java 系统级别提供的缓存。8 种基本数据类型的常量池都是系统协调的，String 类型的常量池比较特殊，它的主要使用方法有两种：

- 直接使用双引号声明出来的 String 对象会直接存储在常量池中。
- 如果不是用双引号声明的 String 对象，可以使用 String 提供的 intern() 方法。

Java6 及以前，字符串常量池存放在永久代中。Java7 及 8 中字符串常量池的位置调整到 Java 堆内。



### 字符串拼接操作

常量与常量（或者常量的引用）的拼接结果在常量池，原理是编译期优化。只要其中有一个是变量，结果就在堆中，变量拼接的原理是 StringBuilder。

如果拼接的结果调用 intern() 方法，则主动将常量池中还没有的字符串对象放入池中，并返回此对象地址：

- 如果存在，则返回字符串在常量池中的地址
- 如果字符串常量池中不存在该字符串，则在常量池中创建一份，并返回此对象的地址

~~~java
@Test
public void test1() {
    // 编译期优化：等同于 "abc"
    String s1 = "a" + "b" + "c";
    String s2 = "abc"; 
    /*
     * idea 反编译结果
     * String s1 = "abc";
     * String s2 = "abc"
     */
    System.out.println(s1 == s2); //true
    System.out.println(s1.equals(s2)); //true
}

 0 ldc #2 <abc>
 2 astore_1
 3 ldc #2 <abc>
 5 astore_2
 6 getstatic #3 <java/lang/System.out>
 9 aload_1
10 aload_2
11 if_acmpne 18 (+7)
14 iconst_1
15 goto 19 (+4)
18 iconst_0
19 invokevirtual #4 <java/io/PrintStream.println>
22 getstatic #3 <java/lang/System.out>
25 aload_1
26 aload_2
27 invokevirtual #5 <java/lang/String.equals>
30 invokevirtual #4 <java/io/PrintStream.println>
33 return
~~~

~~~java
@Test
public void test2() {
    String s1 = "javaEE";
    String s2 = "hadoop";

    String s3 = "javaEEhadoop";
    String s4 = "javaEE" + "hadoop";

    // 如果拼接符号的前后出现了变量，则相当于在堆空间中 new String()
    String s5 = s1 + "hadoop";
    String s6 = "javaEE" + s2;
    String s7 = s1 + s2;

    System.out.println(s3 == s4); // true
    System.out.println(s3 == s5); // false
    System.out.println(s3 == s6); // false
    System.out.println(s3 == s7); // false
    System.out.println(s5 == s6); // false
    System.out.println(s5 == s7); // false
    System.out.println(s6 == s7); // false

    // 此时常量池中已存在该字符串
    String s8 = s6.intern();
    System.out.println(s3 == s8);//true
}
~~~

~~~java
@Test
public void test3() {
    String s1 = "a";
    String s2 = "b";
    String s3 = "ab";
    /**
     * 在 JDK5 之后使用的是 StringBuilder,在 JDK5 之前使用的是 StringBuffer
     * s1 + s2 的执行细节：
     * ① new StringBuilder();
     * ② append("a")
     * ③ append("b")
     * ④ toString() --> 约等于 new String("ab")
     */
    String s4 = s1 + s2;
    System.out.println(s3 == s4); // false
}

 0 ldc #14 <a>
 2 astore_1
 3 ldc #15 <b>
 5 astore_2
 6 ldc #16 <ab>
 8 astore_3
 9 new #9 <java/lang/StringBuilder>
12 dup
13 invokespecial #10 <java/lang/StringBuilder.<init>>
16 aload_1
17 invokevirtual #11 <java/lang/StringBuilder.append>
20 aload_2
21 invokevirtual #11 <java/lang/StringBuilder.append>
24 invokevirtual #12 <java/lang/StringBuilder.toString>
27 astore 4
29 getstatic #3 <java/lang/System.out>
32 aload_3
33 aload 4
35 if_acmpne 42 (+7)
38 iconst_1
39 goto 43 (+4)
42 iconst_0
43 invokevirtual #4 <java/io/PrintStream.println>
46 return
~~~

~~~java
@Test
public void test4() {
    // 带 final 的变量在编译时就已经确定了该变量的值，当做常量来处理
    final String s1 = "a";
    final String s2 = "b";
    String s3 = "ab";
    String s4 = s1 + s2;
    System.out.println(s3 == s4); // true
}

 0 ldc #14 <a>
 2 astore_1
 3 ldc #15 <b>
 5 astore_2
 6 ldc #16 <ab>
 8 astore_3
 9 ldc #16 <ab>
11 astore 4
13 getstatic #3 <java/lang/System.out>
16 aload_3
17 aload 4
19 if_acmpne 26 (+7)
22 iconst_1
23 goto 27 (+4)
26 iconst_0
27 invokevirtual #4 <java/io/PrintStream.println>
30 return
~~~

~~~java
/**
 * 通过 StringBuilder 的 append() 方式添加字符串的效率要远高于使用 String 的字符串拼接方式，原因：
 * ① StringBuilder 的 append() 的方式：自始至终中只创建过一个 StringBuilder 的对象
 * ② 使用 String 的字符串拼接方式：创建多个 StringBuilder 和 String 的对象，由于创建了较多的的对象，内存占用更大，如果进行 GC，需要花费额外的时间。
 *
 * 改进的空间：建议使用构造器实例化
 * StringBuilder s = new StringBuilder(highLevel); //new char[highLevel]
 */
@Test
public void test6() {
    long start = System.currentTimeMillis();
    
    // method1(100000);//4014
    method2(100000);//7

    long end = System.currentTimeMillis();

    System.out.println("花费的时间为：" + (end - start));
}

public void method1(int highLevel) {
    String src = "";
    for (int i = 0; i < highLevel; i++) {
        // 每次循环都会创建一个 StringBuilder、String
        src = src + "a";
    }
}

public void method2(int highLevel) {
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < highLevel; i++) {
        sb.append("a");
    }
}
~~~

**new String("ab") 会创建几个对象？**

~~~java
/**
 * 创建了两个对象：
 * 一个是：new 关键字在堆空间创建的
 * 另一个是：字符串常量池中的对象 "ab"
 */
public class StringNewTest {
    public static void main(String[] args) {
        String str = new String("ab");
    }
}

 0 new #2 <java/lang/String>
 3 dup
 4 ldc #3 <ab>
 6 invokespecial #4 <java/lang/String.<init>>
 9 astore_1
10 return

new #2 <java/lang/String> 在堆空间创建了一个字符串对象  
ldc #3 <ab> 在字符串常量池中放入 "ab"（如果之前字符串常量池中没有 "ab" 的话）     
~~~

**new String("a") + new String("b")  会创建几个对象？**

~~~java
/**
 * 对象1：new StringBuilder()
 * 对象2：new String("a")
 * 对象3：常量池中的 "a"
 * 对象4：new String("b")
 * 对象5：常量池中的 "b"
 *
 * StringBuilder 调用 toString()，toString() 调用 new String()
 * 对象6 ：new String("ab")
 * toString() 的调用，在字符串常量池中没有生成 "ab"
 */
public class StringNewTest {
    public static void main(String[] args) {
        String str = new String("a") + new String("b");
    }
}

 0 new #2 <java/lang/StringBuilder>
 3 dup
 4 invokespecial #3 <java/lang/StringBuilder.<init>>
 7 new #4 <java/lang/String>
10 dup
11 ldc #5 <a>
13 invokespecial #6 <java/lang/String.<init>>
16 invokevirtual #7 <java/lang/StringBuilder.append>
19 new #4 <java/lang/String>
22 dup
23 ldc #8 <b>
25 invokespecial #6 <java/lang/String.<init>>
28 invokevirtual #7 <java/lang/StringBuilder.append>
31 invokevirtual #9 <java/lang/StringBuilder.toString>
34 astore_1
35 return
     
@Override
public String toString() {
    // Create a copy, don't share the array
    return new String(value, 0, count);
}     
~~~

~~~java
public class StringIntern {
    public static void main(String[] args) {
        String s = new String("1");
        // 调用此方法之前，字符串常量池中已经存在"1"
        s.intern();
        String s2 = "1";

        // JDK6：false  JDK7 / JDK8：false
        // s 指向堆空间中的 "1" ，s2 指向字符创常量池中的 "1"
        System.out.println(s == s2);


        String s3 = new String("1") + new String("1");

        // 在字符串常量池中生成 "11"，此时：
        // JDK6：创建了一个新的对象 "11"，也就有新的地址
        // JDK7：常量池中并没有创建 "11"，而是记录了指向堆空间中 new String("11") 的地址（节省空间）
        s3.intern();
        String s4 = "11";

        // JDK6：false  JDK7 / JDK8：true
        System.out.println(s3 == s4);
        // 对象内存地址可以使用 System.identityHashCode(object) 方法获取
        System.identityHashCode(s3);
        System.identityHashCode(s4);
    }
}
~~~

**intern() 方法的总结：**

JDK6 中，将这个字符串对象尝试放入常量池时：

- 如果有，则不会放入，返回已有的对象的地址。

- 如果没有，会**把此对象复制一份**，放入常量池，并返回池中的对象地址。

JDK7 起，将这个字符串对象尝试放入常量池时：

- 如果有，则不会放入，返回已有的对象的地址。
- 如果没有，则会**把对象的引用地址复制一份**，放入常量池，并返回池中的引用地址。

~~~java
/**
 * 当程序中存在大量的字符串，尤其存在很多重复字符串时，使用 intern() 可以节省内存空间。
 */
public class StringIntern2 {
    static final int MAX_COUNT = 1000 * 10000;
    static final String[] arr = new String[MAX_COUNT];

    public static void main(String[] args) {
        Integer[] data = new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

        long start = System.currentTimeMillis();
        for (int i = 0; i < MAX_COUNT; i++) {
            // arr[i] = new String(String.valueOf(data[i % data.length]));
            arr[i] = new String(String.valueOf(data[i % data.length])).intern();

        }
        long end = System.currentTimeMillis();
        System.out.println("花费的时间为：" + (end - start));

        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.gc();
    }
}
~~~



### G1 中的 String 去重操作

背景：对许多 Java 应用（有大有小）做的测试得出以下结果：

- 堆存活数据集合里面 String 对象占了 25%
- 堆存活数据集合里面重复的 String 对象有 13.5%
- String 对象的平均长度是 45

G1 垃圾收集器能实现自动持续对重复的 String 对象进行去重，这样就能避免浪费内存：

1. 当垃圾收集器工作的时候，会访问堆上存活的对象，检查每一个对象是否是候选的要去重的String对象。
2. 如果是，把这个对象的一个引用插入到一个队列中等待后续的处理。后台将会运行一个去重的线程来处理这个队列，处理队列的一个元素意味着从队列删除这个元素，然后尝试去重它引用的 String 对象。
3. 使用一个 HashTable 来记录所有的被 String 对象使用的不重复的 char 数组。当去重的时候，会查询这个 HashTable 是否包含要去重对象的 char 数组。如果存在，则进行去除；如果不存在，则添加进 HashTable。