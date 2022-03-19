---
title: "TP源码：php think run 发生了什么"
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

从代码中也可以看到，命令行实际上是通过Console对象来执行的。Console对象主要是用来管理命令行的各种命令的初始化、注册、运行。

### console的构造函数

```php
public function __construct(App $app)
    {
        $this->app = $app;

        $this->initialize();

        $this->definition = $this->getDefaultInputDefinition();

        //加载指令
        $this->loadCommands();

        $this->start();
    }
```

构造函数主要完成了这几件事情

- 初始化（确认APP容器已初始化、构造Request对象[模拟http环境]）
- 设置初始化的默认选项(--help,--version等)
- 加载命令(默认的命令：help、make等，配置文件里的自定义命令)
- 执行应用启动的回调

## run

console对象初始化完成之后，就可以运行run方法了。run方法的主要完成两件事情

1. 识别需要运行的命名
2. 构造Input和Output对象

完成这两件事情之后，把Input和Output对象注入到命令绑定的Command类中，最后通过Command类的run方法执行真正的命令行业务逻辑。

因此，对于`php think run` 这个命令来说。最后的启动http服务的逻辑实际上是在'run'这个命令绑定的Command类的run方法里。

## RunServer

think\console\command\RunServer 这个类就是run命令绑定的命令行类。这个类只有两个方法。

- configure()
- execute(Input $input, Output $output)

configure() 方法是为该命令注册命令的名称，选项，参数，说明等基础信息。这个方法会在Command对象构造器里执行。执行之后，RunServer就可以通过getName(),getDefinition()等方法获得这些配置信息。

execute() 方法是业务逻辑执行的方法。前面提到过，执行业务逻辑的是通过Command的run()方法执行的。到这里，怎么就变成了execute()方法了呢？

实际上，RunServer是继承于Command抽象类，run()是Command的方法，最后在run()方法里面会调用execute()方法。

RunServer的execute方法也很简单

```php
    public function execute(Input $input, Output $output)
    {
        $host = $input->getOption('host');
        $port = $input->getOption('port');
        $root = $input->getOption('root');
        if (empty($root)) {
            $root = $this->app->getRootPath() . 'public';
        }

        $command = sprintf(
            'php -S %s:%d -t %s %s',
            $host,
            $port,
            escapeshellarg($root),
            escapeshellarg($root . DIRECTORY_SEPARATOR . 'router.php')
        );

        $output->writeln(sprintf('ThinkPHP Development server is started On <http://%s:%s/>', $host, $port));
        $output->writeln(sprintf('You can exit with <info>`CTRL-C`</info>'));
        $output->writeln(sprintf('Document root is: %s', $root));
        passthru($command);
    }
```

不难看出，实际上`php think run`启动的http服务器，实际上是php自带的[http服务器](https://www.php.net/manual/en/features.commandline.webserver.php)。所以我们如果在直接启动php自带的服务器其实也可以达到同样的效果

## 总结

从最后的结果来看，似乎`php think run`这个命令似乎没有多大存在的必要。于本人而言，实际上`php think run`要比`php -S 127.0.0.1:8080` 这样更容易接受一点。通过think执行可以省略一些基本参数，启动过程更为顺畅和简洁。

对于启动一个http服务器而言，Console这个类的作用实际上并不是很大。它更大意义是在其他命令上。例如队列、定时、单元测试等需要依赖TP框架的命令而言，它能帮助开发者省去手动加载框架的麻烦，也可以屏蔽一些命令行运行和http服务运行的差异。使得我们开发命令行应用也可以像开发http应用一样使用框架的特性，实现代码服用，提高开发效率。