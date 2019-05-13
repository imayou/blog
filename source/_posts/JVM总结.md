---
layout: JVM
title: JVM总结
date: 2019-05-13 14:50:32
tags: JVM
---
#### JVM内存模型
* 线程独占  
  - 栈(stack) 线程执行方法创建栈针用来存储局部变量表，操作栈，动态链接，方法出口等信息，调用方法入栈，方法返回时出栈。
  - 本地方法栈(Native Method Stack) 执行native方法时使用
  - 程序计数器(Program counter Gegister) 保存当前线程执行的字节码位置，每个线程工作时都有一个独立计数器，只为执行java方法服务，执行native方法时程序计数器为空，
* 线程共享  
  - 堆(heap) 存放对象的实例 
  - 方法区(method area) 存储被虚拟机加载的类信息，常量，变量，jvm优化后的数据，永久代和jdk8中的metaspace区
  
#### 堆内存划分
Survivor[sərˈvaɪvər] 幸存者

堆内存分为新生代，老年代，永久代（java8后不存在）
新生代又分为`Eden`和`2个Survivor`，Eden又分为多个`TLAB(Thread Local Allocation Buffer)`，每个线程的私有缓存区域。在TLAB中有起始`start`和`stop`位置还有一个`top`指针，当top和end相遇时表示该缓存已满，jvm会再从Eden里面分配一块。Eden和Survivor是按比例划分的，如果`SurvivorRatio=8(-XX:SurvivorRatio=value=8)`，那么Survivor就是Eden的`8/1`，就是新生代的`1/10`，因为`youngGen=Eden+2*Survivor`。

`-XX:NewRatio=value` 新生代与老年代比例 默认值2，表示老年代是新生代的2倍大，也就是说新生代是堆大小的1/3
`-XX:NewSize=value `新生代大小设定具体的内存大小数值
`-Xmx value` 最大堆体积
`-Xms value` 初始的最小堆体积

#### 类加载与卸载
加载是指将编译好的类文件中的字节码读入到内存中，将其放到方法区内并创建对应的class对象。加载-链接-初始化   链接分为验证-准备-解析3步。
加载(文件到内存)-验证(文件格式，元数据，字节码，符号引用)-准备(类变量内存)-解析(引用替换，字段解析，接口解析，方法解析)-初始化(静态块，静态变量)-使用(实例化)-卸载(GC)
加载器
* Bootstrap ClassLoader 启动类加载器--<JAVA_HOME>/lib
* ExtClassLoader        扩展类加载器--<JAVA_HOME>/lib/ext
* AppClassLoader        应用(系统)加载器--java -classpath
* Custom ClassLoader    自定义类加载器
> 双亲委派模式 先让父类加载一直到顶部父类，如果父类无法完成则子类加载。

#### 可达性分析
其原理简单来说，就是将对象及其引用关系看作一个图，选定活动的对象作为 GC Roots，然后跟踪引用链条，如果一个对象和 GC Roots 之间不可达，也就是不存在引用链条，那么即可认为是可回收对象。JVM 会把虚拟机栈和本地方法栈中正在引用的对象、静态属性引用的对象和常量，作为 GC Roots。

#### 新生代对象的移动
* 大部分对象创建都是在Eden的，除了个别大对象外。
* Minor GC开始前，to-survivor是空的，from-survivor是有对象的。
* Minor GC后，Eden的存活对象都copy到to-survivor中，from-survivor的存活对象也复制to-survivor中。其中所有对象的年龄+1(-XX:MaxTenuringThreshold=15，默认15，年龄达到15阀值就晋升至老年代)
* from-survivor清空，成为新的to-survivor，带有对象的to-survivor变成新的from-survivor。重复回到步骤2

#### 收集器较多，特征不一，适用于不同的业务场景：
`Serial[ˈsɪriəl]` 收集器：串行运行；作用于新生代；复制算法；响应速度优先；适用于单CPU环境下的client模式。
`ParNew`收集器：并行运行；作用于新生代；复制算法；响应速度优先；多CPU环境Server模式下与CMS配合使用。
`Parallel Scavenge  [ˈpærəlel][ˈskævɪndʒ]` 收集器：并行运行；作用于新生代；复制算法；吞吐量优先；适用于后台运算而不需要太多交互的场景。
`Serial Old`收集器：串行运行；作用于老年代；标记-整理算法；响应速度优先；单CPU环境下的Client模式。
`Parallel Old`收集器：并行运行；作用于老年代；标记-整理算法；吞吐量优先；适用于后台运算而不需要太多交互的场景。
`CMS`收集器：并发运行；作用于老年代；标记-清除算法；响应速度优先；适用于互联网或B/S业务。
`G1`收集器：并发运行；可作用于新生代或老年代；标记-整理算法+复制算法；响应速度优先；面向服务端应用。逻辑分代，将内存划分为小块的区域(Regon)。收集步骤：
年轻代：直接并行复制(STW)。
老年代4步骤：
1.初始标记(STW对跟对象GCRoot的标记)
2.并发标记(同用户线程一起执行)
3.最终标记(STW，3色标记)
4.复制/清除(STW，优先回收大的Regon，这也是G1的由来)，每次只清理一部分，保证停顿不会太长。
[Epsilon GC](http://openjdk.java.net/jeps/318)，简单说就是个不做垃圾收集的 GC，似乎有点奇怪，有的情况下，例如在进行性能测试的时候，可能需要明确判断 GC 本身产生了多大的开销，这就是其典型应用场景。
[ZGC](http://openjdk.java.net/jeps/333)，这是 Oracle 开源出来的一个超级 GC 实现，具备令人惊讶的扩展能力，比如支持 T bytes 级别的堆大小，并且保证绝大部分情况下，延迟都不会超过 10 ms。虽然目前还处于实验阶段，仅支持 Linux 64 位的平台，但其已经表现出的能力和潜力都非常令人期待。
当然，其他厂商也提供了各种独具一格的 GC 实现，例如比较有名的低延迟 GC，<a href="https://www.infoq.com/articles/azul_gc_in_detail">Zing</a>和<a href="https://wiki.openjdk.java.net/display/shenandoah/Main">Shenandoah</a>等，有兴趣请参考我提供的链接。
收集步骤：
新生代：复制算法
老年代：
1.着色指针 64  4T堆寻址用42位剩下22位，可以用于着色标记。 2.读屏障解决GC线程和应用线程并发修改对象状态的问题，使用读屏障之后对单个对象概率减速。
3.大部分时候都不需要STW的并发处理。
4.没有分代的Region(类似G1的Region)，动态决定大小，好管理大对象。
5.压缩整理，类似G1 短暂的STW

#### 一些工具
* jmap
```
jmap -dump:format=b,file=<路径> pid  堆转储
```
* jstack
```
jstack -l 3036  打出线程信息
```
* jstat
```
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
jstat -gc -h3 31736 1000 10 进程id为31736的gc情况，每隔1000ms打印一次记录，打印10次停止，每3行后打印指标头部
jstat -gc 17325 1000 20 输出pid=17325的垃圾收集状况1000毫秒执行一次共执行20次
```
* jps
```
jps -v 查看现在运行的jvm进程和应用参数
```
* JConsole
```
https://docs.oracle.com/javase/7/docs/technotes/guides/management/jconsole.html
```
* jhat
```
jhat -J-Xmx512m <heap dump file>
```
* Java Mission Control（JMC）
```
https://www.oracle.com/technetwork/java/javaseproducts/mission-control/java-mission-control-1998576.html
```
* visualvm
```
start visualvm.exe --jdkhome "D:\Devloper\Java\tools\openjdk\java\jdk-11.0.1"
```