---
layout: post
title: php-multi-process 
categories: PHP pcntl
description: PHP多进程
keywords: PHP 多进程 pcntl
---

PHP 在处理大数据的业务场景中并不占优，一般的做法就是 foreach 循环要处理的数据，然后逐一处理。
但是在数据量较大或者运算复杂的情况下耗时会比较长，此时一般有两种做法，要么优化处理脚本，要么就可以考虑多个进程同时进行处理了。
本篇就是记录使用 PHP pcntl 扩展完成多进程批量处理数据的业务 

# php-multi-process 

## 使用场景

主要用在处理大数据的情况下，可以自动拆分多个进程同时处理数据，加快处理速度 

## 基础调用

```php

use pcntl\Process;
use pcntl\ProcessPool;


function test($a, $b) {

    echo "this is function execute start , argumet 1: {$a} argument 2: {$b}\n";
    for($i = 1; $i <= 5; $i++) {
        //echo $i . "\n";
        sleep(1);
    }
    echo "this is function execute end , argumet 1: {$a} argument 2: {$b}\n";
}

function testPool($a, $b) {

    echo "test pool this is function execute start , argumet 1: {$a} argument 2: {$b}\n";
    for($i = 1; $i <= 10; $i++) {
        //echo "pool " . $i . "\n";
        sleep(1);
    }
    echo "test pool this is function execute end , argumet 1: {$a} argument 2: {$b}\n";
}

function before($a, $b) {
    echo "this is before execute , argumet 1: {$a} argument 2: {$b}\n";
}

function after($a, $b) {
    echo "this is after execute , argumet 1: {$a} argument 2: {$b}\n";
}

$p = new Process('test', array('test1', 'test2'));
$p->setBeforeCallback('before', array(1, 2));
$p->setAfterCallback('after', array(3, 4));

$pool = new ProcessPool();
$pool->run($p)
     ->run(new Process('testPool', array('test_pool1', 'test_pool2')))
     ->run(new Process('testPool', array('test_pool3333', 'test_pool44444')));
$pool->wait();

```

## 以文件为基准拆分

### 简单调用

* 默认 fork 10个消费进程
* 显示 notice 级别输出
* 默认加执行锁，同一个文件同时只能执行一次


```php

use pcntl\AbstractProcessByFileHandler;
use pcntl\ProcessLogger;

class TestHandler extends AbstractProcessByFileHandler {

	//$line 即为处理数据的最小单元，既文件的一行字符串
    protected function lineHandler($line, $num) {
        echo "lineHandler num is " . $num . "; line is " . $line . PHP_EOL;
    }
}

//要处理的文件名，文件内每行为最小处理单元
$file = 'readyfile.log';


$test = new TestHandler();
$test->setFile($file)->run();

```

### 自定义调用

```php

use pcntl\AbstractProcessByFileHandler;
use pcntl\ProcessLogger;

class TestHandler extends AbstractProcessByFileHandler {

    protected function lineHandler($line, $num) {
        echo "lineHandler num is " . $num . "; line is " . $line . PHP_EOL;
    }
}

$file = 'readyfile.log';
$test = new TestHandler();
$test->setFile($file)
     ->setConsumerNum(20)   //定义启动的消费进程数
     ->setLogLevel(ProcessLogger::ERROR)  //定义显示的日志级别
     ->setIsLock(0) //不加执行锁
     ->run();

```

## 非文件为基准的拆分（每个进程数据自行获取）

```php

use pcntl\AbstractProcessBySelfHandler;
use pcntl\ProcessLogger;


class TestHandler extends AbstractProcessBySelfHandler {

    //这个方法接收当前要处理的数据段，需要开发者自行处理数据
    protected function getDataList($start, $offset) {
        //$sql = 'select * from tablename limit ' . $start . ',' . $offset;
        return array(1, 2, 3, 4, 5);
    }

    //将 getDataList 返回的数据集拆分成一个个的数据元素，单独处理
    protected function dataHandler($data) {
        //sleep(100);
        var_dump($data);
    }
}

$test = new TestHandler();

//默认显示所有输出, fork 10个消费进程
//$test->setTotal(100)->run();

$test->setTotal(100)   //设置总共需要处理的数据量
     ->setConsumerNum(20) //设置需要启动的消费进程数
     ->setLogLevel(ProcessLogger::ERROR)  //设置需要显示的输出信息
     ->run();

```
