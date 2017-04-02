---
title: Node.js - 探索数据流
categories: Node.js学习笔记
date: 2016-11-16 00:00:00
tags:
  - Node.js
  - Stream
  - Buffer
  - Binary
---

近期一位同事问起数据流方面的问题，在做前端开发时很少直接面对数据流，只有在做node.js相关的东西会接触得比较多。我当时简单做了一些解答，但我认为我解答得不是很好，至少她还是一脸的迷茫。于是我准备在这里试着讲清数据流的来龙去脉。

<!-- more -->

## 认识数据流

数据流是数据操作的抽象，它以统一的API、高效的内部实现让数据操作者很方便地操作多种设备上的数据，简单一点可以把数据流当成一组操作数据的API。

数据流并不是一个新概念，Unix系统将很多硬件设备挂载成设备文件，然后提供了一组低级I/O API用于数据交换，如果学过C语言那么这组API肯定会非常熟悉，许多数据(不仅仅是磁盘文件)的读写都可以用它们来做：

```c
int open(const char *path, int oflag, ...);
ssize_t read(int fildes, void *buf, size_t nbyte);
ssize_t write(int fildes, const void *buf, size_t nbyte);
int close(int fildes);
```

可以看出Unix在数据读写方面已经考虑到了API的统一，这组I/O API被称为低级I/O，它们直接使用了系统调用来直接访问硬件进行数据读写，而且还不带缓冲区，这意味着每次调用都将读写一次设备，频繁的读写不仅存在性能问题还会影响到设备的寿命，大家在写C程序的时候一般使用的是标准I/O API：

```c
FILE *fopen(const char *restrict filename, const char *restrict mode);
size_t fread(void *restrict ptr, size_t size, size_t nitems, FILE *restrict stream);
size_t fwrite(const void *restrict ptr, size_t size, size_t nitems, FILE *restrict stream);
int fclose(FILE *stream);
```
标准I/O的底层实现依然是调用了低级I/O，并且在低级I/O上再做了一层封装，实现了缓冲区，允许数据预读、缓写等缓冲功能，减少了与设备打交道的次数，也提高了性能。
请注意标准I/O的原型声明，FILE \*类型的变量被命名为stream。由于C不是面向对象的，File \*stream代表着数据，加上配套的标准I/O共同组成了数据流。**数据流对原始数据操作做了一层封装，使其更合理、高效并且拥有了统一的API**。

## 数据流的使用

Java语言给流抽象了四种类型:

* InputStream - 二进制输入流，应用程序可以使用该流从某个数据设备读取二进制数据
* OutputStream - 二进制输出流，应用程序可以使用它往某个数据设备写入二进制数据
* Reader - 字符输入流，与InputStream不同的是它操作的是字符串
* Writer - 字符输出流，与OutputStream不同的是它写入的是字符串

事实上，Reader和Write看起来更像是InputStream、OutputStream的一种封装，二者仅仅操作的是数据类型不一样，字符串也同样是二进制的。

而我们熟悉的Node.js同样给流抽象了四种类型：

* Readable - 拥有读能力的数据流
* Writable - 拥有写能力的数据流
* Duplex - 双向流，是Readable以及Writable的合体
* Transform - 对Duplex的封装，允许写入数据后对数据做修改，重新读出来的数据将是修改后的数据

*从Java以及Node.js对流的定义上来看，数据流并没有一个固定的形式，但不外乎用于读和写，并且根据实际场景流可以同时拥有读和写的功能。数据流将数据使用者与数据设备隔离开来，数据使用者不需要关心数据设备接口是如何工作的，只需要调用数据流提供的统一的API即可完成操作。*

对于一名前端工程师来说弄懂Node.js的数据流帮助更大，以下就以Node.js的数据流来做一些实验。

### Readable

fs模块提供了文件读写功能，其中有一个方法readFileSync非常方便，它可以一次性把文件内容返回给应用程序，示例代码如下：

```javascript
var fs = require("fs");
var content = fs.readFileSync("./test.js", {
    "encoding": "utf-8",
    "flag": "r"
});
console.log(content);
```

看起来非常方便，但对于小文件可以这样，对于大文件（比如动不动就上G的视频文件）如果还这么做分分钟内存溢出。

换种做法可以解决这种问题，比如使用fs.open/fs.read，但使用数据流是最简单的方法。

Readable以及其它数据流是以事件驱动来与应用程序互动的，它提供了以下几个事件：

* close - 流关闭时分发close事件，当一个流被关闭之后就不再生产数据，即变成不可读状态。手册中提示并不是所有的只读流都会分发该事件，它没有任何参数。
* data - 在缓冲区有数据时分发data事件，拥有一个chunk参数。当data事件分发时，chunk参数默认会得到一个<Buffer>对象，如果数据源提供的是字符串数据，则可以通过setEncoding函数设置文本编码，这时候chunk将会是一个字符串。
* end - 当数据源不再生产数据时（比如磁盘文件读到结尾）分布本事件。
* error - 当操作发生错误时触发。
* readable - 当缓冲区有数据时分发readable事件，但需要主动调用read([size])函数读取数据。

除了事件还提供了以下方法：

* isPaused - 检测流是否处于暂停状态，只有显式调用了pause方法才会让流转入暂停状态，不再从数据源读取数据。
* pause - 显式要求流转入暂停状态，在暂停状态下不再从数据源读取数据，也不会触发data、readable事件。
* pipe - 非常重要的方法，它能够传入一个允许写入的流（除了Readable，其它流都允许写入）当作流数据的目的地，成为一个管道。Readable将会自动读取数据，并且将它写入Writable流中。
* read - 主动从数据缓冲区读取数据，可以显式传入size参数要求读多少字节，如果不传则是读取缓冲区全部数据。
* resume - 显式要求流从暂停状态转入流动状态。
* setEncoding - 如果确认数据源是字符串，那么可以使用setEncoding方法设置文本编码。要注意的是如果设置了文本编码，那么data事件的chunk参数或者read方法都将返回String而不是Buffer。
* unpipe - 将管道断掉，pipe的反操作。
* unshift - 将数据块重新塞回缓冲区，可以对数据块做修改。参考手册提示如果有这类需求应该考虑使用Transform流，个人觉得也确实如此，如果使用这个函数就失去了只读的意义了。
* wrap - 允许传入一个旧数据流的实现，将它做为数据源使用新的流API读取。Node.js在之前有一套旧的流实现，这主要是做代码兼容之用。
* push - 将数据添加到Readable的数据缓冲区，这个函数是为扩展Readable而实现的，除非你实现Readable的子类，否则它不应该被使用。

*一个刚打开的Readable流默认处于流动状态，调用isPaused函数可以看到值确实为false，但网上许多资料说刚打开的Readable是处于暂停状态，要绑定data、调用resume函数之类的操作才会让流进入流动状态。经过试验，在流刚打开时，stream._readableState.flowing的值为null，绑定data事件后会变成true，调用pause函数后变回false。也就是说相当于暂停状态，这么看就合理了，它确实不是暂停状态（值不是false），但也同时不是流动状态，但可以把这种默认状态当成暂停状态处理。*

EventEmitter类是Node.js实现事件模型的基类，所有数据流都是它的子类，数据流的读操作也基本上是靠事件来工作。读操作一般是使用data+end或者readable+end事件来完成，或许在这里会有疑问为什么会提供了如此相似的两个事件。

data事件会将缓冲区第一个Buffer对象作为参数传给事件响应函数，如果使用了setEncoding方法设置了文本编码它将会将Buffer对象转成字符串作为参数，这是Readable主动取出数据交给响应函数。readable事件与data事件不同，readable仅仅是在数据缓冲区有数据可读时调用响应函数，然后由应用程序自己主动去调用read读取数据。

data事件以及pipe方法优先级会比readable事件更高，如果绑定了data事件或者调用pipe方法建立了数据管道，在这种情况下每次读取到数据的时候会触发data事件，但在所有数据都读取完之后会触发一次readable，并且调用read方法会得到null。

如果仅仅是将数据读到内存那么使用data事件妥妥的很方便，如果数据将会被写入到磁盘或者其它写入速度可能比读取速度慢的设备时则需要特别考虑到读写速度不同导致的问题，必须等到数据写完才能够读下一个缓冲区的数据。data事件可以将数据块chunk交给应用程序，应用程序得到chunk之后直接调用pause函数暂停流的读取，然后异步写入数据，在写完数据后再显式调用resume函数重新让流开始读取数据。而readable事件则可以由应用程序决定一次读多少数据到应用程序自己管理的缓冲区（其实就是内存中的一个变量），同样暂停流写入数据再恢复数据流的流动。也就是说readable看起来麻烦了一点，但灵活性更高，毕竟用户可以决定读多少数据。

以下是读取一个文件的例子：

```javascript
var fs = require("fs");
var stream = fs.createReadStream("./test.js");
var content = "";

stream.setEncoding("utf-8");
stream.on("data", function(chunk) {
    content += chunk;
});

stream.on("end", function() {
    console.log("content");
    stream.close();
});
```

在该例子中，首先使用了setEncoding设置该文件为文本文件，并且编码是utf-8。data事件不断触发，将数据追加到内存中，在end的时候关闭流，这是最普通的文件读取例子。

同样，可以使用readable来实现同样的功能：

```javascript
var fs = require("fs");
var stream = fs.createReadStream("./test.js");
var content = "";

stream.setEncoding("utf-8");
stream.on("readable", function() {
    var buffer = this.read();
    
    if (buffer != null) {
        content += buffer;
    }
});

stream.on("end", function() {
    console.log(content);
    stream.close();
});
```

这里要特别注意一点，readable在读到文件结尾的时候，还会触发一次，并且调用read函数读到的数据为null，所以要判断一下。在手册中特别提到，read函数应该仅在流处于暂停状态下才被调用，但在上边的示例中并没有显式调用pause方法将流暂停，流的状态值默认是null。如果手动在绑定readable事件之前调用resume，那么在事件响应中调用read方法将会得到null值，因为读取的数据块已经通过data事件分发出去了，哪怕你没监听data事件。

在以下示例中读取一个简单的js文件，read方法将得到null。

```javascript
var fs = require("fs");
var stream = fs.createReadStream("./test.js");

stream.setEncoding("utf-8");
stream.resume();

stream.on("readable", function () {
    // false true null
    console.log(this.isPaused(), this._readableState.flowing, this.read());
});

stream.on("end", function() {
    stream.close();
});
```

之前提到过写的速度没有读的速度快时，我们需要对Readable流读取的速度进行限制，由于尚未讲到Writable流，因此直接使用fs.appendFile来实现：

```javascript
var fs = require("fs");
var stream = fs.createReadStream("./test.js");

stream.setEncoding("utf-8");
stream.on("data", function ondata(chunk) {
    stream.pause();

    fs.appendFile("./test.dist.js", chunk, function() {
        stream.resume();
    });
});

stream.on("end", function() {
    stream.close();
});
```

fs.appendFile是个异步操作，在触发data事件的时候先让数据流暂停（这时候不再触发data事件），然后开始异步写数据，写完数据调用resume让数据流重新开始流动。

### Writable

Writable流允许向一个数据设备写入数据，它拥有以下事件：

* close - 数据流关闭时触发，数据流关闭后就不能向其写入数据。
* drain - 数据流缓冲区数据量达到高水位（highWaterMark）后需要暂停往Writable写入数据，Writable缓过劲来的时候将会触发本事件，通知外部可以继续往数据流里写数据。
* error - 数据流写入数据失败或者其它错误时触发该事件。
* finish - 当数据流被调用了end方法之后触发，这时候所有数据已经从缓冲区写入数据设备。
* pipe - 如果本Writable流被当成参数传入某个Readable流作为数据管道的下游时，Readable往Writable写入数据时将会触发本事件，表示管道有数据流入。
* unpipe - Readable断开与Writable的管道关系时触发

以下是Writable提供的方法

* cork - 塞住数据流与数据设备的数据交换，这时候所有写入Writable的数据将被存放到缓冲区，除非调用了uncork或者end方法才会将缓冲区的数据写入数据设备。
* end - 要求数据流清空缓冲区，将所有数据全部写入数据设备。调用该方法后就不允许再使用write方法写入数据。
* setDefaultEncoding - 当数据来源是文本数据时可以为其设置一个默认的编码，如果没有设备过则表示数据是Buffer形式。
* uncork - cork的反操作，允许数据流重新开始将保存在缓冲区的数据写入数据设备。
* write - 往数据流写入数据，数据可能会马上被写入数据设备，也可能保存到缓冲区

Writable跟Readable在使用上略有不同，Readable基本上通过监听data或readable事件来被动获取数据，而Writable则是反过来被动地被调用write方法写入数据。

一个最简单的写数据例子如下：

```javascript
var fs = require("fs");
var stream = fs.createWriteStream("./test.dist.js");

stream.setDefaultEncoding("utf-8");
stream.write("hello world");
stream.on("end", function() {
    stream.close();
});
```
外部向数据流写入数据的速度如果比数据流写入数据设备的速度快时，来不及写入的数据将会被放到缓冲区。缓冲区有一个名为高水位（highWaterMark）的限制，如果数据流是对象模式（二进制，数据以Buffer存放）则默认是16个Buffer对象，如果是文本模式（数据以String的形式存放）则默认是16kb。当超过缓冲区高水位时，写入的数据依然会被放到缓冲区，但这时候write方法将会返回false，通知写入者必须缓一缓了。暂停写入数据后，Writable会继续往数据设备写数据，在缓冲区的数据清空后将会分发drain事件，通知写入者可以继续调用write写数据。

使用drain事件可以将Readable中对应的例子简单改改：

```javascript
var fs = require("fs");
var stream = fs.createReadStream("./test.js");
var wstream = fs.createWriteStream("./test.dist.js");

stream.setEncoding("utf-8");
wstream.setDefaultEncoding("utf-8");

stream.on("data", function(chunk) {
    if (!wstream.write(chunk)) {
        this.pause();
    }    
});

stream.on("drain", function() {
    this.resume();
});

stream.on("end", function() {
    this.close();
    wstream.close();
});
```

事实上还有更好的办法可以处理这种情况，比如直接使用Readable的pipe方法让Readable与Writable配对形成管道，Readable读出来的数据会直接写入Writable，同时它会处理好各种情况而不需要工程师自己操心，从这方面上看，pipe简直就是节省代码的神器：

```javascript
var fs = require("fs");
var stream = fs.createReadStream("./test.js");
var wstream = fs.createWriteStream("./test.dist.js");

stream.setEncoding("utf-8");
wstream.setDefaultEncoding("utf-8");
stream.pipe(wstream);
```

### Duplex & Transform

看了Node.js关于双向流的手册，里边介绍非常简单，双向流是Readable和Writable的合体，也就是说我们可以把它当Readable使用也可以当Writable使用，二者有的事件与方法双向流也都有，因此在这里不再重复说明。

双向流要特别注意的一点就是：它既然是Readable和Writable的合体，因此它可以利用pipe形成一个管道链，最简单的就是对数据做压缩并且输出到一个文件中：

```javascript
var fs = require("fs");
var zlib = require("zlib");
var gz = zlib.createGzip();
var stream = fs.createReadStream("./test.js");
var wstream = fs.createWriteStream("./test.js.gz");

stream.setEncoding("utf-8");
stream.pipe(gz).pipe(wstream);
```

示例中，gz对象是zlib创建的一个Transform流，它的工作就是外部写入数据，然后它对其做压缩处理后输出给wstream写入文件中，这种方式在gulp脚本中应用非常多。

## 数据流的扩展

四种数据流都是基类，类似fs.createReadStream返回的数据流其实都是它们的子类。要实现自己的数据流，需要继承相应的数据流，并且实现关键方法：

* Readable: _read
* Writable: _write, _writev
* Duplex: _read, _write, _writev
* Transform: _transform, _flush

### Readable

Readable可以使用new Readable创建一个子类，并传入这些参数：

* highWaterMark - 缓冲区高水位限制，当读取的数据放入缓冲区后超过高水位则暂缓读入，直到缓冲区数据被外部读取消耗掉。
* encoding - 如果数据源是文本数据，则可以传入编码，将数据流从默认的对象模式转为文本模式，使用read方法或者data事件得到的数据块将使用该编码解码，并且类型都将变成String。
* objectMode - 是否使用对象模式，如果值为false则表示数据源是文本数据，将使用默认编码将Buffer转成String。
* read - 实现私有方法_read，实现数据流自定义的数据读取方式，该方法由数据流本身调用，不需要考虑缓冲区的问题。

以下是使用new Readable创建一个新的只读数据流的示例：

```javascript
var fs = require("fs");
var util = require("util");
var Readable = require("stream").Readable;

const FileReadString = new Readable({
    read: function(size) {
        // 在这里实现读取数据的具体实现
    }
});
```

我个人认为使用new Readable方式创建子类的形式很不灵活，更倾向于使用Node.js用于做类型扩展的工具函数util.inherits创建它的了类，以只读文件为例实现如下：

```javascript
var fs = require("fs");
var util = require("util");
var Readable = require("stream").Readable;

function FileReadStream(path, options) {
    // 必须调用基类的构造函数，将配置信息传进去
    Readable.call(this, options);

    this._path = path; // 文件路径，为了省事，不考虑该文件不存在的情况
    this._offset = 0; // 当前文件游标在文件中的位置
    this._fd = null; // 文件描述符
}

util.inherits(FileReadStream, Readable);

/**
* 重写_read方法
* 由于_read应该是一个同步操作，所以这里使用的都是同步API
*/
FileReadStream.prototype._read = function(size) {
    // 第一次使用时打开文件
    if (this._fd == null) {
        this._fd = fs.openSync(this._path, "r");
    }

    // 如果不指定需要读多少数据，则默认读16kb。
    size = size == null ? 1024 * 16 : size;
    
    var buffer = Buffer.alloc(size);
    var len = fs.readSync(this._fd, buffer, 0, size, this._offset);
    this._offset += len;

    if (len == 0) {
        // 如果读不出数据则调用push(null)告诉数据流已经没有数据了
        this.push(null);
        // 关闭掉文件
        fs.closeSync(this._fd);
        this._fd = 0;
        // 分发事件，告诉外部数据流已经关闭
        this.emit("close");
    }
    else {
        // 将读取的二进制数据插入到缓冲区，是否转成文本那是Readable自己的事
        this.push(buffer);
    }

}

var stream = new FileReadStream("./test.js");
stream.setEncoding("utf-8");

stream.on("data", function(chunk) {
    console.log(chunk);
});
```

除了这两个方法，还可以使用es6的extends来扩展，手册中有不再详说。

### Writable

创建一个Writable的子类可以传入以下参数：

* highWaterMark - 写缓冲区的高水位，如果写入的数据超出高水位的限制，那么Writable的write方法将会返回false通知外部需要暂停写入，当数据从缓冲区写入到数据设备后将会触发drain事件通知外部可以重新写入数据。
* decodeStrings - 要求Writable在写入数据时是否先将Buffer以这里设置的编码转成字符串再调用_write方法。
* objectMode - 是否启用对象模式，如果启用写入的数据将是一个Buffer。
* _write - 将数据写到数据设备的具体实现。
* _writev - 批量写入多个数据，它并不是必须实现的，但可以重写。它的参数跟_write相比（请看下方示例）少了一个encoding，而chunk则换成chunks（数组），每个元素是一个形式为{ chunk: ..., encoding: ...}的对象，默认是调用_write来完成工作，可以重写它来实现更高效的批量写入。

具体细节不再讲解，以写一个磁盘文件为例实现一个FileWriteStream：

```javascript
var fs = require("fs");
var util = require("util");
var Writable = require("stream").Writable;

function FileWriteStream(path, options) {
    // 必须调用基类的构造函数，将配置信息传进去
    Writable.call(this, options);

    this._path = path; // 文件路径
    this._fd = null; // 文件描述符
}

util.inherits(FileWriteStream, Writable);

/**
 * _write的实现，这里应该使用异步模式写入数据
 * @param chunk 数据块，可能是一个Buffer也可能是一个String
 * @param encoding 当chunk是一个String时，这里注明了它的编码
 * @param callback 当写入操作完成时_write需要调用callback通知外部已经写完数据,如果写入发生异常也需要将Error对象传给callback
 * @private
 */
FileWriteStream.prototype._write = function(chunk, encoding, callback) {
    var stream = this;

    function write() {
        fs.write(stream._fd, chunk, 0, chunk.length, function(err, written, buffer) {
            if (err != null) {
                callback(err);
                return;
            }

            callback();
        });
    }

    if (this._fd == null) {
        fs.open(this._path, "a", function(err, fd) {
            if (err != null) {
                callback(err);
                return;
            }

            stream._fd = fd;

            stream.once("end", function() {
                fs.closeSync(stream._fd);
                stream._fd = null;
            });

            write();
        });
    }
    else {
        write();
    }
}

var stream = new FileWriteStream("./test.txt");
stream.write("hello world");
stream.end();
```

从这里可以看出Readable跟Writable与数据设备交互的时候略有不同，前者必须用同步读取，因为数据必须马上返回给外部，而Writable则可以异步写数据，因为数据设备的写入速度不可预知，而外部数据是写到缓冲区中的。

### Duplex

Duplex在手册中一直强调是Readable以及Writable的合体，非常经典的应用就是tcp socket。它的扩展方式是Readable以及Writable二者扩展方式的合并，不过在创建子类的时候参数略有不同，单独出现Duplex的原因是因为Javascript不支持多重继承（囧）。

* allowHalfOpen - 是否允许半打开状态，默认为真。Duplex流是双向流，因此跟外部的交互有读有写，如果设置为false，则当其中一个方式结束时自动关闭另一个，默认允许只关闭一端。
* readableObjectMode - 当被作为可读流使用时是否使用对象模式
* writableObjectMode - 当被作为可写流使用时是否使用对象模式
* _read - 实现read方法
* _write - 实现write方法
* _writev - 实现批量写操作，可以不实现。

这里不再对Duplex的实现做示例，因为仅仅是Readable和Writable实现的合并。

### Transform

Transform是对Duplex的扩展，它的作用更多是用在数据的处理环节上，工程师调用Transform的write方法写入数据，然后Transform将数据交给_transform做处理，处理完后再写入read的缓冲区允许外部读取，gulp在这方面应用得特别多。

* _transform - 对写入的数据做处理，并重新塞回Readable缓冲区以供外部读取
* _flush - 将Writable中残留的缓冲区数据交给_transform做处理，本方法可以不实现

以下实现用于去除字符串空格的Transform示例：

```javascript
var fs = require("fs");
var util = require("util");
var Transform = require("stream").Transform;

function StringTransform(options) {
    Transform.call(this, options);
}

util.inherits(StringTransform, Transform);

StringTransform.prototype._transform = function(chunk, encoding, callback) {
    var str = chunk.toString("utf-8").replace(/\s/g, "");
    this.push(str);
    callback();
}

var str = "这 可 是 一 段 文 字 ";
var stream = new StringTransform();
stream.setEncoding("utf-8");

stream.on("data", function(chunk) {
    console.log(chunk);
});

stream.write(str);
```


