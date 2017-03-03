---
layout: post
title: php-process
categories: PHP pcntl
description: PHP多进程
keywords: PHP 多进程 pcntl
---


# php-process

## 使用场景

主要用在处理大数据的情况下，可以自动拆分多个进程同时处理数据，加快处理速度 

### 基础调用

```php

use lib\pcntl\Process;
use lib\pcntl\ProcessPool;

+--  9 lines: function test($a, $b) {----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

+--  9 lines: function testPool($a, $b) {------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

+--  3 lines: function before($a, $b) {--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

+--  3 lines: function after($a, $b) {---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

$p = new Process('test', array('test1', 'test2'));
$p->setBeforeCallback('before', array(1, 2));
$p->setAfterCallback('after', array(3, 4));

$pool = new ProcessPool();
$pool->run($p)
     ->run(new Process('testPool', array('test_pool1', 'test_pool2')))
     ->run(new Process('testPool', array('test_pool3333', 'test_pool44444')));
$pool->wait();

```

### 以文件为基准拆分

#### 简单调用

* 默认 fork 10个消费进程
* 显示 notice 级别输出
* 默认加执行锁，同一个文件同时只能执行一次


```php

use lib\pcntl\AbstractProcessByFileHandler;
use lib\pcntl\ProcessLogger;

class TestHandler extends AbstractProcessByFileHandler {

    protected function lineHandler($line) {
       echo "line " . $line . "\n";
    }
}

$test = new TestHandler();
$test->setFile($file)->run();

```

#### 自定义调用

```php

use lib\pcntl\AbstractProcessByFileHandler;
use lib\pcntl\ProcessLogger;

class TestHandler extends AbstractProcessByFileHandler {

    protected function lineHandler($line) {
       echo "line " . $line . "\n";
    }
}

$test = new TestHandler();
$test->setFile($file)
     ->setConsumerNum(20)   //定义启动的消费进程数
     ->setLogLevel(ProcessLogger::ERROR)  //定义显示的日志级别
     ->setIsLock(0) //不加执行锁
     ->run();

``

### 非文件为基准的拆分（每个进程数据自行获取）

```php

use lib\pcntl\AbstractProcessBySelfHandler;
use lib\pcntl\ProcessLogger;


class TestHandler extends AbstractProcessBySelfHandler {

    //这个方法接收当前要处理的数据段，需要开发者自行处理数据
    protected function getDataList($start, $offset) {
        //$sql = 'select * from router_bind limit ' . $start . ',' . $offset;
        return array(1, 2, 3, 4, 5);
    }

    //将 getDataList 返回的数据集拆分成一个个的数据元素，单独处理
    protected function dataHandler($data) {
        //sleep(100);
        var_dump($data);
    }
}

$test = new TestHandler();

//只显示error及以上输出
//$test->setFile('/tmp/macs.log')->setConsumerNum(20)->setLogLevel(ProcessLogger::ERROR)->run();

//默认显示所有输出, fork 10个消费进程
//$test->setTotal(100)->run();

$test->setTotal(100)   //设置总共需要处理的数据量
     ->setConsumerNum(20) //设置需要启动的消费进程数
     ->setLogLevel(ProcessLogger::ERROR)  //设置需要显示的输出信息
     ->run();

```
