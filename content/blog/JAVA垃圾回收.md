---
external: false
title: 垃圾回收是什么
date: 2023-03-21 23:22:23
tags:
- 后端
- Java
---

# 一、什么是垃圾回收

```ad-quote
garbage collector (GC) is a memory management tool. It achieves automatic memory management through the following operations:

Allocating objects to a young generation and promoting aged objects into an old generation.

Finding live objects in the old generation through a concurrent ( parallel ) marking phase . The Java HotSpot VM triggers the marking phase when the total Java heap occupancy exceeds the default threshold .

Recovering free memory by compacting live objects through parallel copying. 
```

垃圾回收是内存管理的工具,用于回收已失去所有引用的对象占用的内存空间。这些对象被称为“垃圾”

要理解垃圾回收,需要理解以下几个问题:：
1. 哪些内存需要回收？
2. 什么时候回收？ 
3. 如何回收？

# 二、哪些内存需要回收

垃圾回收的目标是清除不再使用的内存，但判断何时回收对象却很困难

如果错误地回收了正在使用的对象，必然会出现问题。 因此，垃圾回收机制需要能准确判断对象的使用状态，以避免删除正在使用的对象

## 2.1、引用计数法

很好理解，既然要判断这个对象有没有在使用，那在对象中添加一个引用计数器不就好了吗

当一个对象被创建时，它的引用计数器被初始化为1。当对象被其他对象引用时，它的引用计数器会加1。当对象的引用被删除时，它的引用计数器会减1。当引用计数器的值为0时，该对象就可以被回收了

这个方式很简单，虽然占用了一些额外的内存空间来进行计数，不过也有很好的判定效率

不过用这种方式有一定的缺陷，比如下面的代码在执行的过程中，`objA`和`objB`的计数器值是多少呢？

```java

public class ReferenceCountingGC {
    public Object instance = null;
    private static final int _1MB = 1024 * 1024;
    private byte[] bigSize = new byte[2 * _1MB];
    public static void testGC() {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objB;
        objB.instance = objA;
        objA = null;
        objB = null;
        System.gc();
    }
    public static void main(String[] args) {
        testGC();
    }
}
```

没错，因为对象A和对象B相互引用，虽然此时A和B应该都是待回收的垃圾对象，但是通过引用计数法无法识别，两个对象一直得不到回收

## 2.2可达性分析算法

为了解决循环引用的问题，又引出了一个新的算法

可达性算法的原理是以一系列叫做 `GC Root` 的对象为起点出发，引出它们指向的下一个节点，再以下个节点为起点，引出此节点指向的下一个结点。这样通过 GC Root 串成的一条线就叫引用链），直到所有的结点都被访问完毕。如果一个对象没有任何引用链相连，即 GC Root 到这个对象不可达，则证明此对象不可用，可以被回收

![](/images/可达性分析引用链.png)

**现代虚拟机基本都是采用可达性分析算法**

### 2.2.1、GC Roots

GC Roots的对象包括以下几种： 

```ad-quote
- 在虚拟机栈（栈帧中的本地变量表）中引用的对象
	- 如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量等
- 在方法区中类静态属性引用的对象
	- 譬如Java类的引用类型静态变量
- 在方法区中常量引用的对象
	- 譬如字符串常量池（String Table）里的引用
- 在本地方法栈中JNI（即通常所说的Native方法）引用的对象
- Java虚拟机内部的引用
	- 如基本数据类型对应的Class对象，一些常驻的异常对象（比如 NullPointExcepiton、OutOfMemoryError）等，还有系统类加载器
- 所有被同步锁（synchronized关键字）持有的对象
- 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等
```

除了这些固定的GC Roots集合以外，还可以临时加入，这些可以临时加入的对象后面会讲到

### 2.2.2、对象自救

```ad-warning
从Java9开始，finalize方法已被标注为**@Deprecated**
```

当一个对象已被判定不可达了，它实际还可以再垂死挣扎一下，Object提供了`finalize` 方法，垃圾回收器回收对象所占内存之前会调用该方法，如果执行这个方法后对象又和`GC Roots` 连接上了，那这个对象就存活下来了

```ad-question
为什么`finalize`可以让对象又活过来呢？
```

因为一个对象要被回收，至少要经过**两次标记**，第一次标记只是判了一个缓刑，标记后会排队执行`finalize` 方法，之后会进行第二次标记，如果这期间它重新连接了，就又复活了，好比一个死刑犯在临刑前发明了专利，被免除了死刑

## 2.3、方法区的回收

方法区并不在堆里，但是它也是可以被回收的，根据Java虚拟机规范的规定，方法区无法满足内存分配需求时，也会抛出`OutOfMemoryError`异常

虽然规范规定虚拟机可以不实现垃圾收集，因为和堆的垃圾回收效率相比，方法区的回收效率实在太低. 在jdk1.7中，方法区在永久代，而永久代本身就是垃圾回收概念下的产物，`full gc`时就会对方法区回收

```ad-quote
The method area is created on virtual machine start-up. Although the method area is logically part of the heap, simple implementations may choose not to either garbage collect or compact it
```

方法区的垃圾收集主要回收两部分内容：
- **废弃的常量**
- **不再使用的类型**

 回收废弃常量与回收Java堆中的对象非常类似判断当前还有没有引用就行，
 而判定一个类型是否属于“**不再被使用的类**”的条件就比较苛刻了
 
 需要同时满足下面三个条件：
 1. 该类所有的实例都已经被回收，也就是Java堆中不存在该类及其任何派生子类的实例
 2. 加载该类的类加载器已经被回收
 3. 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法

同时满足这三个条件，虚拟机就被授权可以进行回收了，不过和对象的回收不同，类型并不是一定会被回收掉的

```ad-tip
JVM提供了`-Xnoclassgc`参数控制对于类的垃圾回收
```

# 三、垃圾回收算法

## 3.1、分代收集

常用的垃圾收集器都遵循着一个共同的设计原则，即将Java堆**划分出不同的区域**，然后将回收对象依据其**年龄**分配到不同的区域之中存储，这就是**分代收集**的理论

```ad-info
- 如果一个区域中**大多数对象都是要被回收的**，那么把它们集中放在一起，每次回收时就只需要关注那些**少量需要保留的对象**，这样就能以较低代价回收到大量的空间
- 如果一个区域中都是**硬骨头**，虚拟机就可以通过降低回收频率来降低垃圾回收的开销

```

这两种区域在堆中指的便是**新生代**和**老年代**

## 3.2、回收算法

![](/images/垃圾回收算法%202.png)
我们需要了解的垃圾回收算法有以下几种：

-   标记-清除算法
-   标记-复制算法
-   标记-整理算法

这些算法的优缺点都是非常明显的

### 3.2.1、标记-清除算法

该算法最早出现也最基础，正如名字一样，分为两个步骤：标记和清除，标记可以是标记保留的对象也可以是标记可回收的对象

它主要存在两个问题
-   效率不稳定 - 堆中有大量对象，且大多是要回收的情况下，需要频繁执行**标记-清除动作**
-   会**产生大量内存碎片**

```ad-hint
内存碎片是指内存的空间比较零碎，缺少大段的连续空间。这样假如突然来了一个大对象，会找不到足够大的连续空间来存放，于是不得不再触发一次gc
```

### 3.2.2、标记-复制算法

该算法为了解决·**标记-清除算法**效率不稳定的问题引入，它的思想是将可用**内存**按容量**划分为大小相等的两块**，每次**只使用其中的一块**。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉

这个算法解决了效率不稳定的问题，也不会产生内存碎片，但是它引出了新的问题：
-  存活对象多的情况下，内存复制的开销很大
-  可用内存缩水到原来的一半

也因此，该算法主要用在**新生代存活对象较少**的情况下

### 3.2.3、标记-整理算法

复制算法的特性决定了它不能用在老年代中，针对老年代的特性引入了该方法，标记的阶段和**标记-清除算法**一样，不同的是，可回收对象并不会直接清理，而是让所有存活的对象都向内存空间一端移动，然后直接清理掉边界以外的内存

```ad-hint
该算法和清除算法主要的**区别**就是**对象要不要移动**，移动则内存回收时会更复杂，不移动则内存分配时会更复杂，影响的程序指标主要是延迟和吞吐量两个指标，不同的垃圾回收器对此有不同的侧重也就有了不同的选择
```
