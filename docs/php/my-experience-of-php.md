# 关于PHP, 我学到了什么

## 自动类加载 (auto-load)

自动类加载说白了就是自动 `include` / `require` php 文件，这里以: `Aliyun_Log_Client` 为例说明。

如果想使用 `Aliyun_Log_Client` 这个类， 需要先 :

```php
require_once('aliyun/Aliyun/Log/Client.php');
```

这里你不仅需要手动 `require`, 还需要知道 `php` 文件的相对位置。

但是如果实现了自动类加载呢， 你只需要 

```php
use Aliyun_Log_Client;
```

我们知道 `PHP` 执行的本质还是需要将依赖的 `php` 文件都加载在一起，那么自动类加载是如何做到的呢？

主要需要做两件事：

1. `spl_autoload_register` 注册一个解析 `class` 名的函数
2. 根据 `class` 名找到对应的 `php` 文件并调用 `require`

<!-- more -->

这里看 `Aliyun_Log` 是如何实现的:

**Log_Autoload.php**

```php
<?php
/**
 * Copyright (C) Alibaba Cloud Computing
 * All rights reserved
 */

$version = '0.6.0';

function Aliyun_Log_PHP_Client_Autoload($className) {
    $classPath = explode('_', $className);
    if ($classPath[0] == 'Aliyun') {
        if(count($classPath)>4)
            $classPath = array_slice($classPath, 0, 4);
        $filePath = dirname(__FILE__) . '/' . implode('/', $classPath) . '.php';
        if (file_exists($filePath))
            require_once($filePath);
    }
}

spl_autoload_register('Aliyun_Log_PHP_Client_Autoload');
```

然后你需要在代码执行之前 `require` 这个文件。

*懂得了这个，你就可以组织自己的代码结构，定义自己的命名习惯，并且可以很方便的给别人使用.*

## 魔术方法 (magic method)

如: `__get()`, `__set()`. `__isset`, `__unset`, `__call`, `__callstatic` 这些方法，在实现框架代码时是很常见且常用的，知道什么时候这些魔术方法会被调用，以及它能用来做什么事情，对于一个 `PHPer` 来说很标配。以 `__get()`, `__set()` 来例:

```php
<?php

class Object {
    public $data;
  
    public function __get($name)
    {
        if (isset($this->data[$name]) || array_key_exists($name, $this->data)) {
            return $this->data[$name];
        }

        return null;
    }
  
    public function __set($name, $value)
    {
        $this->data[$name] = $value;
    }
    
    public function __isset($name)
    {
        return $this->__get($name) !== null;
    }
}
```

这样，`Object` 具有的属性就可以动态扩展了， 如:

```php
$ob = new Object();
$ob->name = 'mike';
$ob->age = 25;
```

那它有什么用武之地呢？ 

它可以作为一个基类，来封装对数据库的操作，实现自己的 `ORM` 框架, 如:

**DB.php**

```php
<?php
class DB {
    public static function insert($collectionName, $data)
    {
        // 创建 mongodb 的连接 $mongodbConnection
        $mongodbConnection->getCollection($collection)->insert($data);
    }
}
```

**Model.php**

```php
<?php
  
class Model extends Object {
    pubic function save()
    {
        $className = get_class();
        $collectionName = strtolower($cl);
        // 根据 $collectionName 可以连接选择数据库 table(mysql), collection(mongodb)
        DB::insert($collectionName, $this->data);
    }
}
```

**Test**

```php
<?php

class User extends Model {}

class Order extends Model {}

$user = new User();
$user->name = 'mike';
$user->age = 25;
$user->languages = ['php', 'golang'];
$user->save();  // 向 mongodb 的 user collection 中插入一条数据

$order = new Order();
$order->orderNumber = 'P201708220001';
$order->price = '100';
$order->name = '爱茜茜里甜品';
$order->save(); // 向 mongodb 的 order collection 中插入一条数据
```

当然这里说的只是一个例子， 真正的框架要做的事很多，但是原理性的东西就是那些。

## `static` 与 `self`

- `self` 正如其名字，指向的是当前类

- `static` 指向的则是实际调用类


看下面的示例:



```php
<?php

class A {
    public static function say()
    {
        static::hint();
        self::info();
    }

    public static function hint()
    {
        echo "hint A \n";
    }

    public static function info()
    {
        echo "info A \n";
    }
}

class B extends A {
    public static function hint()
    {
        echo "hint B \n";
    }

    public static function info()
    {
        echo "info B \n";
    }
}


$test = new B();
$test->say();
```

最后的输出结果为:

```shell
hint B   // 使用 static 调用时会调用父类同名方法
info A   // 使用 self 只会调用当前类，而不是调用类的方法
```



清楚了 `self`, `static`的作用方法，在看一些框架代码，或者自己写具有层级关系的代码时，考虑一些方法的作用域时，就可以通过`static` , `self` 来决定子类是否可以覆盖父类同名方法了。

## `php-cli` 与 `php-fpm`

- `PHP CLI` 是命令行接口, 是位于创建独立的程序 （如：在命令行运行一个php文件，执行一些耗时的操作）
- `PHP CGI` 是 `PHP CGI` (通用网关接口) 的一种实现，主要是用于 web 应用。

举一些常见的区别:

- `pcntl_fork` 命令只能做 `cli` 模式，不能再 `fpm`模式
- `$_SERVER`，`$_POST` 这些 这些全局变量就只能在 `fpm` 模式下有效， 在 `cli` 下不存在

## `composer`

`composer`  是 `PHP`的一个包依赖管理， 定义了一下自动加载规范，可以下载,管理依赖.

## 匿名函数， 闭包

## `trait` 

用于定义一些方法集，可以被其它类使用，感觉主要是为了解决不能多继承的问题

## 命名空间 (`namespace`)

与文件系统中的目录结构类似，可以用来区分同名文件
