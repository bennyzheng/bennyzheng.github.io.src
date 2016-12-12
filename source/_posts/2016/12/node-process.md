---
title: Node.js - 进程学习笔记
categories: Node.js学习笔记
tags:
  - process
  - child_process
  - Node.js
---
我一直都是在浏览器端工作，算是个很普通的Web前端开发工程师，对于进程、线程方面的知识仅限于知识广度需要初步理解，但并没有真正去使用它。

使用Javascript写应用程序的例子不少，在Windows下可以用Javascript调用系统COM组件读写文件甚至写病毒；在ASP大家默认使用VBScript编写代码，同时也可以使用Javascript。Node.js是做得最好的一个，它让Javascript的世界变得更加丰富多彩，它是专门为Javascript搭建的舞台。

Node.js可以非常方便地编写一个应用程序，但要让它很高效地跑起来，几行代码是不足的，对进程和线程不理解也是不成的。我对这方面很感兴趣，但对应用程序是陌生的，思考过怎么学习，想来想去对着手册一点一点理解下去是最简单的。

process是Node.js提供给用户与当前进程交互的全局对象，你可以在任何地方使用它而不需要require任何模块。process是emitEvent的一个实例，意味着它拥有事件机制。我发现Node.js是个类都是emitEvent的子类，是个对象都是emitEvent的实例，毕竟Node.js许多东西都建立在事件驱动的理念上，同时也意味着在Node.js使用费时的同步操作基本上是个愚蠢的做法。

child_process模块则是Node.js提供给父进程使用的模块，想要学习进程相关的操作，process对象与child_process模块都是得一起研究的。

<!-- more -->

# 输入与输出

Javascript在许多宿主环境中由于安全限制没有得到文件读写权限，也没有标准的输入输出，但在Node.js中运行则不一样，这真正是个应用程序，因此可以像其它编程语言一般访问到标准输入输出设备。

标准IO设备其实就是三个启动应用程序时就默认打开的文件：

* process.stdin - 标准输入设备，类型是ReadStream，默认对应键盘输入，文件描述符为0。
* process.stdout - 标准输出设备，类型是WriteStream，默认对应屏幕，文件描述符为1。
* process.stderr - 标准错误设备，类型是WriteStream，默认对应屏幕，文件描述符为2。

标准IO设备允许进行重定向：

```bash
# 将标准输入重定向到filename
node test.js < filename

# 将标准输出重定向到filename
node test.js > filename

# 将标准输出重定向到filename，仅追加
node test.js >> filename

# 将标准输出重定向到filename
node test.js 2> filename

# 将标准错误重定向到filename，仅追加
node test.js 2>> filename

# 将标准错误重定向到标准输出
node test.js 2>&1
```

除了标准IO设备之外，Node.js还提供了警告相关的功能，可以通过process.emitWarning输出一个警告信息，同时也可以为进程绑定warning事件用于监听该警告。

```javascript
process.on("warning", function(ev) {
    console.log(ev);
});

process.emitWarning("this is warning");
```

这段代码将在控制台输出类似这样的内容：

```text
(node:1330) Warning: this is warning
{ Warning: this is warning
    at Object.<anonymous> (/wwwroot/github/bennyzheng.github.io.src/test.js:5:9)
    at Module._compile (module.js:573:32)
    at Object.Module._extensions..js (module.js:582:10)
    at Module.load (module.js:490:32)
    at tryModuleLoad (module.js:449:12)
    at Function.Module._load (module.js:441:3)
    at Module.runMain (module.js:607:10)
    at run (bootstrap_node.js:382:7)
    at startup (bootstrap_node.js:137:9)
    at bootstrap_node.js:497:3 name: 'Warning' }
```

其中第一行是emitWarning函数调用输出的，而在warning事件响应中把事件对象输出则会将警告的详细调用堆栈信息输出。要特别注意的是emitWarning输出的内容是输出到stderr设备中，console.log则是将内容输出到stdout。

# 生命周期

一个进程拥有五种状态，在运行期间总会处于其中的一种，并且在各种状态之间互相转换。

* 创建状态 - 当运行一个应用程序的时候，它并不是直接就开始运行，系统需要将它的必要信息读入内存，为其分配各种资源比如堆栈、代码区，这个初始化的过程被称为创建状态。
* 就绪状态 - 当一个应用程序所有资源都处于就绪状态，但由于CPU正在为其它应用程序服务时，它就处于就绪状态。当CPU空闲下来，有资源为该进程提供服务时，就绪状态就会结束进入运行状态。
* 运行状态 - 应用程序所有资源都就绪，并且CPU分配资源为它做计算时称为运行状态。
* 阻塞状态 - 应用程序可能由于某个资源未就绪而无法继续运行时就是阻塞状态，这个资源不包括CPU资源（否则就是就绪状态了）。这种状态下应用程序很可能在等待一个系统信号，比如一个socket连接。一个最简单的Web服务器在监听端口但没请求进来的时候就应该处于这种阻塞状态，它不退出，但也不占用CPU资源。阻塞状态如果由于资源到位（比如接收到一个信号），这时候就重新进入就绪状态，等到CPU资源到位进入运行状态。
* 退出状态 - 当应用程序由于没有任何需要处理的任务或者接收到退出信号时，系统将会将应用程序置为退出状态，为它释放各种资源，从内存中删除。

# 异常处理

Node.js环境下的应用程序在默认情况下如果发生错误将会马上报错并且退出程序，一般来说可以通过try-catch来捕获错误:

```javascript
setTimeout(function(){}, 5000);

try {
    throw new Error("my error");
}catch(ex) {
    console.log(ex.message);
}
```

try-catch会将异常捕获，并且把ex当参数传给catch代码块，可以在里边做异常处理。当然，如果有需要，在catch代码块中对异常做完处理之后也可以再使用throw抛出错误。

除了try-catch，进程对象还提供了uncaughtException事件用于捕获未使用try-catchr捕获的异常并对其做统一处理：

```javascript
setTimeout(function(){}, 5000);

process.on("uncaughtException", function(ev) {
   console.error('报错了');
});

throw new Error("error");
```

除了uncaughtException还有另一个事件unhandledRejection也极其类似，它与uncaughtException不同的是uncaughtException用于捕获同步代码抛出的异常，而unhandledRejection用于捕获Promise代码中的异常。

```javascript
process.on("unhandledRejection", function() {
    // 控制台将会输出unhandledRejection
    console.error("unhandledRejection");
});

process.on("uncaughtException", function() {
    // 不会输出，因为不会到这来
    console.error("uncaughtException");
});

var promise = new Promise(function(resolve, reject) {
    throw new Error("promise err");
});
```

由于Promise是异步的，在某一步异步处理中抛出异常将会触发unhandledRejection事件，但如果在触发unhandledRejection之后又使用catch方法捕获Promise抛出的异常（也就是出现了没及时处理的promise异常），这时候异常会被catch方法捕获并且同时触发rejectionHandled事件。

```javascript
process.on("unhandledRejection", function() {
    console.error("unhandledRejection");
});

process.on("rejectionHandled", function() {
    console.error("rejectionHandled");
});

process.on("uncaughtException", function() {
    console.error("uncaughtException");
});

var promise = new Promise(function(resolve, reject) {
    throw new Error("promise err");
});

setTimeout(function() {
    promise.catch(function(err) {
        console.log("迟来的Promise捕获");
    });
}, 1000);
```

输出结果为：

```text
unhandledRejection
rejectionHandled
迟来的Promise捕获
```

*要特别注意的是当异常未被捕获，该进程会退出，进程的异常退出不会影响父进程但会影响子进程。如果子进程异常退出，将会于stderr输出错误信息并退出，父进程将收到子进程退出的消息；如果父进程异常退出，那么所有子进程将会被中断并且退出，除非子进程是一个独立进程（查阅options.detached）。*

```javascript
const cluster = require('cluster');

if (cluster.isMaster) {
    cluster.fork();

    setTimeout(function() {
        // 虽然子进程退出了，但主进程依然于5秒后输出Ok，它不受影响
        console.log("Ok");
    }, 5000);
} else {
    throw new Error("child exit!");
}
```

```javascript
const cluster = require('cluster');

if (cluster.isMaster) {
    cluster.fork();

    setTimeout(function() {
        // 1秒后主进程退出
        throw new Error("master exit!");
    }, 1000);
} else {
    setTimeout(function() {
        // 由于父进程的退出，子进程不会再输出Ok，因为它也退出了。
        console.log("Ok");
    }, 5000);
}
```
# 退出处理

当一个Node.js应用程序在没有任何需要处理的任务时将会退出运行，但也可以主动调用process对象的exit方法退出程序，同时指定退出时的状态代码。状态代码默认为0，表示程序是正常退出，非0则表示异常退出，调用该应用程序的父进程（或控制台）可以获取到该退出状态代码的值以做其它处理。

```javascript
setTimeout(function() {
    // 由于exit的调用，导致这里不会被执行。
    console.log("ok");
}, 1000);

process.exit();
```

exit方法一般来说不应该被调用，它会要求应用程序以最快的方式停止程序的运行，这会导致事件循环中的任务不再被处理，同时可能会导致数据丢失，比如stdout与stderr输出一半就停止。建议在没有把握的情况下不要调用exit方法，如果需要设置退出状态码可以为process.exitCode赋值：

```javascript
setTimeout(function() {
    console.log("ok");
}, 1000);

// 不要这么干
// process.exit();
process.exitCode = -1;
```

当应用程序退出时将会触发process对象的exit事件，但请不要在exit事件里做异步调用，绑定该事件不会等待事件循环清空再退出应用程序。

```javascript
process.on("exit", function() {
    console.log("exit!");

    setTimeout(function() {
        console.log("这句代码不会被执行！");
    }, 1000);
});
```

如果希望在应用程序退出之前做异步操作那么可以使用beforeExit事件，它允许在事件响应函数中为事件循环添加新的任务，并且中止退出操作。但需要注意的是，如果使用的是exit方法退出应用程序，那么它不会被触发，必须是由于事件循环中没有任何任务而自然退出的情况才会触发beforeExit事件：

```javascript
process.on("beforeExit", function() {
    console.log("exit!");

    setTimeout(function() {
        console.log("beforeExit");
    }, 1000);
});

```

上边这个示例的beforeExit事件响应函数使用setTimeout为事件循环添加了一个任务，导致应用程序中止退出，并且正常情况下应用程序永远都不会退出成功，因为每次要退出都会触发beforeExit事件并被中止。因此使用beforeExit事件时一定要注意不要形成死循环，导致应用程序无法退出。

```javascript
process.on("beforeExit", function() {
    console.log("exit!");

    setTimeout(function() {
        console.log("beforeExit");
        // 使用exit方法就不会触发beforeExit形成死循环了
        // 但这依然不是真正安全的，比如还有数据没被写到磁盘导致数据丢失。
        process.exit(0);
    }, 1000);
});
```

主动退出应用程序还有另一个与exit类似的方法abort，它表示中断程序运行，同时会给出一个核心文件内容以供开发者分析。

# 延迟执行任务

在写代码时有时候有些任务不需要同步执行，可以使用setTimeout(xxx, 0)来做延迟执行，但Node.js提供了更有效率的方法：process.nextTick。

Node.js的事件循环队列其实可以简单理解成一个数组（当然真实情况没这么简单），每个元素是一个函数，它代表着一个需要被执行的任务。process会取出该数组然后做一次遍历，将每个函数执行一次。当process.nextTick被调用时，任务会被添加到数组的末尾，在它之前的任务全部执行完之后才会执行。

```javascript
setTimeout(function() {
    console.log("1秒")
}, 1000);

setTimeout(function() {
    console.log("2秒");
}, 2000);

console.log("我要输出1");

process.nextTick(function() {
   console.log("什么时候输出？");
});

console.log("我要输出2");
```

输出结果为：

```text
我要输出1
我要输出2
什么时候输出？
1秒
2秒
```

从这里可以看出，text.js的主体代码是一个任务，"我要输出1"以及"我要输出2"最先被执行，然后执行nextTick注册的任务。setTimeout注册的任务并不会马上被执行，它并不在事件循环中，使用的是Javascript内部的任务队列。

# 参数处理

进程对象提供了几个属性用于读取进程的参数信息：

* process.argv

argv包含所有参数的数组，它的第一个元素是node.js可执行文件的路径，第二个元素是当前进程文件的路径，后边再接着传给本进程的各种参数，以空格分隔。

```javascript
console.log(process.argv);
```

运行"node test.js --help"之后输出结果是：

```text
[ '/usr/local/bin/node',
  '/wwwroot/github/bennyzheng.github.io.src/test.js',
  '--help' ]
```

process对象还提供了argv0属性方便我们获取第一个参数（有这必要么……），不过argv0存储的是参数的原始值，比如"node test.js"，它的值不是'/usr/local/bin/node'，而是'node'。

在启动进程的时候我们往往是使用node命令指定脚本文件启动，这时候可能会给node传参数，这些参数可以使用execArgv获取到：

```javascript
console.log(process.execArgv);
```

执行"node -i test.js"运行结果如下：

```text
[ '-i' ]
```

PS:建议使用argv模块来处理参数相关的事务，它可以在[npmjs站点的argv相关页面](https://www.npmjs.com/package/argv)获取，使用npm install argv安装。

# 子进程

## 建立子进程

建立一个子进程相当于执行磁盘上的某个可执行文件，把该可执行文件建立起来的进程当做当前进程的子进程。

* exec(command[, options][, callback])
* execSync(command[, options])

启动一个shell运行一段命令，由于是shell执行因此可以做很多复杂的事情比如建立管道、输入输出流重定向。

```javascript
// test.js
var child_process = require("child_process");

child_process.exec("./test.sh", function(error, stdout, stderr) {
    // 如果test.sh没有可执行权限则会报权限不足
    // chmod u+x ./test.sh
    if (error) {
        console.error(stderr);
        return;
    }
    
    console.log(stdout);
});
```

```bash
# test.sh
echo 'hello'
```

* execFile(file[, args][, options][, callback])
* execFileSync(file[, args][, options])

execFile与exec比较接近，但它不会启动一个shell来解析命令，因此它无法建立管道、重定向IO。由于不会新建shell，因此相对exec来说execFile更高效。

```javascript
var child_process = require("child_process");

child_process.execFile("./test.sh", function(error, stdout, stderr) {
    // 如果test.sh没有可执行权限则会报权限不足
    // chmod u+x ./test.sh
    if (error) {
        console.error(stderr);
        return;
    }

    console.log(stdout);
});
```

要特别注意的是，据称windows下如果没有shell则无法运行.bat之类的shell脚本（未实测，哥没win系统），也就是execFile无法用于运行.bat脚本，但可以这么写（我个人觉得没必要，还不如直接使用exec方法）：

```javascript
var child_process = require("child_process");

child_process.execFile('cmd.exe', ["./test.sh"], function(error, stdout, stderr) {
    if (error) {
        console.error(stderr);
        return;
    }

    console.log(stdout);
});
```

* spawn(command[, args][, options])
* spawnSync(command[, args][, options])

spawn默认不会新建shell，但你可以将options的shell选项设置为true或者设置为某个shell的地址启用shell，在这方面算是比较自由。但这并不是spawn与exec区别最大的地方，exec/execFile会将命令执行完再将stdout/stderr的内容当成参数传递给回调函数，它们能够传递的内容比较有限，在没更改options的设置值的情况下是200KB的缓冲区，如果产生的内容超出200KB则会报错。spawn则可以通过stdout/stderr流来获取数据，比如调用curl下载远程文件并保存内容。

```javascript
var child_process = require("child_process");
var child = child_process.spawn("./test.sh");

// hello
child.stdout.on("data", function(chunk) {
   console.log(chunk.toString());
});

// 如果test.sh没有可执行权限，则会报EACCESS错误
child.stderr.on("data", function(chunk) {
    console.error(chunk.toString());
});
```

* child_process.fork(modulePath[, args][, options])

fork一个有Node.js特色的建立子进程的方式，它允许指定一个js文件作为启动代码在Node.js环境下执行，这是spawn方法的特殊应用场景，子进程将与父进程建立IPC通道用于进程通讯，而fork方法返回的子进程对象的stdio将不会被赋值，父进程与子进程将以IPC通道进行通讯。

```javascript
// test.js
var child_process = require("child_process");
var child = child_process.fork("./child.js");

child.on("message", function(message) {
    console.log("message: %s", message);
});
```

```javascript
// child.js
process.send("this is child!");
```

要注意的是fork是不能通过child.stdout来获取数据的，子进程与父进程是通过IPC通道进行通讯。另外，fork它并不是将当前进程复制一份作为子进程（其它语言都是这么干的），而是指定一个模块建立一个子进程。

## options.stdio

建立子进程时可以通过设置options参数的stdio指定子进程使用的IO，它的值只可以是一个字符串用于一次性为子进程指定输入输出流也可以使用数组分开为stdin/stdout/stderr指定值，这个参数也是父进程与子进程进行通讯的重要基础。

### pipe - 流管道

默认值，相当于['pipe', 'pipe', 'pipe']，它会将ChildProcess的输入当成子进程的输入的管道上流，同时将ChildProcess进程的输出当成子进程输出的管道下流。

```javascript
// test.js
var child_process = require("child_process");
var child = child_process.spawn("node", ['./child.js'], {
    stdio: 'pipe'
});

// child.stdin.pipe(子进程.stdin)
child.stdin.write("hello child!");

child.stdout.on("data", function(chunk) {
    console.log("父进程的stdout接收到: " + chunk.toString());
});
```

```javascript
// child.js
// process.stdout.pipe(ChildParent.stdout)
// stderr同理
process.stdin.on("data", function(chunk) {
    process.stdout.write("子进程的stdin接收到: " + chunk.toString());
});
```

这个示例调用了child.stdin.write向子进程的stdin写入了"hello child!"，而子进程监听了process.stdin的data事件，然后把数据写入process.stdout，process.stdout则将数据再交给父进程的child.stdout，通过child.stdout的data事件获取，到此完成了双向通讯，输出结果如下：

```text
父进程的stdout接收到: 子进程的stdin接收到: hello child!
```

### inherit - 共享IO

如果将stdio设置为'inherit'，则可以让子进程共享父进程的标准IO，二者共用同一套stdio。相当于[process.stdin, process.stdout, process.stderr]或者[0, 1, 2]（因为标准IO套接字就是0、1、2）。

需要注意的是，如果分开设置标准IO，可以将其中一个输入输出指定成某个流或者套接字，比如一个socket stream。

### ipc - 进程间通信

当stdio是一个数组，并且拥有ipc值的时候将为开启IPC通道（仅能开一个），子进程将可以使用process.send向父进程发送消息，父进程可以通过监听ChildProcess的message事件来获取该消息。使用fork函数建立子进程时默认开启双向IPC通道，父进程也可以调用ChildProcess的send方法向子进程发送消息。 

```javascript
var child_process = require("child_process");
var child = child_process.spawn("node", ['./child.js'], {
    stdio: [ 'ipc']
});

child.on("message", function(message) {
    console.log("master: " + message);
});

child.send("hello child!");
```

```javascript
// child.js 
process.send("hello master!");

process.on("message", function(message) {
    console.log("child:" + message);
});
```

运行node test.js之后，将能够看到输出的内容是：

```text
master: hello master!
```

这个结果看起来并不符合我们的本意，因为主进程调用了child.send向子进程发送"hello child!"却没有输出来，但这是正常的，因为子进程压根就没有开启标准输入输出，test.js代码需要做一下小调整：

```javascript
var child_process = require("child_process");
var child = child_process.spawn("node", ['./child.js'], {
    // 与子进程共享本进程的标准IO，子进程也能够向控制台输出信息了
    stdio: [0, 1, 2, 'ipc']
});

child.on("message", function(message) {
    console.log("master: " + message);
});

child.send("hello child!");
```

### ignore - 忽略IO

当给子进程设置标准IO为ignore时，子进程将会把标准IO指向/dev/null，Unix会忽略对这个空设备的读写操作，相当于没有打开任何IO设备，可以单独为某个IO设置ignore值，比如[0, 'ignore', 'ignore']。

## options.detached

detached默认为false，意味着子进程依附父进程运行，当父进程中断退出时子进程也会随之退出，父进程也会等待所有子进程退出运行才退出运行。如果将detached设置为true，那么父进程退出时子进程是不会退出运行的，它是一个独立的进程，父进程调用child.unref方法可以让子进程从父进程的事件循环中删除，父进程可以独立退出运行而不影响子进程。

```javascript
// test.js
var child_process = require("child_process");
// var fs = require("fs");
// var outFile = fs.openSync("./out.log", "a");
var child = child_process.spawn("node", ['./child.js'], {
    // stdio: ['ignore', outFile, 'ignore'],
    detached: true,
    stdio: 'inherit'
});

child.unref();
console.log("master exit!");
```

```javascript
setTimeout(function() {
    console.log("child exit!");
}, 1000);
```

有文章说stdio必须设置为ignore或者另开一个套接字，不能使用父进程的stdio，否则会导致高用child.unref方法时父进程还会等待子进程退出才退出，但我实际测试了并没有这种情况，或许是版本问题，我使用的是v7.0.0，如果发生这种情况可以按示例中注释掉的代码将子进程的stdout重定向到一个文件中。

## 子进程事件

事件是子进程主动对外（父进程）发送消息的重要手段，ChildProcess对象提供了5种事件给父进程绑定：

* close - 当子进程退出时，如果stdio同时也关闭将会触发该事件。要特别注意的是，如果stdio没有关闭，则不会触发该事件，比如子进程与父进程共享stdio。
* exit - 当子进程退出时触发该事件，并且告诉父进程退出信号代码。
* disconnect - 当子进程与父进程之间的IPC通道关闭时触发。
* error - 当子进程无法创建、无法kill以及无法发送消息的时候分发该事件。需要特别注意的是当发生错误退出时未必会分发exit事件，也可能会分发exit事件。
* message - 当子进程调用process.send向父进程发送消息时分发


# 进程信号

进程在支持POSIX的系统中实现了系统信号，具体列表请查阅POSIX信号列表。process可以监听部份允许监听的系统信号做出相应的处理，但也有很多信号是无法监听直接执行默认行为的。

以下是Node.js手册提供的部份信号解释：
* SIGUSR1 node.js 接收这个信号开启调试模式。可以安装一个监听器，但开始时不会中断调试。
* SIGTERM 和 SIGINT 在非 Windows 系统里，有默认的处理函数，退出（伴随退出代码 128 + 信号码）前，重置退出模式。如果这些信号有监视器，默认的行为将会被移除。
* SIGPIPE 默认情况下忽略，可以加监听器。
* SIGHUP 当 Windowns 控制台关闭的时候生成，其他平台的类似条件，参见signal(7)。可以添加监听者，Windows 平台上 10 秒后会无条件退出。在非 Windows 平台上，SIGHUP 的默认操作是终止 node，但是一旦添加了监听器，默认动作将会被移除。 SIGHUP is to terminate node, but once a listener has been installed its
* SIGTERM Windows 不支持, 可以被监听。
* SIGINT 所有的终端都支持，通常由CTRL+C 生成（可能需要配置）。当终端原始模式启用后不会再生成。
* SIGBREAK Windows 里，按下 CTRL+BREAK 会发送。非 Windows 平台，可以被监听，但是不能发送或生成。
* SIGWINCH - 当控制台被重设大小时发送。Windows 系统里，仅会在控制台上输入内容时，光标移动，或者可读的 tty在原始模式上使用。
* SIGKILL 不能有监视器，在所有平台上无条件关闭 node。
* SIGSTOP 不能有监视器。

可以利用kill方法向一个进程发送信号：

```javascript
var child_process = require("child_process");
var child = child_process.spawn("node", ['./child.js'], {
    stdio: 'inherit'
});

setTimeout(function() {
    child.kill("SIGTERM");
}, 1000);
```

```javascript
process.on("SIGTERM", function() {
    console.log("父进程发送SIGTERM信号要求子进程退出，但我不退出！");
});

process.on("exit", function() {
    console.log("子进程退出了");
});

setTimeout(function() {}, 3000);
```

可以看出，kill方法并不是真的把子进程杀死，而是起到一个发送信号的作用，具体怎么处理完全取决于子进程。但并不是所有信号都会有给子进程反应的机会，如果将上边示例的kill调用参数换成"SIGKILL"，那么子进程完全没有输出任何信息的机会直接退出运行。



