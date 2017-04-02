---
title: Node.js - http模块客户端篇
categories: Node.js学习笔记
date: 2017-03-22 00:00:00
tags:
    - Node.js
    - http
    - request
    - client
    - Agent
---

过年休息了一阵子，似乎吃胖了一圈，想想该继续研究Node.js的东西了。
http模块涉及到的知识特别丰富，Node.js提供的API主要分成两大块，一部份是客户端发请求，另一部份是服务器端的http服务。涉及到的模块也较多，比如net模块、dns模块等等。

<!-- more -->

## 发送请求

作为客户端，发起一个请求是最基本的操作。http模块提供了两个函数用于发起请求，包括了get函数以及request函数，get函数是对request函数的一个封装，它所有的功能都可以使用request函数实现，get函数存在的意义就是让使用者更方便地发起一个get请求，毕竟get请求是最常见的。

```javascript
var http = require("http");

http.get("http://test.dev.com/index.php",function(res) {
    var str = "";

    res.on("data", function(chunk) {
        str += chunk.toString("utf-8");
    }).on("end", function() {
        console.log(str);
    });
});
```

request函数是完整形态的请求方式，提供了大量的参数设置定制请求，它的函数说明是**http.request(options[, callback])**。与get不同，第一个参数是一个对象，并且需要自己解析
url填充参数，以下是上一个例子的request写法：

```javascript
var http = require("http");
var url = require("url");

var uri = url.parse("http://test.dev.com/index.php");

var request = http.request({
    method: "get",
    host: uri.host,
    path: uri.path
}, function(res) {
    var str = "";

    res.on("data", function(chunk) {
        str += chunk.toString("utf-8");
    }).on("end", function() {
        console.log(str);
    });
});

request.end();
```

要注意的是，get和request都会返回一个ClientRequest对象，get会自动调用该对象的end方法，而request则需要使用者自己调用。

## 请求报文头操作

当发起一个请求，在调用ClientRequest的write方法或者end方法之前都可以操作请求的报文头。get函数会自动调用end方法，但并不是调用get函数之后马上就调用end方法，按手册描述就是报文头已经进入发送队列，但是它依然处于可修改的状态。

```javascript
// ...

var request = http.request({
    method: "post",
    host: uri.host,
    path: uri.path
}, function(res) {
   // ... 
});

request.setHeader("Content-Type", "application/x-www-form-urlencoded");
request.write("name=benny");
request.end("name=benny");
```

request函数除了可以在option设置headers属性之外，它返回的HttpClient也提供了几个API供使用者操作header，其中包括：

* headersSent - 检测当前是否已经发送报文头，如果为true则无法写报文头
* getHeader - 获取某个报文头的值
* setHeader - 设置某个报文头的值
* removeHeader - 删除某个已经添加的报文头
* flushHeaders - 将缓冲区的报文头发送，当它被调用后就无法写报文头
* addTrailers - 在报文主体后添加报文头

由于http报文头发送完就开始发送报文主体，当开始发送报文主体之后就不能再发送报文头，但有些情况下报文头的值与报文主体相关，比如某报文头的值是报文主体的内容验证hash值，而报文主体的内容又是动态生成，如果还严格按照先发送报文头再发送报文主体的规定，那么性能、内存消耗可能会很高，因此http1.1允许追加报文头，这种报文头就是trailer。

添加trailer的方法是先设置Transfer-Encoding为chunked以及指定追加报文头的名字Trailer，最后再加追报文头。

```javascript
// ...

var request = http.request({
    method: "post",
    host: uri.host,
    path: uri.path,
    headers: {
        "Content-Type": "application/x-www-form-urlencoded",
        "Transfer-Encoding": "chunked",
        "Trailer": "Content-Hash"
    }
}, function(res) {
    // ... 
});

request.write("name=benny");
request.addTrailers({ "Content-Hash": encodeURIComponent("这是主体内容的hash值")});
request.end();
```

## 请求报文主体操作

get方式的请求不需要报文主体，而post之类的请求则需要报文主体，最常见的报文主体是application/x-www-form-urlencoded类型，内容格式就相当于URL上的查询字符串（比如：a=1&b=2）。请求发送报文主体需要同时设置报文主体的格式，比如上一个例子需要将请求报文头Content-Type的值设置为application/x-www-form-urlencoded。

一般来说常用的格式除了application/x-www-form-urlencoded之外还有multipart/form-data，这是用于上传文件之用的格式。

ClientRequest是一个stream，因此可以使用write方法写入报文主体，或者在调用end方法时也可以最后一次写入报文主体。

ClientRequest涉及到请求报文主体的API有以下几个：

* write - 追加报文主体内容
* end - 最后一次追加报文主体（可选）并关闭写入流
* flush - 将缓冲区的报文主体内容清空并发送

## 请求超时与中断处理

网上许多文章使用了setTimeout来做超时，Node.js在v0.5.9时为ClientRequest新增加了一个setTimeout方法，它能够指定请求发起后多久如果响应数据还没接收完成则调用回调函数告诉使用者时间已经到了，使用者可以在setTimeout方法中做超时处理，比如调用ClientRequest的abort方法中断请求。另外，ClientRequest提供了clearTimeout方法用于取消计时器。

以下是php代码，它将在接收到请求后先睡眠3秒再发送数据：

```php
<?php
sleep(5);
header("Content-Type: text/html; charset=UTF-8");
echo 'done';
?>
```

以下代码定义了1秒超时：

```javascript
var http = require("http");

var request = http.get("http://test.dev.com/index.php",function(res) {
    var str = "";

    res.on("data", function(chunk) {
        str += chunk.toString("utf-8");
    }).on("end", function() {
        console.log(str);
    });
}).setTimeout(1000, function() {
    console.log("已经超时");
});
```

运行结果是请求发起后1秒控制台输出了"已经超时"，到了第3秒再输出"done"。也就是说这里的setTimeout仅仅是告诉使用者时间到了，除此之外并不做任何事情，是否中断请求或者做其它事由使用者自己决定。

把代码简单改一下：

```javascript
var http = require("http");

var request = http.get("http://test.dev.com/index.php",function(res) {
    // ... 
}).setTimeout(1000, function() {
    request.abort();
    console.log("已经超时");
});
```

运行结果将会是：

```text
已经超时
events.js:160
      throw er; // Unhandled 'error' event
      ^

Error: socket hang up
    at createHangUpError (_http_client.js:253:15)
    at Socket.socketCloseListener (_http_client.js:285:23)
    at emitOne (events.js:101:20)
    at Socket.emit (events.js:188:7)
    at TCP._handle.close [as _onclose] (net.js:501:12)
```

非常奇怪，报错了！这里意思是说socket被中断了，可以用error事件接住：

```javascript
var http = require("http");

var request = http.get("http://test.dev.com/index.php",function(res) {
    // ... 
}).setTimeout(1000, function() {
    request.abort();
    console.log("已经超时");
}).on("error", function(err) {
    console.log("报错了");
});
```

运行结果会在一秒后输出：

```text
已经超时
报错了
```

这种现象比较奇怪，中断请求是使用者自己中断的，怎么会抛出异常？个人表示不解，但事实就是如此。ClientRequest有个事件叫abort，它会在客户端中断请求时触发，把代码改一下看看如果监听了该事件是否就不报异常：


```javascript
var http = require("http");

var request = http.get("http://test.dev.com/index.php",function(res) {
    // ... 
}).setTimeout(1000, function() {
    request.abort();
    console.log("已经超时");
}).on("abort", function() {
    console.log("中断请求了");
}).on("error", function(err) {
    console.log("报错了");
})
```

结果是： 

```text
已经超时
中断请求了
报错了
```

经过多次试验，我还发现并不是每次都会抛出异常，现在把服务器端的代码改一下：

```php
<?php
ob_end_clean();
header("Content-Type: text/html; charset=UTF-8");
echo 'hello';
flush();
sleep(3);
echo 'done';
?>
```

这段代码会先输出'hello'，然后马上清空缓冲区将报文头和'hello'发送到客户端，接着进入睡眠，3秒后再输出'done'，运行结果将是：

```text
已经超时
中断请求了
```

也就是说如果已经开始接收到服务器端的响应数据再调用abort方法中断请求将不会触发"socket hang up"，但不管如何这个异常都必须被处理。
经过我的试验，基本上可以总结为以下两条规则：

* 如果服务器尚未输出数据到客户端时中断请求，ClientRequest将触发abort事件并抛出"socket hang up"异常，该异常可以使用error事件接住，这个过程不会调用请求发起时传入的响应回调函数。
* 如果服务器已经开始输出数据到客户端时中断请求，ClientRequest将触发abort事件但不会抛出异常，这时候响应回调函数已经被调用，IncomingMessage对象将触发aborted事件，然后再触发end事件，最后触发close事件。

第二条规则试验代码如下：

```php
<?php
ob_end_clean();
header("Content-Type: text/html; charset=UTF-8");
echo 'hello';
flush();
sleep(3);
echo 'done';
?>
```

```javascript
var http = require("http");

var request = http.get("http://test.dev.com/index.php",function(res) {
    var str = "";

    res.on("data", function(chunk) {
        str += chunk.toString("utf-8");
    }).on("aborted", function() {
        console.log("响应中断");
        str = ""; // 抛弃结果
    }).on("close", function() {
        console.log("连接关闭")
    }).on("end", function() {
        console.log("响应结束");
    });
}).setTimeout(1000, function() {
    request.abort();
    console.log("已经超时");
}).on("abort", function() {
    console.log("中断请求了");
}).on("error", function(err) {
    console.log("报错了");
});
```

输出结果是：

```text
已经超时
中断请求了
响应中断
响应结束
连接关闭
```

## 禁用请求Nagle算法

Nagle算法是用于减少网络负符的数据发送规则算法，由于每次发送数据都会带上20字节的TCP头以及20字节的IP头，因此哪怕数据只有一个字节最终也需要发送41个字节的数据。如果通讯发送的数据都是小数据为主，那么将会有很多冗余消耗，Nagle算法就是为了解决这种问题，它会将小包数据暂缓存起来，等数据量达到某个值时再一次性发送。

ClientRequest提供了一个方法用于打开或关闭Nagle算法: setNoDelay([noDelay])，该值默认是真，表示关闭。

## 100-continue协议处理

当客户端POST数据给服务器端时一般会带上一个值为100-continue的报文头Expect用于询问服务器是否处理POST数据（这年头有不处理POST数据的WebServer吗？），如果服务器返回状态码100则表示它会处理POST数据，客户端这时候就可以发送报文主体内容了。一般来说只有需要POST大量数据给服务器端才会使用100-continue协议。

ClientRequest提供了continue事件用于处理100-continue，当服务器返回状态码100时触发，这时候就可以调用write方法写入大量数据了，要特别注意的是写完才能够调用end方法。

```php
<?php
header("Content-Type: text/html; charset=UTF-8");
echo $_POST['name'];
?>
```

```javascript
var http = require("http");
var url = require("url");

var uri = url.parse("http://test.dev.com/index.php");

var request = http.request({
    method: "post",
    host: uri.host,
    path: uri.path,
    headers: {
        "Content-Type": "application/x-www-form-urlencoded",
        "Expect": "100-continue"
    }
}, function(res) {
    var str = "";

    res.on("data", function(chunk) {
        str += chunk.toString("utf-8");
    }).on("end", function() {
        console.log(str);
    });
}).on("continue", function() {
    console.log("收到了100 continue");
    this.write("name=benny");
    request.end();
});
```

运行完输出结果为：

```text
收到了100 continue
benny
```

## 响应信息操作

在调用get或者request函数发起请求时可以传一个回调函数callback作为参数，相当于监听了ClientRequest对象的response事件。当请求开始接收响应主体内容的时候将会调用该回调函数，这时候响应报文头已经解析完毕。

响应回调函数会传入一个IncomingMessage对象，它提供了多个属性或方法用于获取响应信息：

* httpVersion - http协议版本，比如1.1
* headers - 一个JSON对象，以{ key1: value1, key2: value2, ... }形式存储了报文头，需要注意的是这里的key全部是小写
* rawHeaders - 一个数组，以[key, value, key, value, ...]形式存储了报文头，需要注意的是这里的key全部是原始写法，客户端传什么它就是什么。
* trailers - 一个JSON对象，以{ key1: value1, key2: value2, ... }形式存储了追加的报文头，需要注意的是这里的key全部是小写。由于trailers是追加在某些主体内容后的报文头，因此当响应回调函数被调用时它未必会有值。
* rawTrailers - 一个数组，以[key, value, key, value, ...]形式存储了追加的报文头，需要注意的是这里的key全部是原始写法，客户端传什么它就是什么。
* upgrade - 协议是否升级（比如http升级为websocket）
* url - 请求的url
* method - 请求的方式，比如GET
* statusCode - 响应状态码，比如200
* statusMessage - 响应描述，比如OK

除此之外，IncomingMessage对象还提供了两种事件：

* aborted - 当request已经中断或者网络中断，则触发此事件
* close - 当它依赖的socket连接已经关闭时触发

由于IncomingMessage本身就是一个stream，因此在使用时可以使用data、end等事件，具体示例见上方各种示例。

