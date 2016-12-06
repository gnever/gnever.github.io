---
layout: post
title: php7新特性
categories: php
description: php5.4 升级 php7 新特性，同时补充跳过的 php5.5 版本新特性
keywords: php php7新特性
---

全站升级 PHP7 还是比较靠后的，一直等到 redis 2016/06/10 redis出来官方扩展之后才陆续进行。升级过程中出现的问题都记录在了笔记上，格式太乱了，转移到 markdown 比较费力，加个 todo ...

升级是从 5.4.41 版本升级的，中间遗漏了 5.5 这个笔记大的改版。所以在此将两个版本中有意思的几个特性一并补全吧

# PHP 5.5.x 新特性

## yield

> 这个很有意思，单独开一篇来学习。//todo 此处应该增加跳转

## finally

Java/C#等语言都有，对于之前的 PHP 来说，在需要处理异常的时候可以这么写


```php

function test() {
    try {
        //xxxx
    } catch (Exception $e) {
        doAction();
        throw $e;
    }
    doAction();
}

```

但是要写两遍 doAction() 。如果使用 finally 就可以避免这个情况。上例可以写为：

```php

function test() {
    try {
        //xxxx
    } finally {
        doAction();
    }
}

```

但是有一点需要注意，finally 中的 逻辑会覆盖 catch 内的逻辑，同时 finally 中的 return 会最终被执行

```php

var_dump(test_a());

function test_a() {
    try {
        return 1;
    } catch(Exception $e) {
        return 2;
    } finally {
        return 3;
    }
}


```

> int(3)


# foreach 现在支持 list()

```php

$array = [
    [1, 2],
    [3, 4],
];

foreach ($array as list($a, $b)) {
        echo "A: $a; B: $b\n";
}

```

>A: 1; B: 2
>A: 3; B: 4

# password_*

* password_hash(string $password , integer $algo [, array $options ]) 对密码加密
* password_verify(string $password , string $hash) 验证已加密的密码，与保存的 hash 是否一致
* password_needs_rehash(string $hash , integer $algo [, array $options ]) 给密码重新加密
* password_get_info(string $hash) 返回加密算法的一些信息

```php

/**
 * 如果使用默认算法哈希密码，当前是 BCRYPT，并会产生 60 个字符的结果。
 *
 * 需要注意，随时间推移，默认算法可能会有变化，
 * 所以需要储存的空间能够超过 60 字（255字不错）
 */

$pwd = 'HiWiFi';

$options = [
    'cost' => 10,//理解为一种性能的消耗值，cost越大，加密算法越复杂，在不明显拖慢服务器的情况下可以设置最高的值。8-10 是个不错的底线，在服务器够快的情况下，越高越好。在咱们的环境做了个测试，小于 50ms 的情况下，10 是最大的了
    'salt' => mcrypt_create_iv(22, MCRYPT_DEV_URANDOM),
];

//加密

$hash = password_hash($pwd, PASSWORD_DEFAULT, $options);
// $2y$11$gJEX2bLfbI7pWIBKWkCW3ehep4eh7M7MS6BLCvg6hTZSOUOPblbEa

//验证

if (password_verify($pwd, $hash)) {
    echo "pwd is valid\n";
} else {
    echo "invalid pwd\n";
}

//如果 cost 或者 加密算法有了修改。可以这样检查某个 hash 是否是用新的方案进行加密的

$options = [
    'cost' => 11//比如此时改为 11
];

if(password_needs_rehash($hash, PASSWORD_DEFAULT, $options)) {
    $new_hash = password_hash($pwd, PASSWORD_DEFAULT, $options);
}

//注意 password_hash() 返回的哈希包含了算法、 cost 和盐值。

var_export($hash);
/*
*array (
*  'algo' => 1,
*  'algoName' => 'bcrypt',
*  'options' => 
*  array (
*    'cost' => 11,
*  ),
*)
*/

var_export($new_hash);

/*
*array (
*  'algo' => 1,
*  'algoName' => 'bcrypt',
*  'options' => 
*  array (
*    'cost' => 10,
*  ),
*)
*/

```

# array_column


# PHP 7.0.x 新特性

