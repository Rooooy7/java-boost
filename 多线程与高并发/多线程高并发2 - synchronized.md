---

---

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

## 那么这个对synchronized有什么影响呢？

**在早期JDK版本中实现synchronized，是通过向操作系统申请重量级锁的方式实现的。而申请重量级锁就必须通过操作系统的内核的允许再进行系统调用（以中断的方式），如此一来，导致早期synchronied的实现特别低效。**

注：*中断和异常可以使用系统调用来调用内核方法。

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

现在具体来讲讲每一位都对应着这个对象的什么信息	

![markword-bit](https://raw.githubusercontent.com/Rooooy7/java-boost/master/img/markword-bit.png)





