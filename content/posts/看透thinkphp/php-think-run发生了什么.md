---
title: "php think run 发生了什么"
date: 2022-01-09T14:26:05+08:00
draft: false
isCJKLanguage: true
categories:
- 源码阅读
tags:
- 源码
- PHP
---

安装好TP6.0之后，根据官方的文档指引，我们可以运行这个命令来启动一个http服务器

```bash
$ php think run

ThinkPHP Development server is started On <http://0.0.0.0:8000/>
You can exit with `CTRL-C`
Document root is: /var/www/tp6.0/public
```

启动完成之后，我们就可以通过8000端口来访问我们的服务了。

## think

在项目的根目录下，有个名为think的文件。这个文件在项目创建的时候就已经生成了。实际上这个文件可以理解为
命令行模式下的"入口文件"。与之类似的，public目录下的index.php文件是http服务的入口。

think 文件很短小，很简洁。同index.php一样，与6.0之前的版本相比，去除很多常量的设置。

```php
#!/usr/bin/env php
<?php
namespace think;

// 命令行入口文件
// 加载基础文件
require __DIR__ . '/vendor/autoload.php';

// 应用初始化
(new App())->console->run();
```
第1行是Shebang，这意味着在执行think的时候，如果think文件有执行权限的话，是不需要加上php命令的，默认会使用php进行执行

```
$ ./think run
```

第2行是载入composer的autoload文件，这样就把require委托给了autoload去处理。我们只需要use相关的命名空间就可以了。

第3行是初始化了TP的应用类(容器)，然后通过内置的console对象来执行命令行的逻辑。如果是index文件的话，这里就是通过Request对象来处理来http的请求。

总得来说，think文件有两个作用：

1. 加载autoload.php，初始化自动引入的功能
2. 执行命令行的入口函数，即Console对象的run函数

## console

从代码中也可以看到，命令行应该实际上是同个console对象来执行的。那么