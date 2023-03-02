### Java 注解

注解是对代码的一种增强，可以在代码编译或者程序运行期间获取注解的信息。



#### 定义注解

~~~java
public @interface 注解名称{
	[public] 参数类型 参数名称1() [default 参数默认值];
	[public] 参数类型 参数名称2() [default 参数默认值];
	[public] 参数类型 参数名称n() [default 参数默认值];
}
~~~

注解中可以定义多个参数，参数的定义有以下特点： 

1. 访问修饰符必须为 public，不写默认为 public；
2. 该元素的类型只能是基本数据类型、String、Class、枚举类型、注解类型（体现了注解的嵌套效果）以及上述类型的一维数组；
3. 该元素的名称一般定义为名词，如果注解中只有一个元素，请把名字起为 value（后面使用会带来便利操作）；
4. 参数名称后面的 () 不是定义方法参数的地方，也不能在括号中定义任何参数，仅仅只是一个特殊的语法；
5. default 代表默认值，值必须和第2点定义的类型一致，如果没有默认值，代表后续使用注解时必须给该类型元素赋值。



#### 指定注解的使用范围

使用 @Target 注解定义注解的使用范围，自定义注解上也可以不使用 @Target 注解，如果不使用，表示自定义注解可以用在任何地方。

~~~java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
    ElementType[] value();
}



public enum ElementType {
    /** 类、接口、枚举、注解上面 */
	TYPE,
	/** 字段上 */
	FIELD,
	/** 方法上 */
	METHOD,
	/** 方法的参数上 */
	PARAMETER,
	/** 构造函数上 */
	CONSTRUCTOR,
	/** 本地变量上 */
	LOCAL_VARIABLE,
	/** 注解 */
	ANNOTATION_TYPE,
	/** 包上 */
	PACKAGE,
	/** 类型参数上(1.8) */
	TYPE_PARAMETER,
	/** 类型名称上(1.8) */
	TYPE_USE
}
~~~



#### 指定注解的保留策略

Java 程序的 3 个过程：源码阶段；源码被编译为字节码之后变成 class 文件；字节码被虚拟机加载然后运行。可以通过 @Retention 注解来指定注解保留在哪个阶段。

~~~java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention {
    RetentionPolicy value();
}



public enum RetentionPolicy {
	/** 注解只保留在源码中，编译为字节码之后就丢失了，也就是 class 文件中就不存在了 */
	SOURCE,
	/** 注解只保留在源码和字节码中，运行阶段会丢失 */
	CLASS,
	/** 源码、字节码、运行期间都存在 */
	RUNTIME
}
~~~

