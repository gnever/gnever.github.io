---
layout: post
title: php内缓存的使用
categories: php
description: php内缓存的使用及缓存的更新策略
keywords: php php缓存 缓存更新
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

## Redis 缓存
优点：高并发集群读写
常用数据结构： key/string/hash/set/list/sortedset
功能及应用场景
* key
* string
* hash
* set
* list
* sortedset

# 缓存常见问题
## 穿透
    * 现象
    * 如何避免
## 雪崩
## Redis 缓存过期 
    ### 如何删除过期建？
    * 当一个键被访问时，程序会对这个键进行检查，如果键已经过期，那么该键将被删除
    * 底层系统会在后台渐进地查找并删除那些过期的键，从而处理那些已经过期、但是不会被访问到的键

    ### Redis 内存回收机制
    * volatile-lru -> remove the key with an expire set using an LRU algorithm
    * allkeys-lru -> remove any key accordingly to the LRU algorithm
    * volatile-random -> remove a random key with an expire set
    * allkeys-random -> remove a random key, any key
    * volatile-ttl -> remove the key with the nearest expire time (minor TTL)
    * noeviction -> don’t expire at all, just return an error on write operations

    ### 什么时候设置过期时间
    * 正常 key-value 存储，使用 expire expireat 等，在更新缓存之后更新
    * 对于hash/set/list/sortedset 数据结构，要先更新过期时间，再添加/修改数据
    * * 为什么？ 
    以hash结构举例，如果 key_test 中现在已经有
    -------------
    | a | b | c |
    |------------
    | 1 | 2 | 3 |
    -------------
    那么此时如果要新增一个域为 d 值为 4 的数据

    ```php

    $key = 'key_test';
    $redis = new Redis();
    $redis->hset($key, 'd', 4);
    $redis->expire($key, 60);
    
    ```
    一般情况下都会这么写，但是如果在执行 hset 之前，key_test 恰好到期了，那么缓存中就只剩下
    -----
    | d |
    |----
    | 4 |
    -----
    这个时候缓存就成脏数据了，所以代码需要这么写

    ```php

    $key = 'key_test';
    $redis = new Redis();
    $rs = $redis->expire($key, 60);
    if(!$rs) {
        //回源更新
        ......
    }
    $redis->hset('key_test', 'd', 4);
    
    ```

    ### 注意,每次设置 setex 和 expire ，key的TTL也会重新开始计算，要避免 setex 和 expire 
    ```php
    //限制用户第一次访问站点之后，60s内不能访问

    //加锁等
    $is_lock = isLock($uid);
    lock($uid, $expire);

    if($is_lock) {
        return false;
    }

    function isLock($uid) {
        $key = 'pre-' . $uid;
        $redis = new Redis();
        return $redis->exists($key);
    }

    function lock($uid, $expire) {
        $key = 'pre-' . $uid;
        $redis = new Redis();
        $redis->setex($key, $expire, 1);
    }
    ```
    这个逻辑会造成用户第一次可以访问，如果再 60 s 内一直在刷页面的话，会导致 key 的 TTL 一直在更新，将永远不能再次访问成功

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
