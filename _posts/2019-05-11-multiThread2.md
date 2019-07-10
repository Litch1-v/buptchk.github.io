---
layout: post
title: Java多线程学习(二)
tags: Java 多线程
typora-copy-images-to: ../assets/img
typora-root-url: ..
---

## Java线程同步机制

### 锁概述

临界区：锁获得与锁释放之间执行的代码称为临界区

一个锁一次只能被一个线程持有，称为排他锁或者互斥锁（Mutex）

java平台中的锁包含内部锁（通过关键字synchronized）和显示锁（通过java.concurrent.locks.Lock接口的实现类）

#### 锁是如何保证线程安全？

##### 原子性

通过互斥保障，程序在执行临界区间没有其他线程能够访问相应的共享变量（要是synchronized，就是synchronized的锁句柄）

##### 可见性

锁的获得隐含着刷新处理器缓存这个动作，锁的释放隐含着冲刷处理器缓存这个操作，所以可以保证可见性

>锁的互斥性和可见性，可以保证临界区内的代码能够读取到共享数据的最新值。（读取这个共享变量的线程在读取并使用该变量时，其他线程无法更新该变量的值）

##### 有序性

原子性+可见性使得操作线程对其他线程来说，这些共享变量好像是同一时刻更新的，并不用关心操作线程是以什么顺序来更新变量。

> 要保证线程安全需要满足以下两点：
>
> 1. 线程访问同一组共享数据的时候必须使用同一个锁
> 2. 任意一个线程，即使仅仅是读取这组共享数据，没有对其进行更新的话，也需要在读取的时候持有相应的锁

#### 锁的几个概念

##### 可重入锁

一个线程在持有一个锁的时候能否再次（或多次）申请该锁。如果一个线程持有一个锁的时候还能够继续成功申请该锁，那么我们就称该锁是可重入的。可重入锁可以用一个计数器属性来实现，获得锁+1，释放锁-1；可重入锁使得持有锁的线程再次后的锁的开销很小。

##### 锁泄露

一个线程获得某个锁之后，由于程序错误，该锁一直无法被释放，其他线程无法获得该锁。一般出现在显示锁中，未正确处理异常。

### 内部锁：synchronized 关键字

synchronized修饰方法以及代码块。

```
修饰代码块:synchronized(锁句柄){
    //在此代码块中访问共享数据;
}
```

书上的描述

> 习惯上我们直接称锁句柄为锁，锁句柄对应的监视器称为相应同步块的引导锁，线程在执行临界区代码的时候必须持有该临界区的引导锁。如果这个线程没有执行临界区代码，则不需要持有,需要注意的是，synchronized锁住的是对象，但是只有执行临界区代码才需要去获得这个引导锁。

这里需要理解什么是监视器：监视器其实是一种同步机制，Java中每个对象都有一个监视器与之关联，利用这个来实现同步。jvm实现synchronized时实际上是在调用同步方法之前调用Monitor.enter,结束之后调用Monitor.exit;

![绘图1.jpg](https://upload-images.jianshu.io/upload_images/16120453-c841b416076becdc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


内部锁不会导致锁泄露，javac对临界区可能抛出又未捕获的异常进行了处理

### 显式锁：LOCK

显示锁是`java.util.concurrent.locks.Lock`接口的实例，默认实现是ReentrantLock

```java
private final Lock lock=...;//创建一个显示锁
...
lock.lock();//申请锁lock
try{
    //在此对共享数据处理
}fianlly{
    lock.unlock();//这里注意：必须在finally中释放锁,否则可能导致锁泄露
}
```

使用方法大概与synichronized一样。

### 显式锁与内部锁的对比

1. 内部锁基于代码块的锁，使用比较不灵活，而显式锁是基于对象的锁，较为灵活。
2. 调度方面，内部锁只支持非公平，显式锁支持公平和非公平
3. 如果一个内部锁持有线程一直不释放这个锁，所有同步在该锁上的所有其他线程就会一直被暂停使其任务无法进行，但是显式锁可以用lock.tryLock()来尝试获取锁，而不堵塞。
4. 性能方面：内部锁做了优化，在特定情况下可以减少锁的开销：包括锁消除(Lock Elimination)、偏向锁(Biased Lock)和适配锁（Adaptive Lock）。

### 读写锁

读写锁允许多个线程可以同时读取（只读）共享变量，但是一次只允许一个线程对共享变量进行更新，读取共享变量的时候无法更新这些变量，更新变量的时候其他线程无法读取。

#### 读写锁的功能实现

读写锁的功能是通过其==扮演的两种角色==读锁（Read Lock)和写锁（Write Lock）实现的。

读写锁的两种角色

|      | 获得条件                                                 | 排他性                         | 作用                                                         |
| ---- | -------------------------------------------------------- | ------------------------------ | ------------------------------------------------------------ |
| 读锁 | 相应的写锁未被其他任何线程持有                           | 对读线程是共享的，对写线程排他 | 允许多个读线程同时读取共享变量，并保障读线程读取共享变量期间没有其他任何线程能够更新这些共享变量 |
| 写锁 | 写锁未被其他任何线程持有，相应的读锁未被其他任何线程持有 | 对读线程和写线程都是排他的     | 使得写线程能够以独占的方式访问共享变量                       |

#### java平台的读写锁实现

java.util.concurrent.locks.ReadWriteLock 接口，默认实现是java.util.concurrent.locks.ReentrantReadWriteLock。ReadWriteLock接口定义了两个方法：readLock()和writeLock(),分别用于返回**相应**读写锁的实例

> 注意事项：并不是一个ReadWriteLock的实例对应两个锁，而是一个ReadWriteLock接口实例可以扮演两个角色

```java
//使用示例，另外ReentrantReadWriteLock是重入锁，支持锁的降级，即在持有写锁的时候，可以申请读锁。但是
//不支持锁的升级
public class ReadWriteLockExample{
    private final ReadWriteLock rwLock=new ReentrantReadWriteLock();
    private final Lock readLock=rwLock.readLock();
    private final Lock writeLock=rwLock.writeLock();
    public void readExample(){
        readLock.lock();
        try {
            //读取共享变量
        } finally{
            readLock.unlock();
        }
    }
    public void writeExample(){
        writeLock.lock();
        try {
            //读取,更新共享变量
        } finally{
            writeLock.unlock();
        }
    }

}
```

#### 读写锁的选用

> 读写锁的内部实现比其他显式锁复杂很多，所以读写锁适合于
>
> 1. 只读操作比写操作要频繁得多
> 2. 读线程持有锁的时间比较长

只有同时满足上面两个条件时，读写锁才是合适的选择，否则，使用读写锁就会得不偿失。

### 内存屏障

这里的内存屏障其实防止的是重排序，但是这里的重排序并不是对指令来说，而是对处理器对内存的操作。即读内存Load以及写内存Store操作。

```assembly
如x86汇编代码：
mov edx,0f80f802ch;//将内存0f80f802ch中的内容读入edx
mov 0f80f802ch,edx;//将寄存器edx的内容存放到内存中
```

按照可见性保障划分：

加载屏障(Load Barrier),存储屏障(Store Barrier).加载屏障的作用是刷新处理器缓存，存储屏障的作用是冲刷处理器缓存

按照有序性保障划分：

获取屏障（Acquire Barrier）进行后续操作之前必须要先获取数据，禁止读操作与后面的操作重排序，释放屏障（Release Barrier）写操作之前插入，禁止写操作中与前面操作重排序。

```
MonitorEnter
Load Barrier;
Acquire Barrier
临界区
Release Barrier
MonitorExit
Store Barrier
```

### Volatile关键字

volatile关键字用于修饰共享可变变量，没有用final修饰的共享变量或者静态变量。volatile变量不会被编译器分配到寄存器进行存储，读写操作都是内存访问操作。

#### volatile的作用

> 保障可见性
>
> 保障有序性
>
> 保障long/double型变量读写操作的原子性

对volatile变量的赋值操作，其右边表达式中只要涉及共享变量（包括volatile变量本身），那么这个赋值操作就不是原子操作。

对于volatile变量的写操作，Java虚拟机会在该操作之前插入一个释放屏障，并在操作之后插入一个存储保障。

```java
//写的时候前面的操作防止重排序，写后保持可见性
sharedA=1;
Realease Barrier;//禁止写操作与前面的任何读写操作重排序
volatileVar=true；
Store Barrier;//冲刷处理器缓存
```

对于volatile变量的读操作

```java
//读前保证可见性，读后防止重排序
Load barrier；刷新处理器缓存
localVar=volatileVar；//读变量操作
Acquire Barrier；//禁止读操作与之后的操作重排序
localA=sharedA;//普通变量的读写操作
```

> 注意点：
>
> 如果被修饰的变量是一个数组，volatile只能保证对数组引用本身的操作（读取数组引用，更新数组引用）起作用，同样地，对于引用型volatile变量，volatile关键字只能保证线程读取到一个指向对象的相对新的内存地址。

volatile不会导致上下文切换，所以开销要比锁小一点。

#### volatile的典型应用场景

1. 使用volatile变量作为状态标志。也就是说其中一个线程来管理这个变量，其他线程读取变量作为计算的依据（不进行写操作），这样只要这个线程只采用写操作（可以是一个赋值，右边可以是自己，但不能是其他共享变量（保证其他线程对这个变量只有读操作）），就可以保证安全。
2. 多个线程共享一组可变状态变量的时候，通常我们需要用锁来保证这些变量的更新操作的原子性。
3. 使用volatile代替锁，多个线程共享一组可变状态变量的时候，我们需要使用锁来保障对这些变量的更新操作的原子性。而我们可以把这些变量封装成一个对象，那么对这些状态变量的更新操作就可以通过创建一个新对象并将该对象的引用赋值给响应的引用型变量。（原则：只写或只读，而不读后写）
4. 实现简易的读写锁。变量声明为volatile。写操作声明为synichronized，写线程就可以互斥写，但是读线程可能读到的共享变量不是最新值。
