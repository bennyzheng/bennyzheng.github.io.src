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

# 进程生命周期

一个进程拥有五种状态，在运行期间总会处于其中的一种，并且在各种状态之间互相转换。

* 创建状态 - 当运行一个应用程序的时候，它并不是直接就开始运行，系统需要将它的必要信息读入内存，为其分配各种资源比如堆栈、代码区，这个初始化的过程被称为创建状态。
* 就绪状态 - 当一个应用程序所有资源都处于就绪状态，但由于CPU正在为其它应用程序服务时，它就处于就绪状态。当CPU空闲下来，有资源为该进程提供服务时，就绪状态就会结束进入运行状态。
* 运行状态 - 应用程序所有资源都就绪，并且CPU分配资源为它做计算时称为运行状态。
* 阻塞状态 - 应用程序可能由于某个资源未就绪而无法继续运行时就是阻塞状态，这个资源不包括CPU资源（否则就是就绪状态了）。这种状态下应用程序很可能在等待一个系统信号，比如一个socket连接。一个最简单的Web服务器在监听端口但没请求进来的时候就应该处于这种阻塞状态，它不退出，但也不占用CPU资源。阻塞状态如果由于资源到位（比如接收到一个信号），这时候就重新进入就绪状态，等到CPU资源到位进入运行状态。
* 退出状态 - 当应用程序由于没有任何需要处理的任务或者接收到退出信号时，系统将会将应用程序置为退出状态，为它释放各种资源，从内存中删除。

## 进程异常处理

Node.js环境下的应用程序在默认情况下如果发生错误将会马上报错并且退出程序，

# 进程的生命周期
process.abort
process.exit
Event: 'beforeExit'
Event: 'exit'
Event: 'rejectionHandled'
Event: 'uncaughtException'
Event: 'unhandledRejection'
process.exitCode
process.nextTick

# 读写进程信息
process.chdir
process.cpuUsage
process.cwd
process.getegid
process.geteuid
process.getgid
process.getgroups
process.getuid
process.hrtime
process.initgroups
process.memoryUsage
process.setegid
process.seteuid
process.setgid
process.setgroups
process.setuid
process.setuid
process.umask
process.uptime
process.arch
process.config
process.env
process.pid
process.platform
process.release
process.title
process.version
process.versions

# 参数处理
process.argv
process.argv0
process.execArgv
process.execPath

# 子进程
process.mainModule
child_process.exec
child_process.execFile
child_process.execFileSync
child_process.execSync
child_process.fork
child_process.spawn
child_process.spawnSync
child.disconnect
child.kill
child_process:Event: 'close'
child_process:Event: 'disconnect'
child_process:Event: 'error'
child_process:Event: 'exit'
child_process:Event: 'message'
child.kill
child.pid
child.stderr
child.stdin
child.stdio
child.stdout
options.detached
options.stdio

# 进程通讯
process.kill
process.disconnect
process.send
Event: 'disconnect'
Event: 'message'
process.channel
process.connected
child.send
child.channel
child.connected
child.disconnect



# 系统信号
Signal Events



