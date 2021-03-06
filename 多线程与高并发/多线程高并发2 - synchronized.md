# Synchronized

本篇内容将全方位讲述synchronized关键字的实现，并拓展与其相关的知识。

我们先从一些相关知识讲起。

# CAS

CAS 全称 Compare And Swap (又称Compare And Exchange) / 自旋 / 自旋锁 / 无锁

因为经常配合循环操作，直到修改完成为止，所以泛指一类操作

## 执行原理

![CAS算法执行原理](https://raw.githubusercontent.com/Rooooy7/java-boost/master/img/CAS%E7%AE%97%E6%B3%95%E6%89%A7%E8%A1%8C%E5%8E%9F%E7%90%86.png)

## 细节事项

### 1、保证原子性

​	CAS操作虽然可以做到不需要主动上锁就可以解决原子操作问题。但是其实在执行对比A与V的值时，在底层仍然是加锁了的，不然无法避免指令乱序执行而导致的数据不一致，只是这个操作对我们程序员是无感知的。

​	**这里拓展一下，在CAS的底层实现中，（多核）CPU实际调用了指令 lock cmpxch。事实上也是通过上锁来保证CAS操作的原子性，只不过这个锁实际锁定的是北桥信号而不是内存总线，效率较高些。**

​	具体可以参考文章[CAS你以为你真的懂？](https://zhuanlan.zhihu.com/p/126384164)

### 2、ABA问题

​	ABA问题可以理解为，待修改对象值原本是A被修改为B后又改为了A，那么此时再单纯的对比对象值就没办法确定是否应该更新了。

​	解决办法使用版本号来确定修改版本**（版本号 AtomicStampedReference），如果是基础类型简单值则不需要版本号。**



# 内核态与用户态

这个是操作系统中的一个概念。操作系统为了避免应用程序直接使用内核中的服务，把内核空间与用户空间进行了分割。这也代表着，应用程序（如JVM）向操作系统申请内核服务或资源时，需要花费更多的时间与资源。

## 对synchronized有什么影响呢？

**在早期JDK版本中实现synchronized，是通过向操作系统申请重量级锁的方式实现的。而申请重量级锁就必须通过操作系统的内核的允许再进行系统调用（以中断的方式），如此一来，导致早期synchronied的实现特别低效。**

注：*中断和异常可以使用系统调用来调用内核方法。

**后来出现的偏向锁和轻量级锁（自旋锁）在用户态中处理，所以效率也高了很多。**



# Markword

讲到Java的markword那就必须要讲到Java对象的内存布局了。

当我们创建一个对象，对象信息会被jvm写入内存，对象在内存中的分布如下图**（以下是64位的jvm环境，默认开启压缩指针的情况）**



![markword](https://raw.githubusercontent.com/Rooooy7/java-boost/master/img/markword.png)

## 8字节对齐

**我们称markword+class pointer = 12Byte ，这12字节可以称作object header。它管理着一个对象的各种属性状态。**object header后面的内容则是对象的成员变量。

JVM对内存的管理有8字节对齐的要求，所以我们可以看到1、3图中都因为原始内容不足8*倍而进行了补齐填充。

但是他们的情况有点不一样，一个是在类成员后面补充，一个则是在class pointer和long类型成员间补充。所以对齐填充是有2种情况的，这个也是我偶然发现的。。具体补齐的实现我没有去查看细节，这个不作为重点讨论。

## Markword的位数对应关系

刚刚谈到了**我们称markword+class pointer = 12Byte ，这12字节可以称作object header。它管理着一个对象的各种属性状态。**

现在具体来讲讲Markword中与锁相关的位对应着什么信息。

![markword-bit](https://raw.githubusercontent.com/Rooooy7/java-boost/master/img/markword-bit.png)

我们直接看最后三位，首先，最后两位决定了锁的类型	

-  **0 1** ：两种可能

  -  **0 0 1：**无锁，新对象

  -  **1 0 1：**偏向锁
-  **0 0** ：自旋锁
-  **1 0** ：重量级锁
-  **1 1** ：GC标记



# 锁升级

synchronized优化的过程和markword息息相关

markword上面已经看过了每一位的表示意义，现在来理解synchronized锁升级的过程。

![lock-upgrade](https://raw.githubusercontent.com/Rooooy7/java-boost/master/img/lock-upgrade.png)

- 匿名偏向锁

  -XX:BiasedLockingStartupDelay会设定偏向锁的启动延时（默认4s），当JVM启动时会有很多线程竞争，所以默认情况启动时不直接打开偏向锁，再一段时间后设为偏向锁，此时为匿名偏向锁，线程ID为0。

- 偏向锁

  偏向锁特别的指向了某个线程，看markword-bit图中可以看到写入了线程指针。（一般此时毫无竞争，哪个线程来使用这把锁，锁就指向哪个线程）

- 轻量级锁

  又称自旋锁，偏向锁在轻度竞争时会撤销偏向锁，升级轻量级锁，也有可能从普通对象直接升级为轻量级锁。如果产生了竞争，竞争的线程们会生成一个Lock Record于自己的线程栈中，并用CAS操作将markword设置为指向自己线程的LockRecord的指针，设置成功者得到锁。其余没有获取到锁的线程继续自旋。

- 重量级锁

  竞争加剧，有线程超过10次自旋，（ 由-XX:PreBlockSpin控制自旋次数，默认为10）， 或者自旋线程数超过CPU核数的一半，升级重量级锁，向操作系统申请资源， CPU从3级到0级系统调用，线程挂起，进入等待队列，等待操作系统的调度，然后再映射回用户空间（JAVA1.6之后，加入自适应自旋， JVM自己控制锁升级）
  



**为什么有自旋锁还需要重量级锁？**

> 自旋是消耗CPU资源的，如果锁的时间长，或者自旋线程多，CPU资源会被大量消耗
>
> 重量级锁（ObjectMonitor）有等待队列（WaitSet），所有拿不到锁的进入等待队列，不需要消耗CPU资源

**偏向锁是否一定比自旋锁效率高？**

> 不一定，在明确知道会有多线程竞争的情况下，偏向锁肯定会涉及锁撤销，这时候直接使用自旋锁
>
> JVM启动过程，会有很多线程竞争（明确），所以默认情况启动时不打开偏向锁，过一段儿时间再打开

**如果计算过对象的hashCode，则对象无法进入偏向状态！**

> 轻量级锁重量级锁的hashCode存在与什么地方？
>
> 答案：线程栈中，轻量级锁的LockRecord中，或是代表重量级锁的ObjectMonitor的成员中

# 锁重入

可重入锁，它的可重入性表现在同一个线程可以多次获得锁。

首先synchronized就是可重入锁，也必须是可重入锁。否则子类调用父类的super.m()的操作就无法实现。

而且重入的次数必须被记录，因为解锁需要对应的重入次数。

- 偏向锁/轻量级锁：记录重入次数在线程栈中，每多一次重入LockRecord就会创建多一个，解锁时则删除LockRecord
- 重量级锁：与上面类似的操作，锁重入信息记录到ObjectMonitor



# synchronized vs Lock (CAS)

```assembly
 在高争用 高耗时的环境下synchronized效率更高
 在低争用 低耗时的环境下CAS效率更高
 synchronized到重量级之后是等待队列（不消耗CPU）
 CAS（等待期间消耗CPU）
```



# 锁消除 lock eliminate

```java
public void add(String str1,String str2){
         StringBuffer sb = new StringBuffer();
         sb.append(str1).append(str2);
}
```

我们都知道 StringBuffer 是线程安全的，因为它的关键方法都是被 synchronized 修饰过的，但我们看上面这段代码，我们会发现，sb 这个引用只会在 add 方法中使用，不可能被其它线程引用（因为是局部变量，栈私有），因此 sb 是不可能共享的资源，JVM 会自动消除 StringBuffer 对象内部的锁。

# 锁粗化 lock coarsening

```java
public String test(String str){
       
       int i = 0;
       StringBuffer sb = new StringBuffer():
       while(i < 100){
           sb.append(str);
           i++;
       }
       return sb.toString():
}
```

JVM 会检测到这样一连串的操作都对同一个对象加锁（while 循环内 100 次执行 append，没有锁粗化的就要进行 100  次加锁/解锁），此时 JVM 就会将加锁的范围粗化到这一连串的操作的外部（比如 while 体外），使得这一连串操作只需要加一次锁即可。