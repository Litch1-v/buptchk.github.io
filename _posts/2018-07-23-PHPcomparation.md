---
layout: post
title: 关于PHP的弱类型比较
tags: CTF Web安全

---



前几天做到qctf的题目，里面有一题就考察了php的弱类型比较这个知识点，发现自己还不是特别理解，整理了一波.

参考博客：`https://www.cnblogs.com/Mrsm1th/p/6745532.html`

php中的比较有两种，一种是== ，还有一种是=== 

=== 在判断时，会先比较两个比较对象的类型是否相等

== 在判断时，会先将两个比较对象转换成相同类型，之后再进行比较。

#### 字符串和数值比较

**一个数值和字符串进行比较的时候，会将字符串转换成数值** 

关于字符串转换成数值  php手册是这样写的：

```
当一个字符串欸当作一个数值来取值，其结果和类型如下:如果该字符串没有包含'.','e','E'并且其数值值在整形的范围之内
该字符串被当作int来取值，其他所有情况下都被作为float来取值，该字符串的开始部分决定了它的值，如果该字符串以合法的数值开始，则使用该数值，否则其值为0
```

所以这类比较的关键是字符串的开始部分，一直比较到不合法的字符,不合法的部分会被舍弃，其中e,E作为科学计数法被识别。

以下是测试代码

```php
var_dump('php'==0);//  bool(true)
var_dump('1php'==1);// bool(true)
var_dump('1ephp'==1);// bool(true)
var_dump('1e0php'==1);// bool(true)
var_dump('php10'==0);// bool(true)
```

in_array() 函数检查数组中是否存在某个值 ，如果没有设置第三个参数 strict 的值为 TRUE ，则也会引发转换问题。

#### 字符串和布尔类型比较

通过上图可以看出，所有非‘0’的字符类型在与布尔类型进行比较时，均等于true。

测试代码

```php
var_dump('1'==true);// true
var_dump('0'==true);// false
var_dump('php'==true);// true
var_dump("false"==true);// true
```

这里就会存在json_decode和unserialize的bool欺骗，测试代码

```php

```

 

```php

```

#### 数组与其他类型的比较

值得注意的点是 数组和null的比较为true

同时涉及到数组得到还有常见的strcmp函数，当传入一个数组与字符串进行比较时，返回值时0。（`int strcmp ( string $str1 , string $str2 )`,str1是第一个字符串，str2是第二个字符串，如果str1小于str2，返回<0,如果str1>str2,返回>0,两者相等返回0 ）

#### 关于NULL

NULL和false比较为true（**NULL,0,”0″,array()使用==和false比较时，都是会返回true的** ），在php中利用md5加密函数进行加密，如果传入的值为数组，则返回null

#### 关于AND

整理时恰好看到，顺便记下来 可以绕过and后面的is_numeric();

```php
$a=$_GET['a'];
$b=$_GET['b'];
$c=is_numeric($a) and is_numeric($b);
var_dump(is_numeric($a));
var_dump(is_numeric($b)); 
var_dump($c);  //$b可以不是数字，同样返回true
$test=true and false;
var_dump($test); //返回true
```

