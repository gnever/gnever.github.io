---
layout: post
title: php内缓存的使用
categories: php
description: php内缓存的使用及缓存的更新策略
keywords: Java
---

为了使程序执行速度更快，所以现有业务中大量使用了缓存机制，这里主要介绍下平时业务中缓存的应用及缓存更新的策略

# 缓存的应用 

## 变量级别的缓存

适用场景：对于少了数据，读多写少
缺点：仅在生命周期内生效，进程间无法共享（PHP7+对于）
如： 

```php
class A {
    public static $x = null;
    public static $xx = null;

    public function getX() {
        if(!is_null(self::$x)) {
            return self::$x;
        }
        //todo 数据处理
        return self::$x = 'xxxx';
    }

    public function getXx($key) {
        if(isset(self::$xx[$key])) {
            return self::$xx[$key];
        }

        self::$xx = array();
        //todo 数据处理
        return self::$xx = 'xxxx';
    }
}
```

这里可以思考下 geX 和 getXx 的应用场景

## redis缓存
高并发集群读写
常用数据结构 ： key/string/hash/set/list/sortedset
功能及应用场景
* key
* string
* hash
* set
* list
* sortedset

# 缓存常见问题
## 穿透
## 雪崩
## 过期

# 更新缓存的策略

先看集中不同的更新策略

* 新更新缓存再更新数据库
* 先更新数据库再更新缓存
* 先淘汰缓存再更新数据库
* 先更新数据库再淘汰缓存
 
分析

* 新更新缓存再更新数据库
  会有脏数据。比如并发两个请求，A请求要将数据修改为aa,B请求要将数据修改成bb。
  当A请求前修改了cache为aa,等待数据库连接并且发送修改命令。此时B请求也来了将cache修改为bb,然后也执行了数据库操作，由于最先抢到修改数据库数据为bb，然后才轮到A请求修改数据库数据为aa。那么此时，cache内数据为bb，但是数据库内数据为aa。*异常*
* 先更新数据库再更新缓存
* 先淘汰缓存再更新数据库
* 先更新数据库再淘汰缓存
