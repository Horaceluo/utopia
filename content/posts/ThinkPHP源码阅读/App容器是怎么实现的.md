---
title: "App容器时怎么实现的"
date: 2022-01-16T17:18:00+08:00
draft: true
isCJKLanguage: true
categories:
- 源码阅读
tags:
- 源码
- PHP
---

# 前言

从5.0版本开始，ThinkPHP已经开始逐步使用依赖注入了。从那时开始，就可以看到，框架的核心入口类App开始继承一个名为Container的类。到了6.0，又更加完善了这个Container类，实现Psr(PSR-11)的ContainerInterface。那么什么时容器？容器在框架里到底起到什么样的作用？ThinPHP又时怎么样实现容器的？我们来一探究竟。

## 控制反正与依赖注入

通常来说，我们要一个类里面调用另外一个类的方法时，最直接的方式就是，在类里面通过new关键字生成一个类的实例对象，再通过这个对像来调用目标方法。

对于简单的应用而言，直接new可能没有多大的影响。但是对于大型应用而言，这就会使得几十甚至几百、几千个类中形成错综复杂的依赖关系。而且直接在方法里面new一个对象，很可能导致该方法无法进行单元测试。例如，A 类型 的a方法new了一个B类，当执行a方法的单元测试时候，就一定得保证B类是正常的，不然就会导致a方法测试失败。

那要怎么做才能改变这种局面呢？一个比较常见的方法就是“控制反转”，而依赖注入，就是控制反转的一种实现。

### “反转”的关键在于类是如何获得依赖

A类需要B类，如果是A自己去寻找、获取、生成B类，例如直接在类里面new B，这种就是强依赖的关系。如果A需要B类，且这个类并不是自己去new获取的，而是外界通过一定方式将依赖类传输给A的，那么这种依赖关系就“反转”了。原本是A需要控制B的生成，现在不需要了，A只需要使用B就行，无效关系B如何产生的了。

### 依赖的管理者

当对象自己不需要去“new”其他对象的时候，那么对象的依赖关系由谁去处理呢？在ThinkPHP6.0中，App就是这样的一个角色，它可以管理应用中各种的类型实例化，并将这些实例注入到依赖他们的类或者方法中。


## Container

App对象实际上并只是一个容器的角色，它还承担着应用的初始化等很多的功能。我们要了解它的容器功能，更好的方式是看它的父类，Container是怎么实现的。

### Container实现的接口

Container一共实现了4个接口，他们分别是：

- Psr\Container\ContainerInterface Psr的容器管理接口，有get和has两个函数
- ArrayAccess PHP的数组访问接口，这个意味着，App类可以像数组一样访问
- IteratorAggregate PHP的外部迭代器接口，意味着可以使用foreach来循环输出
- Countable PHP的count接口，App对象可以作为count()函数的输入

从Container实现的接口来看，Container很重要的一个功能是实例的获取、访问。ThinkPHP的Container也提供很多的方式(get、数组访问)来获取容器里的实例。

### Container实例的来源

Container的实例来源以前几个

- bind 方法，可以将实例，闭包绑定到容器里
- instance 方法，可以将类实例绑定到容器里
- make 方法，创建实例并返回实例，创建之后也会将实例绑定到容器

bind和instance方法比较类似，实际上他们绑定的对象和方式是有区别的。bind方法可以将实例、闭包绑定到容器。而instance方法只是把类实例绑定到容器。所以适用范围上，bind是大于instance方法的。在bind的实现里，如果绑定的是个对象，其实就是调用instance方法来进行绑定。

#### bind 方法
```php
public function bind($abstract, $concrete = null)
{
    if (is_array($abstract)) {
        foreach ($abstract as $key => $val) {
            $this->bind($key, $val);
        }
    } elseif ($concrete instanceof Closure) {
        $this->bind[$abstract] = $concrete;
    } elseif (is_object($concrete)) {
        $this->instance($abstract, $concrete);
    } else {
        $abstract = $this->getAlias($abstract);
        if ($abstract != $concrete) {
            $this->bind[$abstract] = $concrete;
        }
    }

    return $this;
}
```

#### instance 方法
```php
public function instance(string $abstract, $instance)
{
    $abstract = $this->getAlias($abstract);

    $this->instances[$abstract] = $instance;

    return $this;
}
```

### $bind和$instance

$bind 和 $instance 是容器两个比较重要的属性。$bind是用于存在绑定的“标识”，例如绑定的类名、闭包。$instance则是存储类的实例。

App类里默认绑定的类

```php
/**
     * 容器绑定标识
     * @var array
     */
    protected $bind = [
        'app'                     => App::class,
        'cache'                   => Cache::class,
        'config'                  => Config::class,
        'console'                 => Console::class,
        'cookie'                  => Cookie::class,
        'db'                      => Db::class,
        'env'                     => Env::class,
        'event'                   => Event::class,
        'http'                    => Http::class,
        'lang'                    => Lang::class,
        'log'                     => Log::class,
        'middleware'              => Middleware::class,
        'request'                 => Request::class,
        'response'                => Response::class,
        'route'                   => Route::class,
        'session'                 => Session::class,
        'validate'                => Validate::class,
        'view'                    => View::class,
        'filesystem'              => Filesystem::class,
        'think\DbManager'         => Db::class,
        'think\LogManager'        => Log::class,
        'think\CacheManager'      => Cache::class,

        // 接口依赖注入
        'Psr\Log\LoggerInterface' => Log::class,
    ];
```

$bind相当于一个占位符，先把某些标识和类名绑定起来，但是不进行实例化操作。等通过标识从容器里访问类的时候就取出类名，并实例化。实例化之后，如果围显示指定，就会将实例存储到$instance里。

### 依赖注入

假设有以下两个类，Class A, Class B，其中Class B通过构造器，注入Class A。

```php
class A
{
    public function a()
    {

    }
}

Class B
{
    public function __cunstruct(A $a)
    {

    }
}
```

如果是手动实例化，需要先将A类实例化，再把A的实例作为B的构造器参数来实例化B。过程如下面代码所示：
```php
$classA = new A();
$classB = new B($classA);
```
这样实际上还是需要人花比较多精力去逐个实例化和逐个注入。但是如果使用容器进行实例化就不需要这样操作了。

```php
$classB = app()->make(B::class);
```

通过容器提供的make方法，可以省略掉手动实例化的步骤，让容器取代人去管理类的依赖关系。这样，即便B类的构造器增加其他类的依赖，在实例化的地方依旧不需要修改代码。

```php
class A
{
}

class C
{
}

Class B
{
    public function __cunstruct(A $a, C $c)
    {

    }
}

// 实例化B
$classB = app()->make(B::class);
```

B类增加了C类的依赖, 在实例化的地方，可以保持不变。

#### make

对于容器来说，它是怎么知道实例化的类需要哪些参数的？--反射。

正是通过反射，在实例化之前，make方法会先解析类的构造函数，逐一分析构造函数的参数。通过获取参数的类型提示，容器就可以知道参数绑定的是哪个类。

例如上面的例子，通过B类的反射，容器就能知道B类型的构造函数第一个参数是A类的实例，然后就会去寻找容器里是否已有A类的实例。如果没有，重新获取A类的反射，由于A类的构造器没有参数，所以容器就能将A类实例化并把A的实例化传入B的构造器里。如果A类型还有参数，容器就会继续“反射-实例化”的过程，直到所有的参数都满足实例化要求。

正是由于有这个不管递归的实例化过程，如果A类，B类相互依赖了，就会造成循环依赖，导致实例化错误。

```php
Class A {
    public function __cunstruct(B $b) {}
}

class B {
    public function __cunstruct(A $a) {}
}

// A和B循环依赖，无法实例化B
$classB = app()->make(B::class);
```
