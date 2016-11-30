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
缺点：仅在生命周期内生效，进程间无法共享
如： 

```php

class A {
    public static $x = null;
    public static $xx = null;

    public function getX() {
        if(!is_null(self::$x)) {
            return self::$x;
        }
        //todo 耗时处理
        return self::$x = 'xxxx';
    }

    public function getXx($key) {
        if(isset(self::$xx[$key])) {
            return self::$xx[$key];
        }

        self::$xx = array();
        //todo 耗时处理
        return self::$xx[$key] = 'xxxx';
    }
}

```

这里可以思考下 geX 和 getXx 的应用场景

## Redis 缓存

    * 优点：高并发集群读写
    * 常用数据结构： key/string/hash/set/list/sortedset
    
    ###功能及应用场景

    * key 键
        * * keys 线上服务杜绝使用
    
    * string 字符串
        * * incr 原子性的计数器。对于并发的计数如PV/点击等可以使用 incr 存储，然后再入库
        * * setex set 某个值同时设置过期时间

    * hash   哈希表
        * * 存储结构化的数据
        * * 避免将数据 json_encode 之后按照 key - value 方式存储，更加节省空间 
        
    * set    集合
        * * set元素最大可以包含(2的32次方-1)个元素，可以做 并集、交集、差集 的处理，比如 共同好友 

    * list   列表
        * * 一般作为队列使用。注意队列消费者使用阻塞模式的blpop
    
    * sortedset  有序集合
        * * 一般存储有权重的数据，比如 排行榜

# 缓存常见问题
## 穿透

    * 现象
        
        其实就是查询一个不存在的数据，缓存中没有，然后又去数据库读，数据库中获取不到则不会更新缓存。这就导致每次的请求都会去存储层查询，失去了缓存的意义。

    * 如何避免

        如果有大量请求都穿透的话，数据库压力会非常大。所以在特定情境中需要对穿透的数据设置缓存，比如缓存一个标识，告诉请求没有数据，避免穿透。

## 雪崩
    
    * 现象
        
        缓存服务器重启或缓存集中在同一段短时间内失效，导致大量的强求都打到数据库或后端服务，造成服务压力过大。

    * 如何解决
    
        * * 缓存服务器 高可用
        * * key的过期时间尽量分散

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
        //当TTL更新失败时表明缓存已经丢失了，需要做回源更新
        ......
    }
    $redis->hset('key_test', 'd', 4);
    
    ```

    ### TTL 
    * 注意,每次设置 setex 和 expire ，key的TTL也会重新开始计算*

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