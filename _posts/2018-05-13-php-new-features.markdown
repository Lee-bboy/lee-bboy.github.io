---
layout:     post
title:      "PHP7.2有哪些新特性？"
subtitle:   ""
date:       2017-05-13 16:00:00
author:     "Lee"
header-img: ""
tags:
    - PHP
---

> 我们知道php被称为“世界最好的语言“，可见人们对其是又爱又恨。我是其中一位开发者，但我对php是绝对地喜爱。我对php 了如指掌。自从php7.2发布以来，我更加喜欢这门语言。让我们看最新版本给我们带来哪些精彩。  

**最重要的安全**



7.2版本提供了一些非常必要的安全性改进。

停止使用sha1() or md5()，请使用：


```
password_hash('password', PASSWORD_ARGON2I)
```


使用argon2i算法还支持自定义模式：


```
$options = [

    'memory_cost' => PASSWORD_ARGON2_DEFAULT_MEMORY_COST,

    'time_cost' => PASSWORD_ARGON2_DEFAULT_TIME_COST,

    'threads' => PASSWORD_ARGON2_DEFAULT_THREADS,

];

password_hash('password', PASSWORD_ARGON2I, $options);
```



argon2算法解决了我个人的现有算法的缺点，在他们设计的最高内存填充率。



libsodium库现在正式作为PHP核心的扩展。我一直在等待这样的一段时间了。



Mcrypt被取消



mcrypt密码库扩展已正式取消。PHP的开发小组说，mcrypt大大抑制PHP语言的发展，越来越像“老软件。”



对SSL / TLS（安全套接字层/传输层安全）常数进行了改进。



改进的语言特性



还有其他的更新，用来帮助解决一些开发者关于PHP语言的改进和建议。我们一起来看看。



PHP7.2在调用count（）函数时，它接收一个参数为一个标量函数，如果参数为空，或者一个对象，将返回未实现接口的警告信息。



关于对象类型声明修复的情况，以前开发者不能声明一个函数需要传递一个对象作为参数或声明一个函数应该返回一个对象。PHP7.2可以使用object作为一个参数类型和返回类型声明。



hashcontext对象将哈希扩展使用对象，而不是使用资源。



在使用对象/数组模型解决了与Zend引擎数字key转换的问题。



在以前的开发实例中，哈希数组的Key可以包含数字和字符串，而对象哈希表是整数的索引。在这种情况下，导致PHP代码找不到key。



PHP 7.2对此作了修复，数组或对象哈希表的key会自动转换为适当的类型，所以数字字符串属性名对象会成为整数数组中的key，反之亦然，解决了无法访问的性能问题。
