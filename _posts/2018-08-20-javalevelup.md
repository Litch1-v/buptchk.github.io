---
layout: post
title: java升级之路
tags: Java开发
---


java升级之路，看了一波《疯狂Java讲义精粹 》，整理了一些自己感觉有用的东西出来,持续更新



### 分支结构

使用switch语句时，有两个值得注意的地方：第一个地方是switch语句后的expression表达式的数据类型只能是byte、short、char、int四个整数类型和==枚举类型==（这里可以体现出枚举类型的好处）；第二个地方是如果省略了case后代码块的break；将引入一个陷阱。 



### 成员变量

在《疯狂java讲义精粹》里对成员变量的讲解，我觉得是非常赞的。所以重新拿出来说一下

成员变量被分为类Field和实例Field两种，定义Field时没有static修饰的就是实例Field，有static修饰的就是类Field 。类Field从这个类的准备阶段开始存在，直到系统完全销毁这个类，类Field的作用域域这个类的生存范围相同。而实例Field与对应实例的生存范围相同。

关于成员变量初始化和内存中的运行机制：系统会在第一次使用一个类的时候加载这个类，并初始化这个类。在类的准备阶段，系统将会该类的Field分配内存空间，并指定默认初始值。类初始化会为这个类创建一个类对象，在**堆内存**，类Field也会初始化保存在**堆内存**中。而创建实例对象之后的分布如下图示例：

![]({{site.baseurl}}\public\img\post\java01.jpg)

栈内存存放的是一块堆内存的空间地址，堆内存保存的是对象的真正数据，想要开辟堆内存空间，只能靠关键字new来开辟

### 隐藏和封装

如果一个Java类的每个Field都被使用private修饰，并为每个Field都提供了public修饰setter和getter方法，那么这个类就是一个符合JavaBean规范的类 

如果某个类主要用做其他类的父类，该类里包含的大部分方法可能仅希望被其子类重写，而不想被外界直接调用，则应该使用protected修饰这些方法 

> 为什么我们经常在声明变量的时候，声明变量为接口类型

因为这里要是声明成接口类型，可以屏蔽掉调用这个实例的某些方法时是如何实现的，我们不需要关注其实现。但是如果这个类实现了除了接口定义之外的其他一些自己的方法，我们需要使用就需要声明成具体实现

### 继承与组合

继承要表达的是一种“是（is-a）”的关系，而组合表达的是“有（has-a）”的关系 。

### IO流

Java的输入流主要由InputStream和Reader作为基类，而输出流则主要由OutputStream和Writer作为基类。它们都是一些抽象基类，无法直接创建实例。 

字节流主要由InputStream和OutputStream作为基类，而字符流则主要由Reader和Writer作为基类 。

#### 内部类

内部类提供了更好的封装，可以把内部类隐藏在外部类之内，不允许同一个包中的其他类访问该类 。

##### 关于静态内部类，非静态内部类

静态成员不能访问非静态，所以外部类的静态方法，静态代码块不能访问非静态内部类。静态内部类不能访问外部类的实例成员

需要注意的是，非静态内部类是可以访问外部类的private成员。但是外部类不可以访问非静态内部类的private成员

当在非静态内部类的方法内访问某个变量时，系统优先在该方法内查找是否存 在该名字的局部变量，如果存在就使用该变量；如果不存在，则到该方法所在的内部类中查找是否存在该名字的成员变量，如果存在则使用该成员变量；如果不存在，则到该内部类所在的外部类中查找是否存在该名字的成员变量，如果存在则使用该成员变量；如果依然不存在，系统将出现编译错误：提示找不到该变量。 （总结：查找顺序：局部变量--内部类变量-外部类变量）

内部类的上一级程序单元是外部类，使用static修饰可以将内部类变成外部类相关，而不是外部类实例相关。 

##### 关于在外部类以外的访问

在内部类以外的地方使用内部类变量时`OuterClass.InnerClass varName`

在外部类以外的地方创建非静态内部类``OuterInstance.new InnerConstructor()`（即非静态内部类的构造器必须通过外部类对象调用）

在外部类以外的地方创建静态内部类`new OuterClass.InnerConstructor()`

##### 匿名内部类

匿名内部类适合用于创建那些仅需要一次使用的类，示例如下

```java
public class Device
{
    private String name;
    public Device(String name){
        this.name=name;
    }
    public getName(){
        return this.name;
    }
}
public class AnonymousInner
{
    public void test(Device d){
        System.out.println("设备的名字为:"+d.getName())
    }
    public static void main(String[] args)
    {
        AnonymousInner ai=new AnonymousInner;
        ai.test(new Device("电子示波器"){
        })
    }
}
```

##### 关于回调

举例：有一个Teacher类和一个Programmable接口，都含有work方法。设计一个类既是Teacher的子类，又实现了Programmable接口。这时候可以设计一个内部类继承Programmable接口，实现work方法。再写一个回调方法返回这个接口的实现内部类

```java
public Programmable getCallbackReference()
{
    return new Closure();
}
```

这样子，我们想调用Teacher的work方法，只要ClassInstance.work();

想调用Programmable的work方法只要ClassInstance.getCallbackReference().work();

