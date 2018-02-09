---
layout: post
title: PHP Redis Lock 
categories: PHP Redis
description: PHP 使用 Redis 构建简单业务锁
keywords: Redislock PHP锁 lock 分布式锁
---

在高并发业务下会出现多进程操作同一任务单元的情况，此时需要保障能够互斥的访问共享资源或数据。比如秒杀、抽奖、积分兑换等。此类业务如果出现并发请求，就会出现超发问题，而分布式锁就是快速解决这个问题的简单方案。

Redis 为单进程单线程模式，可以将高并发变成串行访问，非常适合实现分布式锁。

# 应用场景

## 场景

```php

$score = getNumFromDb();
if($score > 10) {
    //执行核心业务，减小 num 值并存库
}

```

这个代码可以理解为当积分 score 大于 10 时，进行礼品兑换业务并减少相应的积分。

## 问题

加入一个用户并发请求该业务逻辑，可能从 getNumFromDb 获取到的值相同且都大于 10，此时这些请求就会都进入到核心兑换业务中，然后只减少一次兑换所对应的积分。
既用户使用一次兑换机会，就兑换了多个礼品

RedisLock 即可解决此类问题

# Redis 分布式锁基础知识

* Redis 单进程单线程运行模式
* Redis SET key value \[EX seconds\] \[PX milliseconds\] \[NX\|EX\] : 将字符串值 value 关联到 key
    * EX second ：设置键的过期时间为 second 秒。 SET key value EX second 效果等同于 SETEX key second value 
    * NX ：只在键不存在时，才对键进行设置操作。 SET key value NX 效果等同于 SETNX key value
* Redis setnx  key value : 将 key 的值设为 value ，当且仅当 key 不存在
* Redis setex key seconds value : 并将 key 的生存时间设为 seconds (以秒为单位)
* Redis getset  key value : 将给定 key 的值设为 value ，并返回 key 的旧值(old value)

> 有个不错的中文文档，可以参考 http://redisdoc.com

# 实现

[PHP 代码](https://github.com/gnever/redis-lock)

# 基本需求

* 安全： 保证一个锁只有一个进程能拿到
* 可靠： 锁可以超时自动释放，避免持有锁的进程挂掉或者服务异常导致的解锁失败而引起的死锁问题

> 以上是建立在redis服务正常的情况下。不考虑 Redis 服务或节点宕机问题

# 使用举例

```php

$key = 'lock_demo_' . $uid;//锁的名字，比如同一个用户 uid 下同一时间只能获得一个进程持有锁
$timeout = 10;//强制过期时间，默认 30s
$retry_delay = 100;//单位毫秒，如果发现被锁则等待该时间
$retry_num = 3;//在没有抢到锁时尝试的次数

$redis_lock = new RedisLock($key, $timeout, $retry_delay, $retry_num);
$rs = $redis_lock->lock();
if(!$rs) {
    //没有抢到锁
    return false;
}

//核心业务代码
......
//

//解锁
$redis_lock->unlock();

```

# 实现基本需求

## 安全

保证一个锁只有一个进程能拿到

```php

$rs = $redis->set($this->key, time(), array('nx', 'ex'=>$this->timeout));

```

将当前时间戳作为 value 保存，使用 nx 保障只有在 key 不存在时才能创建成功，并且将超时时间设为传入的参数

## 可靠

### 避免死锁

```php

private function doLock() {
    $obj = CommonRedis::init();
    $rs = $obj->set($this->key, time(), array('nx', 'ex'=>$this->timeout));
    if($rs) {
        return true;
    }

    $lock_time = $obj->get($this->key);
    if(time() - $lock_time < $this->timeout) {
        return false;
    }

    $lock_time = $obj->getSet($this->key, time());
    if(time() - $lock_time < $this->timeout) {
        return false;
    }
    $obj->expire($this->key, $this->timeout);
    return true;
}

```

当加锁失败时，获取当前 key 所关联的 value（最近一次 lock 的时间戳）。判断如果当前时间距离上次 lock 时间没有没有超时的话，则认为一切正常，返回 false。如果已超时，则使用 getSet 获取 oldvalue (最近一次 lock 时间) 并把当前时间戳进行关联。此时判断如果 oldvalue 仍然超时，则认为现在的 lock 超时，可以认为 lock 失效，设置过期时间并返回 true。

之所以使用 getSet 而不是直接使用 set 是为了避免同时有进程执行到此逻辑，出现抢锁导致锁失效的情况。

比如 A 请求抢到锁，由于某些原因导致锁没有释放。此时 B 请求和 C 请求同时执行到了 $lock_time = $obj->get($this->key); 为了避免 B 已经执行 set 之后，C 又重新 set 造成锁失效的情况。使用 getSet 之后如果 B 设置了当前时间，那么 C 会重新判断 lock 是否有效。

### 分散 Redis 操作

```php

if($retry_num > 0) {
    $delay = mt_rand(floor($this->retry_delay / 2), $this->retry_delay);
    usleep($delay * 1000);
}

```

如果设置了重试次数，需要将重试延迟时间打散，避免在高并发下使用相同延迟时间造成 Redis 请求出现波峰波谷现象。利用随机值将请求打散


### 避免错误解锁

```php

public function unLock() {
    $obj = CommonRedis::init();
    $lock_time = $obj->get($this->key);
    //如果锁已经超时，则不能再删除key了
    if(time() - $lock_time < $this->timeout) {
        return $obj->del($this->key);
    }
    return false;
}

```

解锁的逻辑其实就是删除 lock 对应的 key，但是需要避免获得锁的进程执行时间超过设置的 timeout，以至于被其他进程又重新抢到锁，此时删除新锁而造成新锁失效的情况。
