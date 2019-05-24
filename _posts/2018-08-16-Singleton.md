---
layout: post
title: 设计模式之单例模式(Singleton)
tags: Java开发
---

设计模式之单例模式 （Singleton)



#### 什么是在单例模式？

单例模式确保某个类只有一个实例，而且自行实例化并向整个系统提供这个实例。 单例对象通常作为程序中的存放配置信息的载体，因为它能保证其他对象读到一致的信息。例如在某个服务器程序中，该服务器的配置信息可能存放在数据库或 文件中，这些配置数据由某个单例对象统一读取，服务进程中的其他对象如果要获取这些配置信息，只需访问该单例对象即可。这种方式极大地简化了在复杂环境 下，尤其是多线程环境下的配置管理 

#### 在什么时候使用单例模式？

1. 想确保任何情况下都绝对只有一个实例
2. 想在程序上表现出“只存在一个实例”

#### 实例代码

```java
public class Singleton{
    private static Singleton singleton = new Singleton();//类Field，加载类时便生成这个实例
    private Singleton(){
        
    }//构造函数是private的，禁止从类外部调用构造函数
    public static Singleton getInstance(){
        return singleton;
    }//便于程序从Singleton类外部获取这个实例。
    
}
```

#### 应用场景

从properties中读取配置信息用于数据库的连接，这时候，为了保证读取到的配置信息的一致性，就可以使用单例模式。然后通过访问这个单例来获取信息，得到连接。

```java
public class ConnectionTest {

    private static String driver;
    private static String dburl;
    private static String user;
    private static String password;
    private static final ConnectionTest connectionTest=new ConnectionTest();//用来存储数据库信息
    private Connection connection;
    static {
        Properties properties=new Properties();
        try {
            InputStream inputStream=ConnectionTest.class.getClassLoader().getResourceAsStream("jdbcTest.properties");
            properties.load(inputStream);
        }
        catch (Exception e){
            e.printStackTrace();
        }
        driver=properties.getProperty("driver");
        dburl=properties.getProperty("dburl");
        user=properties.getProperty("user");
        password=properties.getProperty("password");
    }//类加载的时候就初始化
    private ConnectionTest(){

    }//单例模式，保证只有一个ConnectionTest对象
    public static ConnectionTest getInstance(){
        return connectionTest;
    }
    //连接方法
    public Connection makeConnection(){
        try {
            Class.forName(driver);
            connection = DriverManager.getConnection(dburl,user,password);
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
        return connection;
    }
}
```

