---
layout: blog
title: 深入浅出Java反射
date: 2018-08-28 17:35:55
categories: [java]
tags: [java核心技术]
---

> 反射的概念是由Smith在1982年首次提出的，主要是指程序可以访问、检测和修改它本身状态或行为的一种能力。有了反射，使Java相对于C、C++等语言就有了很强大的操作对象属性及其方法的能力，注意，反射与直接调用对象方法和属性相比，性能有一定的损耗，但是如果不是用在对性能有很强的场景下，反射都是一个很好且灵活的选择。

<!--more-->

**反射，它就像是一种魔法，引入运行时自省能力，赋予了 Java 语言令人意外的活力，通过运行时操作元数据或对象，Java 可以灵活地操作运行时才能确定的信息。**

下面笔者就分为 `Java反射基础`和`反射实现原理` 2部分来分析Java反射机制。

## Java反射基础

如果不知道某个对象的确切类型，[RTTI](https://baike.baidu.com/item/RTTI/5752573) 可以告诉你，但是有一个前提：这个类型在编译时必须已知，这样才能使用RTTI来识别它。而Class类与java.lang.reflect类库一起对反射进行了支持，该类库包含Field、Method和Constructor类，这些类的对象由JVM在启动时创建，用以表示未知类里对应的成员。

反射机制并没有什么神奇之处，当通过反射与一个未知类型的对象打交道时，JVM只是简单地检查这个对象，看它属于哪个特定的类。因此，那个类的.class对于JVM来说必须是可获取的，要么在本地机器上，要么从网络获取。所以对于RTTI和反射之间的真正区别只在于：
* RTTI：编译器在编译时打开和检查.class文件
* 反射：运行时打开和检查.class文件

*反射示例（获取对象所有属性值）：*
```java
public class Person {
    private String name;
    private int age;
    // ... setter/getter
}
 
Person person = new Person();
person.setName("luo");
person.setAge(25);
 
try {
    Class clazz = person.getClass();
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
        field.setAccessible(true);
        System.out.println(field.getType() + " | " +
                field.getName() + " = " +
                field.get(person));
    }
 
    // 通过反射获取某一个方法
    Method method = clazz.getMethod("setName", String.class);
    method.invoke(person, "bei");
 
    System.out.println(person);
} catch (Exception e) {
    e.printStackTrace();
}
```

## Java反射机制

反射技术在在框架和中间件技术应用较多，有一句老话就是反射是Java框架的基石。典型的使用就是Spring的IoC实现，不管对象谁管理创建，只要我能用就行。再比如RPC技术可以借助于反射来实现，本地主机将要远程调用的对象方法等信息发送给远程主机，这些信息包括class名、方法名、方法参数类型、方法入参等，远程主机接收到这些信息后就可以借助反射来获取并执行对象方法，然后将结果返回即可。

说了那么多，那么Java反射是如何实现的呢？简单来说Java反射就是靠JVM和Class相关类来实现的，Class相关类包括Field、Method和Constructor类等。

类加载器加载完成一个类之后，会生成类对应的Class对象、Field对象、Method对象、Constructor对象，这些对象都保存在JVM（方法区）中，这也说明了反射必须在加载类之后进行的原因。使用反射时，其实就是与上述所说的这几个对象打交道呀（貌似Java反射也就这么一回事哈）。

既然了解了Java反射原理，可以试想一下C++为什么没有反射呢，想让C++拥有反射该如何做呢？Java相对于C++实现反射最重要的差别就是Java可以依靠JVM这一悍将，可以由JVM保存对象的相关信息，然后应用程序使用时直接从JVM中获取使用。但是C++编译后直接变成了机器码了，貌似类或者对象的啥信息都没了。。。 其实想让C++拥有反射能力，就需要保存能够操作类方法、类构造方法、类属性的这些信息，这些信息要么由应用程序自己来做，要么由第三方工具来保存，然后应用程序使用从它那里获取，这些信息可以通过（函数）指针来记录，使用时通过指针来调用。

### Java反射实现原理

下面以`Method.invoke`流程为例来分析下反射的底层实现原理：
```java
public class Person {
    private String name;
    private int age;
 
    public String getName() {
        return name;
    }
    // 其他setter/getter方法
}
 
public static void main(String[] args) throws Exception {
    Person person = new Person();
    person.setName("luo");
    person.setAge(26);
 
    for (int i = 0; i < 20; i++) {
        Method method = Person.class.getMethod("getName");
        System.out.println(method.invoke(person));
    }
}
```

以上代码通过反射调用person对象中的方法，下面跟着源码看下`Method.invoke`的执行流程：
```java
// Method
public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
{
    MethodAccessor ma = methodAccessor;             // read volatile
    if (ma == null) {
        // 这里会调用reflectionFactory.newMethodAccessor(this)创建一个新的MethodAccessor
        // 并赋值给methodAccessor，下次就不会进入到这里了
        // ma实际类型是DelegatingMethodAccessorImpl，代理对目标方法的调用
        ma = acquireMethodAccessor();
    }
    return ma.invoke(obj, args);
}
 
class DelegatingMethodAccessorImpl extends MethodAccessorImpl {
    private MethodAccessorImpl delegate;
    DelegatingMethodAccessorImpl(MethodAccessorImpl var1) {
        this.setDelegate(var1);
    }
 
    public Object invoke(Object var1, Object[] var2) throws IllegalArgumentException, InvocationTargetException {
        return this.delegate.invoke(var1, var2);
    }
 
    void setDelegate(MethodAccessorImpl var1) {
        this.delegate = var1;
    }
}
 
// NativeMethodAccessorImpl
public Object invoke(Object var1, Object[] var2) throws IllegalArgumentException, InvocationTargetException {
    // ReflectionFactory.inflationThreshold()默认15，如果某一个Method反射调用超过15次，
    // 则自动生成GeneratedMethodAccessor赋值给DelegatingMethodAccessorImpl.delegate
    if (++this.numInvocations > ReflectionFactory.inflationThreshold() && !ReflectUtil.isVMAnonymousClass(this.method.getDeclaringClass())) {
        // 通过asm自动生成MethodAccessorImpl的实现类GeneratedMethodAccessor
        MethodAccessorImpl var3 = (MethodAccessorImpl)(new MethodAccessorGenerator()).generateMethod(this.method.getDeclaringClass(), this.method.getName(), this.method.getParameterTypes(), this.method.getReturnType(), this.method.getExceptionTypes(), this.method.getModifiers());
        this.parent.setDelegate(var3);
    }
 
    return invoke0(this.method, var1, var2);
}
// native方法，jni方式调用对应方法，最后调用的是对应的java方法
private static native Object invoke0(Method var0, Object var1, Object[] var2);
```

在默认情况下，方法的反射调用为**委派实现**，委派给本地实现来进行方法调用。在调用超过 15 次之后，委派实现便会将委派对象切换至动态实现。这个动态的字节码是在Java运行过程中通过ASM自动生成的，它将直接使用 invoke 指令来调用目标方法。

这里可以引申处一个问题，JDK为什么是要以**委派实现**来进行反射的调用呢？在运行过程中，可能涉及到本地实现切换到动态实现，这里使用委派实现，是为了对外统一封装反射的2种实现（`本地实现和动态实现`）。

下面分别看下`NativeMethodAccessorImpl(委派给本地实现)`和`GeneratedMethodAccessor(委派给自动生成对象)`的调用栈信息：

{% asset_img 20180828101811338.png %}

{% asset_img 20180828101846884.png %}

在执行`Method.invoke`超过15次时，会通过asm生成`GeneratedMethodAccessor`类，由该类调用对应的method方法。执行`NativeMethodAccessorImpl.invoke`是通过调用JNI方法，在JNI方法中再调用对应的java method方法，这种方式相对于使用`GeneratedMethodAccessor.invoe`方法来说，性能较弱，原因有以下几点：
* 针对本地方法，jvm无法优化，无法动态inline，其他高级的优化方案都无法优化jni。
* 执行native涉及到运行栈切换（虚拟机栈切换到本地方法栈），如果本地方法中再调用java方法是有一定的开销的，肯定比不上Java中调用Java方法。
* 二者内存模型不一样，参数需要转换，比如字符串，数组，复杂结构。转换成本非常高。此开销和调用接口参数有关。

除了上面`Method.invoke`的例子外，调用 `Class.forName`会调用本地方法，`Class.getMethod `则会遍历该类的公有方法。如果没有匹配到，它还将遍历父类的公有方法。可想而知，这两个操作都非常费时。

> 这里，留一个问题：如果一个面试官问你：谈谈 Java 反射机制，动态代理是基于什么原理？这个题目给我的第一印象是稍微有点诱导的嫌疑，可能会下意识地以为动态代理就是利用反射机制实现的，这么说也不算错但稍微有些不全面。功能才是目的，实现的方法有很多。 限于篇幅，这里就不再展开讨论动态代理了，留到下一篇文章再一起讨论吧 :)

参考资料：
1、[https://www.zhihu.com/question/38509124](https://www.zhihu.com/question/38509124)
2、深入拆解JVM-极客时间


