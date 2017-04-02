---
title: 浏览器事件循环以及多线程
tags:
  - event-loop
  - multi-thread
  - web-worker
categories: javascript
keywords: '多线程, 事件循环, WebWorker, 单线程, 性能'
date: 2016-11-26 02:33:32
---
Javascript在浏览器中一直都是以单线程的方式运行，这意味着它同一时间只能干一件事，前端们不用担心你在读完一个变量的时候该变量马上被另一个线程修改而导致后续计算变得不准确，也不用担心对一个DOM节点做修改操作的时候突然它被删除。支持多线程的语言都有一个东西叫线程锁，但是Javascript作为操作HTML页面的脚本语言它应该足够简单，所以单线程是最合适的，但同样有方法实现多线程以提高系统资源利用率。

<!-- more -->

## 事件循环

首先了解一下什么叫进程。进程就是一个应用程序在内存中的实例，一个CPU核心只能在同一时间处理一个进程的工作，操作系统同时在跑的应用程序当然不可能只有一个，所以操作系统就不停地把CPU资源交给其中一个进程，而其它没排到队的进程则进入沉睡状态。许多时候为了更好的利用起多核CPU的优势，一个进程往往会把自己fork成多个进程，让应用程序的工作能够同时在多个CPU上跑起来以达到利用更多CPU资源提高性能的目的，比如Node.js就是这么干的。

线程是轻量版的进程，一个进程可以拥有多个线程，线程也是CPU资源调度的最小单位，但线程与线程可以共享进程的公用资源。在超线程技术发明之后，多核多线程CPU再配上相应的操作系统、软件可以实现多个线程利用多个CPU核心的资源。

上边仅是简单地讲一下进程和线程，我对这方面并不擅长，或许有什么理解上的错误，但是GUI界面的应用程序一般都是对UI的交互交给一个线程处理，其它线程用于做计算。这样，其它线程在计算的同时不会阻塞了UI线程的运行，界面也就不会卡顿。

可能有人会提起setTimeout，说当我设置两个定时器，都是在1秒后执行，那么它们不就同时执行了吗？看起来像是多线程了，事实上二者不可能同时执行。使用setTimeout注册过的函数并不保证非常准时地执行，可能会延后一点时间执行，两个注册在定时器里的函数也是先后执行的，只有一个执行完了才会执行另一个。

这个先后关系是通过一个叫事件循环的东西来做调度的，页面上所有任务都会被放到一个先进先出的队列中进行排队，事件循环机制会从队列头取出一个任务执行，该任务完成后再取下一个任务执行。如果一个任务有大量的运算或费时的同步操作（比如同步ajax）将会导致下一个任务一直处于等待状态。

我们可以做一个实验：

```html
<!DOCTYPE html >
<html>

<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />
    <meta name="renderer" content="webkit" />
    <title>测试</title>
    <script src="http://lib.sinaapp.com/js/jquery/1.7.2/jquery.min.js" type="text/javascript"></script>
</head>

<body>
    <script type="text/javascript">
        console.time("begin");

        setTimeout(function() {
            console.timeEnd("begin");
        }, 1000);

        for (var i = 0; i < 1000; i++) {
            $.ajax({
                url: "./test.html",
                async: false,
                method: "get"
            });
        }

    </script>
</body>

</html>
```

在这个示例中，一开始就注册了个函数在一秒后输出从开始注册到开始执行的时间差，随后再调用ajax同步请求当前页面（./text.html）一千次，在我的机子上输出了"begin: 1.74e+03ms"，也就是事实上1740ms后才执行了setTimeout的回调函数。这是因为同步操作导致事件循环长时间耗在请求代码的任务上而没有开始下一个注册的任务。

事件循环的任务队列可以理解成是一个数组，每个数组元素都是一个函数（待执行的任务），它是一个同步执行的任务队列，事件循环机制会按顺序将它们一个一个执行，这个就是主线程任务队列。除了主线程任务队列之外还有一个异步的任务队列，这个异步任务队列里的任务并不会被主动执行，它会在它应该被执行的时候往主线程任务队列插入任务，setTimeout就是这种异步任务，网页上的各类事件也是类似的异步任务，它们随时向主线程任务队列插入任务。

## 初识Worker

尽管在浏览器中脚本确实是以单线程的方式运行的，但考虑到可能会有大量数据计算之类的耗时操作，IE10以上版本以及其它标准浏览器都已经支持了Web Worker，它实现了Javascript在浏览器中多线程的需求，在一个Worker中不管做什么同步操作都不会影响到页面的操作，也不会导致页面卡顿（当然，你的硬件很差这个就回天无力了，毕竟系统资源就那么多）。

先来看一个简单的例子：

```javascript
console.time("timeout");

setTimeout(function() {
    console.timeEnd("timeout");
}, 100);

console.time("concat");

var str = "";

for (var i = 0; i < 10000000; i++) {
    str = str + "abc";
}

console.timeEnd("concat");
```

在我机子上执行结果是：

```text
concat: 609ms
timeout: 674ms
```

由于现在我们将字符拼接的代码放入Worker中执行：

```javascript
console.time("worker");

var str = "";

for (var i = 0; i < 10000000; i++) {
    str = str + "abc";
}

console.timeEnd("worker");
```

```javascript
var worker = new Worker("./worker.js");
console.time("timeout");

setTimeout(function() {
    console.timeEnd("timeout");
}, 100);

console.time("worker");
```

很惊人地结果：

```text
timeout: 100ms
worker: 109ms
```

虽然并不是每次都一样，但基本上timeout都是100ms，而worker都是100ms出头，也就是说从性能上看提高幅度非常大。为什么同样的代码在页面上执行需要609ms，而在worker中却只需要109ms？按我的理解是浏览器主页面线程的脚本是跟整个页面的各种任务在抢资源，而worker则是另起一线程独占了一份独立的CPU资源，自然运行速度就快很多了。

在MDN上关于Worker的介绍有一段文字很自信地拍着胸口向使用者们保证Worker的线程安全问题：

{% blockquote MDN https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API/Using_web_workers 使用 Web Workers %}
Worker 接口会生成真正的操作系统级别的线程，如果你不太小心，那么并发(concurrency)会对你的代码产生有趣的影响。然而，对于 web worker 来说，与其他线程的通信点会被很小心的控制，这意味着你很难引起并发问题。你没有办法去访问非线程安全的组件或者是 DOM，此外你还需要通过序列化对象来与线程交互特定的数据。所以你要是不费点劲儿，还真搞不出错误来。
{% endblockquote %}

Worker在运行期间与主线程几乎是完全分开的，二者无法直接共享数据，Worker也不能直接操作页面的DOM节点，简单来说不能把它当面浏览器中的普通脚本，它的环境更加纯净。

Worker可以做以下几件事：

* 与主线程互相通讯
* 使用XMLHttpRequest发起请求
* 使用定时器

除此之外，Worker同时能够使用许多函数以及小量的限制版对象，比如location以及navigator。这方面的信息可以查阅《[Worker支持的函数](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Functions_and_classes_available_to_workers 来自MDN)》。

## 使用Worker

### 初始化

Worker初始化非常简单，直接使用new关键词新建一个实例，传入新线程代码所在的脚本文件路径(注意：**必须遵守同源策略，但Worker内部使用importScripts函数加载的代码则不需要遵守，你可以加载CDN上的jquery库**)，比如：

```javascript
var worker = new Worker("./test.js");
```

事实上为了减少http请求，除了IE之外的浏览器还支持用URL对象当参数：

```html
<script type="text/code" id="worker-code">
    console.time("inline worker");

    var str = "";

    for (var i = 0; i < 1000000; i++) {
        str = str + "abc";
    }

    console.timeEnd("inline worker");
</script>
<script type="text/javascript">
    var code = document.getElementById("worker-code").innerHTML;
    var blob = new Blob(Array.from(code), { type: "text/javascript" });
    var url = URL.createObjectURL(blob);

    var worker = new Worker(url);
    console.time("timeout");

    setTimeout(function() {
        console.timeEnd("timeout");
    }, 100);

    console.time("worker");
</script>
```

在我的机子上，运行结果是：

```text
timeout: 101ms  
inline worker: 108ms
```

其实跟前边的运行结果没什么区别，本来timeout就有可能延时执行。我个人是比较推荐内嵌worker代码在页面或者放入javascript代码内执行（很简单啊，写个函数然后toString一下就得到它的代码了…），这样可以节省http请求，加载Worker的脚本文件也是要花时间的，尽管浏览器很聪明地帮我们隐藏了一切，你可以把它当成瞬间就加载完一般使用，这个后边会讲。

### 线程通讯

Worker线程可以与主线程通讯，这主要是通过postMessage以及message事件来实现，并且可以传递序列化后的数据给对方：

```html
<script type="text/code" id="worker-code">
    addEventListener("message", function(ev) {
        // ev是一个message对象，可以用这种方法看它的结构
        // console.dir(ev);
        console.log("worker accept: %s", ev.data);
        postMessage("hello master");
    });
</script>
<script type="text/javascript">
    var code = document.getElementById("worker-code").innerHTML;
    var blob = new Blob(Array.from(code), { type: "text/javascript" });
    var url = URL.createObjectURL(blob);

    var worker = new Worker(url);

    worker.addEventListener("message", function(ev) {
        console.log("master accept: %s", ev.data);
    });

    worker.postMessage("hello worker");
</script>
```

在主线程中首先创建了一个Worker，并为该Worker绑定了message事件用于响应来自Worker post过来的消息。Worker同样使用绑定了message事件，用于接收来自主线程的消息并随后向主线程发送消息，这就是最简单的通讯方式。这段代码在我的机子上运行结果如下：

```text
worker accept: hello worker
master accept: hello master
```

一般来说线程之间无法互相post对象，只能post可序列化的数据，比如将示例中的代码稍改一下：

```html
<script type="text/code" id="worker-code">
    addEventListener("message", function(ev) {
        // ev是一个message对象，可以用这种方法看它的结构
        // console.dir(ev);
        console.log("worker accept: %s, type: %s");
        postMessage("hello master");
    });
</script>
<script type="text/javascript">
    var code = document.getElementById("worker-code").innerHTML;
    var blob = new Blob(Array.from(code), { type: "text/javascript" });
    var url = URL.createObjectURL(blob);

    var worker = new Worker(url);

    worker.addEventListener("message", function(ev) {
        console.log("master accept: %s", ev.data);
    });

    worker.postMessage(document);
</script>
```

该示例将document对象post过去，然后报了这样的异常：Uncaught DOMException: Failed to execute 'postMessage' on 'Worker': An object could not be cloned.意思是说document无法被复制一个副本用于post。

没错，postMessage方法在传递数据的时候是复制一个副本而不是直接将内容本身送过去，类似于函数传递参数，函数的参数就相当于函数的局部变量，它是外部传递进来的值的一个副本，二者毫无关系互不影响。而document对象却无法复制（只要是DOM对象皆如此），无法传递对象的引用。

或许现在你会马上尝试传递一个普通的Javascript对象过去，可以的，一点问题都没有！因为普通的对象依然是可以通过JSON.stringify(obj)的方式转换成json字符串。我尝试过，确实非常方便，在Worker内部接收到的自动帮我们反序列化成一个跟原来一模一样的对象。要注意的是：**该对象依然是原来对象的一个副本，在Worker中进行修改并不会导致主线程的对象发生改变！**

```html
<script type="text/code" id="worker-code">
    addEventListener("message", function(ev) {
        var obj = ev.data;
        console.log("worker: %s", JSON.stringify(obj));
        ev.data.value = "world";
        // { "value": "world" } 值已经改变
        console.log("worker: %s", JSON.stringify(obj));
        postMessage("ok!")
    });
</script>
<script type="text/javascript">
    var code = document.getElementById("worker-code").innerHTML;
    var blob = new Blob(Array.from(code), { type: "text/javascript" });
    var url = URL.createObjectURL(blob);

    var worker = new Worker(url);
    var obj = { value: "hello" };

    worker.addEventListener("message", function(ev) {
        // 检查一下外部的obj
        console.log("master: %s", JSON.stringify(obj));
    });

    worker.postMessage(obj);
</script>
```

这就是输出结果，符合我上边的说法：

```text
worker: {"value":"hello"}  
worker: {"value":"world"}  
master: {"value":"hello"}
```

但这种做法并不能让我们比较满意，如果需要传输一段文件内容，那么这个序列化、复制、反序列化的过程将会比较耗时。还好，Worker提供了一种方式可以让我们某些类型的对象直接传递引用过去，但出于线程安全考虑，该对象在原来的上下文中会消失！也就是如果主线程将对象转让给Worker之后，主线程就访问不到该对象了。根据文档中描述，现阶段仅有MessagePort以及ArrayBuffer两种对象可以用于转让。

postMessage方法其实拥有两个参数，第二个参数是一个可选的数组，用于放置转让对象列表：

```html
<script type="text/code" id="worker-code">
    addEventListener("message", function(ev) {
        var typeArray = ev.data;
        // 1, 2, 3
        console.log(typeArray.toString());
        postMessage("ok");
    });
</script>
<script type="text/javascript">
    var code = document.getElementById("worker-code").innerHTML;
    var blob = new Blob(Array.from(code), { type: "text/javascript" });
    var url = URL.createObjectURL(blob);

    var worker = new Worker(url);
    var typeArray = new Int8Array([1, 2, 3]);

    worker.addEventListener("message", function(ev) {
        // Int8Array[0]
        console.dir(typeArray);
    });

    worker.postMessage(typeArray, [typeArray.buffer]);
</script>
```

示例将typeArray传递给Worker，它将被序列化。但在postMessage时却将typeArray的buffer列在转让列表中，要求该对象直接进行转让，也就是说在Worker中复制的typeArray使用的ArrayBuffer对象其实就是主线程中的typeArray使用的ArrayBuffer，它并没有经过复制。在Worker得到typeArray之后，重新发了个消息通知主线程，这时候主线程再来做检查，外部的typeArray还在（因为仅是复制了一份），但它的buffer却不见了，缓冲区变成0字节。

这里必须强调一点，转让列表中的对象必须是postMessage第一个参数的本身或者一部份，如果与它毫无关联那么在Worker内部无法得到它。

### 停止Worker

想要将一个Worker停止非常简单，worker对象提供了terminate方法，而Worker内部则可以调用close方法关闭本线程。

## 执行顺序

Worker允许加载一个同源的js文件作为它的执行代码，同时在Worker内部可以使用importScripts函数无限制地跨域加载js文件。由于加载js文件是需要时间的，在执行过程中它们是同步的还是异步的还需要进行实验：

```php
<?php
sleep(1);
header('Content-Type: application/javascript');
?>
console.log("worker load: %d", new Date().getTime());

addEventListener("message", function(ev) {
    console.log("worker accept: %s, time is %d", ev.data, new Date().getTime());
});
```

```javascript
console.log("worker create: %d", new Date().getTime());
var worker = new Worker("./worker.php?t=" + new Date().getTime());
worker.postMessage("here?")
console.log("master send: %d", new Date().getTime());
```

worker.php文件先来了个sleep(1)睡眠了1秒才开始返回数据，也就是说Worker是至少1秒后才开始工作。而主线程在建立Worker之后马上发送一句"here?"并输出时间，现在看一下运行结果，令人惊讶：

```text
worker create: 1480097663629
master send: 1480097663631
worker load: 1480097664656
worker accept: here?, time is 1480097664660
```

"worker create"与"master send"之间的时间间隔是2ms，也就是说几乎同时执行，证明建立一个Worker并不是同步操作。在1000ms后控制台出现"worker load"，证明Worker真的是1秒后才开始工作，但后边的"worker accept"却表示它接收到了"here?"。按常理来说消息是在Worker运行前一秒发的，当Worker开始运行时消息早已经丢失，因为在发送时没人接收，但事实上并非如此。

现在再将php代码稍改改看看运行结果：

```php
<?php
sleep(1);
header('Content-Type: application/javascript');
?>
console.log("worker load: %d", new Date().getTime());

setTimeout(function() {
    addEventListener("message", function(ev) {
        console.log("worker accept: %s, time is %d", ev.data, new Date().getTime());
    });
}, 100);
```

我将message事件的绑定放到setTimeout之中，也就是0.1秒后才绑事件，运行结果如下：

```text
worker create: 1480098216641
master send: 1480098216643
worker load: 1480098217667
```

这次Worker真的接收不到未执行前发送的消息了。由此可见Worker的执行代码加载确实是异步的，在Worker加载期间主线程发送的消息都会被暂存起来，直到Worker开始执行并且绑定了message事件，浏览器才会将保存起来的消息真的发送过去，但是在Worker脚本执行完之后还没碰上事件绑定，消息就真的被抛弃了，哪怕你在某个时刻终于想起要绑定message事件都无法收到消息。浏览器在这方面封装得非常棒，至少我们不需要确保Worker绑定事件后才开始发送消息，唯一需要注意的是必须在Worker的全局作用域下直接绑个message事件而不能在某个异步操作之后再绑。

需要考虑执行顺序的还有Worker内部的importScripts数，它能够加载一个js文件在当前作用域下执行，但这个我们无需试验，因为MDN上说得非常清楚：

{% blockquote MDN https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API/Using_web_workers 使用 Web Workers %}
 注意： 脚本的下载顺序不固定，但执行时会按照你将文件名传入到 importScripts() 中的顺序。这是同步完成的；直到所有脚本都下载并运行完毕， importScripts() 才会返回。
{% endblockquote %}

也就是说importScripts是一个同步操作，它允许你一次传入多个文件路径导入并执行，虽然这些文件由于网络原因加载完成的时间不一样，但Worker会保证它们在全部加载完之后再执行，而且执行顺序跟参数的顺序一致。

```javascript
importScripts("a.js", "b.js", "c.js");
// other code
```

