---
layout: post
title: 记一次前端性能优化
categories: 前端性能
description: 前端性能优化
keywords: 前端优化 js加载顺序
---

*以下内容复制自2014年时的笔记，只做了排版的整理，些许内容现在看有些稚嫩，但是作为成长的足迹也一并保留下来，大家仅供参考*

# 背景

app.hiwifi.com 站点登陆成功之后到加载完成耗时平均在 8s 左右，非常之慢，所以要对其进行优化

## 页面渲染耗时

![从点击登陆到首屏加载完毕耗时](/images/posts/front_end/time-statistics.png)

模拟了从登陆到进入已安装列表，其中有两次302，而且明显的静态文件加载缓慢

## 静态文件耗时

静态文件类型加载时间分布

![静态文件类型加载时间分布](/images/posts/front_end/static-file-time-statistics.png)

非常明显，静态文件的耗时占比达到90%以上，那么此次的重点优化就在这里了

## 域名耗时

![所在域名耗时统计](/images/posts/front_end/dns-time-statistics.png)

*备注：ss.hiwifi.com 以前是作为静态资源加载的域名存在的*

## DNS耗时

![DNS解析耗时](/images/posts/front_end/dns-reslove-time-statistics.png)

出现过这么一次情况，DNS 解析时间达到 1s ，时间很长，复现几率较小，怀疑跟 DNS TTL 有关系，问了下建华设置的是 10 分钟，很有可能碰到了过期，导致又到各级 DNS 代理走了一圈

# 解决方案

* 合并 js\css 以登录页为例，要加载 11 个 js 和 3 个 css ，共 14个 请求，平均每个请求 0.2s，合并后可缩短为 2 个请求，减少 https 请求消耗。
* 所有的静态文静走第三方域名，静态文件要避免使用 cookie  可以使用 s.histatic.com [注1]
* 静态文件配置多域名（ js 和 css 合并后主要是图片使用），需要统一的 header 和 footer 及 js\css 加载方法
* 开启gzip压缩，expire 目前 s.hiwifi.com 、c.hiwifi.com 没有
# 开启keepalive
* 使用cdn  但目前国内貌似没有 https 的CDN
* css 放在 js 前加载，保证 css 的并行下载（其中登录页js阻塞较明显。有js的渲染）[注2]
* 代码规则：
    * css 优先放在 header 中，页面在下载完 css 之后才会再下载其他的。
    * 避免使用内嵌样式或放在 body 中，避免重绘。
    * js 除 jQuery 外全部放到 footer 中。使用 (function($) { } )(jQuery)，方便使用合并 js 功能。
    * 页面头部优先定义编码。浏览器会缓冲一定的字节数据来查找编码信息，尽量减少缓冲数据量（在预设的缓冲量还没有找到编码信息后会使用当前默认的编码，但是在加载后续中如发现编码格式跟默认不一致则又会重新渲染）。
* php 中使用header跳转一定要写准确的url，比如不要写http://app.hiwifi.com 要用https，否则服务端会调转至https。过程就是用户发送请求，收到服务端302跳转请求后，在发送请求。[问1]

## 静态资源合并 

### concat

使用了淘宝的 mod_concat

在测试机搭了个示例，大家可以看下效果：

>合并后：http://192.168.5.113/

>没有合并：http://192.168.5.113/noconcat.html

*当时搭建的测试服务，已经关掉了，主要形式就是 js 或者 css 可以写成*

``` http
http://domain.com/??js1.js,js2.js,js3.js

http://domain.com/??style1.css,style2.css,style3.css
```

主要功能是将静态资源文件的请求合并成一个，然后由 nginx 做分发，拉取不同的文件。通过减少 web 请求来减低服务器压力，同时减少 client 端请求建立连接的耗时 [注5]

### fis

>http://fis.baidu.com/ 百度前端

与 concat 不同的时，该工具在代码发布时，在物理上将多个静态文件合并为一个文件，所以前端只要请求加载这一个文件即可

# 实施
* 增加 s.histatic1.com s.histatic2.com，作为 https 页面下的资源加载域名 [注3]
* 增加 c.histatic1.com c.histatic2.com，作为 http 页面下的资源加载域名，同时增加 CDN [问2]
* nginx 开启 gzip
* nginx 设置 expire [注4]
* nginx 开启 keepalie
* js/css concat
* php 统一 header、footer 及 css/js 加载
* 代码调整（ css/js 顺序、header 跳转 ）

# 补充

当时考虑的并不是很周全，所以有些需要补充的

* 减小 cookie
  * 减小 cookie 数量和长度
  * 不要将所有的 cookie 都设置在根域下，合适的位置才是最好的
* 延迟加载。只加载必须的资源，对于某些数据和用户不可见图片等可以延迟加载
* css 内图片压缩合并
* 设置 DNS 预解析

```html
<meta http-equiv="x-dns-prefetch-control" content="on" />
<link rel="dns-prefetch" href="https://s.histatic.com" />
```

* 按域名划分内容，对于 css 资源需要最快速的加载出来，所以在 css 资源不多的情况下尽量只用同一个独立域名，减少 DNS 的 解析
* 页面内 ajax 尽量使用 GET 方式，因为 POST 请求其实是分两步的，先发送头信息，然后再发送数据.
    >POST is implemented in the browsers as a two-step process: sending the headers first, then sending data.

# 注

[注1]

>一般会把 cookie 设置在顶级域名 hiwifi.com 下，那么它所有的子域名传输时都会这这个 cookie ，比如 s.hiwifi.com/xxx.png 或者  s.hiwifi.com/xx.js 或者 s.hiwifi.com/xx.css 
>但是对于静态文件来说，这些 cookie 是毫无用处的，完全是在浪费带宽。
>所以静态文件需要使用独立的域名

[注2]

> js 的加载会阻塞整个页面的渲染。这么设计也是很有意思的
> 由于静态资源可以并发下载参考 [注释3] ，而 js 脚本可能会改变页面，所以需要先将整个未知部分解决掉。最终要的是 js 脚本是顺序加载的，比如某个脚本需要调用 jQuery，那么就必须等待 jQuery 加载完毕之后才能再加载自己。

[注3]

>浏览器对每个域名的并发链接是有限制的，使用多个独立域名，可以大大增加并发连接数，可以使流浪器并行下载更多资源，加速展示
>但是从 DNS 的角度上讲，每增加一个域名就代表着要增加一个 DNS 的解析（一般 DNS 解析的耗时在 20ms 以上），并且 keepalive 也不可能用在不同的域名上使用，所以还需要根据页面中加载的静态文件资源的数量和 DNS 服务商的性能权衡如何使用。

[注4]

>大家应该注意到加载静态资源会有 http 304 code 是因为浏览器缓存了目标资源但不确定该缓存资源是否是最新版本的时候,就会发送一个 If-Modified 的请求头，头信息中有文件上次修改时间和一个标识，nginx 根据这两个头信息判断是否需要更新文件。如果不需要则返回 304 ，浏览器还是使用之前下载的资源。否则返回 200 ，浏览器重新下载资源。
>但是如果 nginx 设置了 Cache-Control 或者 expire ，则在有效期内，浏览器打开页面之后直接使用缓存的文件，连 If-Modify 的请求都不用发。
>但是由于 CDN 和浏览器自身的一些缓存机制，上述的两个方案不一定好用，或者要缓存的更彻底一些。

[注5]

>一直在强调减少 http 的请求，因为每个请求都有成本的，可以分为时间成本和资源成本。
>一个完整的请求都需要经过 DNS 解析、与服务器建立连接、发送数据、等待服务器响应、接收数据、关闭连接。对于 https 的请求还有证书下载、数据加密、数据解密的步骤

# 问题

[问1]

>1 服务端发送跳转后整个逻辑是怎么执行的？
>2 为什么跳转了还要加 exit
```php 
header("Location: http://www.hiwifi.com/"); 
exit;
```
>3 既然说到了 header 跳转，还有一个相似的 case
    >nginx 499 的 code
    >这个 code 是 nginx 自定义的，产生原因是 client 主动断开了与 nginx 的连接
    >那么问题来了，按照咱们后端的架构 client -> nginx -> phpfpm 。如果一个 client 请求 php 执行很耗时，用户等烦了直接刷新网页或者关闭了网页，那么此时 nginx 会断掉，记录为 499，但是 php-fpm 怎么处理这个请求？
    >答案：php-fpm 不受 client 关闭影响，继续执行。
    >那么这就产生两个问题
        >(1) 耗时请求过多时，就算 client 端关闭了请求，但是 php-fpm 的资源也不会自动释放，直到执行结束或超时。此时 php-fpm 就很容易被打满，没有资源处理新进入的请求。
        >(2) 如果 client 发起的是数据修改的请求，即使 client 关闭的请求，也有可能会修改成功的
    >对于平时经常用到的业务影响就是
        >(1) 请求修改用户信息接口，即使 curl 返回的错误 code 是 28，但是也有可能成功。计数、扣款、增加付费时长等数据敏感的接口，即使返回 28，也不一定修改失败。注意不要因为调用超时就循环请求接口，否则可能就会造成 重复扣款或重复添加付费时长。
        >(2) php 内调用 openapi 同理

[问2]

>为什么要分 s.histatic.com 和 c.histatic.com ，即 为什么加载 https 和 http 的需要区分开来？
