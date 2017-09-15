---
layout: post
title: 从报警看 PHP7.0.x 到 PHP7.1.x 的改变
categories: PHP7.1
description: migrating from PHP7.0.x to PHP7.1.x
keywords: PHP 多进程 pcntl
---


PHP消费进程经常报 segfault 错误，导致load异常。输出 core 文件使用 gdb 工具分析之后发现是 phpredis 扩展进行 stream write 操作时空指针错误。最为偷懒的方案就是想直接升级 phpredis 3.0.0 to phpredis 3.1.3（中修复了 Fix memory leak and potential segfault 和 Fix Redis/RedisArray segfaults ）

但是，单纯升级 phpredis 之后会在 redis 调用频繁的逻辑中直接报500错误，3.1.3 的扩展与 PHP 7.0.9 存在兼容性问题。所以打算直接升级 PHP7，毕竟 7.1 的速度较 7.0 又有了提升

## 发现问题

本次升级本以为是小版本升级。偷了个懒，省略了文档的查看。so 在开发环境升级完毕之后出现了一系列的报警提示。重新读下官方文档说明吧。

## 参考

> http://php.net/manual/en/migration71.php

> https://wiki.php.net/rfc#php_71

# 从报警看 PHP7.1 升级变动


## $this 相关报错

$this 作为关键字被严格限制了使用环境

```php

function test($this) { // Fatal error: Cannot use $this as parameter
}

function test() {
	var_dump($this): //Fatal error:  Uncaught Error: Using $this when not in object context
}

tatic $this;// Fatal error: Cannot use $this as static variable

global $this;// Fatal error: Cannot use $this as global variable

$this = 123;// Fatal error:  Cannot re-assign $this

unset($this);// Fatal error: Cannot unset $this

$a = 'this';
$$a = 1;//Fatal error:  Uncaught Error: Cannot re-assign $this

extract(["this" => 123]); //Fatal error:  Uncaught Error: Cannot re-assign $this 

```

## Fatal error: A void function must not return a value

```php

function shouldReturnNothing(): void {
	return 123;
}

```

## Fatal error: void cannot be used as a parameter type

```php

function test(void $a) {
}

```

## Fatal error:  Uncaught ArgumentCountError: Too few arguments

```php

function test($a) {
}
test();

```

该变化主要是将之前版本的 Warning: Missing argument 提示升级为 Fatal error: Uncaught ArgumentCountError 。 开发者必须严格检查调用方法参数是否正确


##  Warning:  A non-numeric value encountered
```php

$a = '';
var_dump($a + 2);

$a = 'wasd';
var_dump($a + 2);

```

##  Notice:  A non well formed numeric value encountered
```php
$a = '123sadf';
var_dump($a + 2);

```

## mcrypt 

mcrypt 扩展已经过时了大约10年，并且用起来很复杂。因此它被废弃并且被 OpenSSL 所取代。 从PHP 7.2起它将被从核心代码中移除并且移到PECL中。

需要注意 OpenSSL uses PKCS#7 and mcrypt uses PKCS#5 

> https://paragonie.com/blog/2015/05/if-you-re-typing-word-mcrypt-into-your-code-you-re-doing-it-wrong 
