---
layout: java, 反射
title: 07-JVM是如何实现反射的
date: 2019-04-30 20:05:28
tags: java, 反射
---
#### 方法的反射调用也就是 Method.invoke是怎么实现的
```java
public final class Method extends Executable {
  ...
  public Object invoke(Object obj, Object... args) throws ... {
    ... // 权限检查
    MethodAccessor ma = methodAccessor;
    if (ma == null) {
      ma = acquireMethodAccessor();
    }
    return ma.invoke(obj, args);
  }
}
```
可见invoke是委派给了接口MethodAccessor,每个Method实例第一次反射调用都会生成一个委派实现，具体实现是一个本地实现。当进入了java虚拟机内部之后，我们便有Method实例所指向的方法的具体地址，反射调用无非是将传入参数准备好，然后调用进入目标方法
```java
// v0 版本
import java.lang.reflect.Method;

public class Test {
  public static void target(int i) {
    new Exception("#" + i).printStackTrace();
  }

  public static void main(String[] args) throws Exception {
    Class&lt;?&gt; klass = Class.forName("Test");
    Method method = klass.getMethod("target", int.class);
    method.invoke(null, 0);
  }
}

# 不同版本的输出略有不同，这里我使用了 Java 10。
$ java Test
java.lang.Exception: #0
        at Test.target(Test.java:5)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl .invoke0(Native Method)
 a      t java.base/jdk.internal.reflect.NativeMethodAccessorImpl. .invoke(NativeMethodAccessorImpl.java:62)
 t       java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.i .invoke(DelegatingMethodAccessorImpl.java:43)
        java.base/java.lang.reflect.Method.invoke(Method.java:564)
  t        Test.main(Test.java:131
```
可以看到，反射调用先是调用了 Method.invoke->委派实现（DelegatingMethodAccessorImpl）->本地实现（NativeMethodAccessorImpl）->目标方法。
> 括号里的Native Method其实是jvm底层执行c++代码

动态生成字节码的实现（动态实现）直接使用ivmoke指令来调用目标方法，之所以采用委派是为了能够在本地实现以及动态实现中切换。

许多反射仅会执行一次，java虚拟机设置了一个阀值15（可以通过Dsun.reflect.inflationThreshold=来调整），当这个值在15以下时采用本地实现，当达到15时便开始动态生成字节码，并将委派实现的委派对象切换到动态实现，这个过程我们称为`Inflation`。

反射调用的Inflation机制可以通过参数（-Dsun.reflect.noInflation=true）来关闭，关闭后一开始便会直接生成动态实现，而不会使用委派活本地实现。

#### 反射调用的开销
上面例子中先后调用了Class.forName,Class.getMethod以及Method.invoke三个操作，Class.forName会调用本地方法，Class.getMethod会遍历类的公用方法，如果该类没匹配到，将遍历父类的公用方法，这两个操作都是非常费时。

getMethod为代表的查找方法操作会返回查找结果的一份拷贝，应当避免在热点代码中使用返回Method数组的getMethods或者getDeclaredMethods方法，以减少不必要的堆空间消耗。应该是在应用程序中缓存Class.forname和Class.getMethod的结果。

然后分析实践，方法的反射调用会带来不少性能开销，原因主要有三个：变长参数方法导致的Object数组，基本类型的自动装箱、拆箱，还有最重要的方法内联。


#### 附录：反射API简介
通常来说，使用反射API的第一步便是获取Class对象，在Java中常见有这么三种
1. 使用静态方法Class.forName来获取
2. 调用对象的getClass()方法。
3. 直接用类名+".class"访问。对于基本类型来说，他们的包装类型（wrapper classes）拥有一个名为"TYPE"的final静态字段，指向该基本类型对应的Class对象。

