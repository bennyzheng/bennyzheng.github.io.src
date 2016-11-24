---
title: Node.js - path模块
categories: Node.js学习笔记 
date: 2016-11-18 00:00:00
tags:
  - Node.js
  - Path
  - URL
---
Node.js提供了一组非常方便的API专门用来操作路径。不过当你翻开Node.js手册关于path模块的时候，开头那个大大的章节标题"**Windows vs. POSIX**"呈现在你的面前，不知道你是什么感觉？反正我是觉得蛋疼乳酸菊花紧，兼容问题到哪都逃不掉啊！  
 Win32以及类Unix的POSIX标准在路径上真心差距很大，这个只要玩过windows以及类Unix系统的人都知道，许多代码由于路径兼容问题直接无法跨平台使用。
<!-- more -->
# 不同平台的路径差异

路径格式在大多数平台上都是POSIX标准格式，文件系统的根目录为/，然后各种设备（包括多个磁盘也一样）以它为根目录挂载为某个路径，因此任何目录在整个文件系统中的绝对路径都是一致的。

Windows与POSIX不同，它将磁盘分为多个区，它们并没有一个统一的根目录，比如C盘的根目录是C:\，D盘的根目录是D:\，如果你在D盘使用了/来代表根目录，那么它指的就是D:\。

Windows与POSIX在文件、目录名的命名上也略有不同，早期FAT16系统采用的是8.3命名方式，文件名最多8个字符，再加一个.以及最多3个字符的扩展名，比如”filename.ext”，后来win95开始才开始有了长文件名，但也是问题很多。不过我们现在基本上不需要关注短文件名以及长文件名，需要特别关注的是路径分隔符。在Windows的路径中分隔一个目录或文件的采用了\，而POSIX采用的是/，事实上Windows同样认识/分隔符（我宁愿它干脆不认识，净给人添麻烦）。

或许许多前端工程师没关注过什么叫POSIX，简单来说类Unix的系统（linux、BSD、OSX、Unix……）都是遵守POSIX规范的，这里引用一下百度百科对其做说明：

>POSIX表示可移植操作系统接口（Portable Operating System Interface ，缩写为 POSIX ），POSIX标准定义了操作系统应该为应用程序提供的接口标准，是IEEE为要在各种UNIX操作系统上运行的软件而定义的一系列API标准的总称，其正式称呼为IEEE 1003，而国际标准名称为ISO/IEC 9945。  
POSIX标准意在期望获得源代码级别的软件可移植性。换句话说，为一个POSIX兼容的操作系统编写的程序，应该可以在任何其它的POSIX操作系统（即使是来自另一个厂商）上编译执行。  
POSIX 并不局限于 UNIX。许多其它的操作系统，例如 DEC OpenVMS 支持 POSIX 标准，尤其是 IEEE Std. 1003.1-1990（1995 年修订）或 POSIX.1，POSIX.1 提供了源代码级别的 C 语言应用编程接口（API）给操作系统的服务程序，例如读写文件。POSIX.1 已经被国际标准化组织（International Standards Organization，ISO）所接受，被命名为 ISO/IEC 9945-1:1990 标准。

编写一个路径在POSIX和Windows下可以分别这么写：

```javascript
var winUrl = "d:\\wwwroot\\site\\file.ext";
var posixUrl = "/wwwroot/site/file.ext";
```

posixUrl在windows下同样认识，但它的位置取决于file.ext文件在哪个分区，如果它在C盘，那么它就相当于”c:\wwwroot\site\file.ext”，如果在其它盘就得换盘符。

为了处理这类不同路径规范，path模块提供了两种实现：

* path.win32 - 强制使用windows的路径规范处理路径
* path.posix - 强制使用posix的路径规范处理路径，根据我的实验，貌似这就是默认的处理方式。

# path模块的方法

Node.js的path模块针对上述路径差异也提供了相应的API或属性，个人认为，尽可能不要使用windows的路径写法。

## path.basename(path[, ext])

basename函数可以把路径最后一部份的名字返回，同时你如果使用了扩展名参数，它将自动将扩展名去掉，仅返回文件名（如果最后一部份是文件名的话）。

这里需要注意的是路径最后一部份是否必须是文件，如果是目录怎么办？请看示例代码：

```javascript
var path = require("path");
var url = "/wwwroot/site/myfile.js";

// myfile.js
console.log(path.basename(url));

// myfile，要特别注意的是：扩展名包含这个"."，如果仅传"js"的话，将得到"myfile."！
console.log(path.basename(url, ".js"));

// myfile到底是目录还是文件？
url = "/wwwroot/site/myfile";

// myfile，可见在这种情况下，它确实不在乎
console.log(path.basename(url));

// 如果路径最后一部份还带了个/呢？
url = "/wwwroot/site/myfile/";

// 同样是myfile！这点要特别注意！
console.log(path.basename(url));
```

之前提到过windows以及POSIX的区别，那么如果路径是一个windows路径，它又是如何工作的？

```javascript
var url = "E:\\wwwroot\\mysite\\myfile.js";

// 将得到 E:\wwwroot\mysite\myfile.js
console.log(path.basename(url));
```

从运行结果可见，API默认是使用了POSIX，这也难怪，微软本来就是喜欢出一些东西自己玩，不受人待见很正常，但如果你确认你的路径是windows的路径格式，可以使用path.win32提供的同名API：

```javascript
var url = "E:\\wwwroot\\mysite\\myfile.js";

// 将得到 myfile.js
console.log(path.win32.basename(url));
```

## path.dirname(path)

获取指定路径的目录名，它默认将basename当成文件名处理，也就是获取除了路径最后一部份之前的那一段路径。

与basename有个同样的疑问，如果路径最后一个字符是/，也就是路径最后一部份其实也是目录名，那又该怎么样?继续试验：

```javascript
var path = require("path");
var url = "/wwwroot/site/myfile.js";

// /wwwroot/site 没什么奇怪的，唯一要注意的是最后不带/
console.log(path.dirname(url));

// 看看没扩展名的情况
url = "/wwwroot/site/myfile";

// /wwwroot/site 它不在乎你有没有扩展名
console.log(path.dirname(url));

// 看看site本身也是目录的情况
url = "/wwwroot/site/";

// /wwwroot 看来它不在乎最后是不是文件，如果最后一段是目录名，那么它就获取该目录所在的目录路径。
console.log(path.dirname(url));
```

## path.extname(path)

获取路径的扩展名。

关于扩展名，其实在类Unix系统中，扩展名并不重要，比如你给一个文件加上可执行权限，那么它就会被认为是一个应用程序。如果你用GUI界面操作系统，扩展名的主要工作用于确定mime-type（当然可以造假），然后决定用什么应用程序来打开这个文件。

扩展名是文件名后半部份，它的内容包含"."，比如myfile.txt的扩展名是".txt"。

官方给的例子挺详细，直接上：

```javascript
path.extname('index.html')
// returns '.html'

path.extname('index.coffee.md')
// returns '.md'

path.extname('index.')
// returns '.'

path.extname('index')
// returns ''

path.extname('.index')
// returns '' 这个要特别注意一下！扩展名可以没有，但文件名不能省。
```

很幸福的，这个函数在windows系统下表现与POSIX一致。

## path.format(pathObject)

格式化一个pathObject，将其转成一个合法的字符串。pathObject可以拥有以下属性：

* dir - 目录路径，这个路径最后不要带有/，否则会出现使用//来分隔目录的情况
* root - 根目录
* base - 文件名+扩展名
* name - 文件名
* ext - 扩展名

从手册描述上来看，格式化函数其实是个很弱智的函数，它仅仅是按一个简单的规则将这个pathObject定义的各个属性拼装起来成为一个字符串罢了，基本上遵守以下规则：

* path分为两个部份，一个是路径，对应dir以及root。一个是文件名，对应base、name以及ext。
* dir/root是互斥的，如果dir存在，则不会管root，路径部份以dir为准。
* base与name+ext是互斥的，如果base存在，则无视name以及ext，以base为准，也就是说base等于name+ext。

为了更好的了解这个规则，继续上代码：

```javascript
var path = require("path");

// 文件名为file.txt，dir和root同时存在
var pathObject = {
    dir: ".",
    root: "/",
    base: "file.txt"
}

// ./file.txt root被无视，结果等于dir + "/" + base
console.log(path.format(pathObject));

// 文件名为file.txt，只存在root
pathObject = {
    root: "/",
    base: "file.txt"
}

// /file.txt 结果等于root + base
console.log(path.format(pathObject));

// 这里引出了一个疑问，根据上一次输出的结果，
// format不会给root和base之间添加/，那么如果root不是/，并且不以/结束会怎么样?
// 文件名为file.txt，只存在root，并且值为/wwwroot，它不以/结束
pathObject = {
    root: "/wwwroot",
    base: "file.txt"
}

// /wwwrootfile.txt 囧啊，结果真的等于root + base，我早说它不智能了
console.log(path.format(pathObject));

// 同样，dir注明它被添到结果时，会在后边跟一个/隔开文件名
// 文件名为file.txt，只存在dir，并且值以/结尾
pathObject = {
    dir: "/wwwroot/",
    base: "file.txt"
}

// /wwwroot//file.txt 只能微笑一下了
console.log(path.format(pathObject));

// 我们如果故意把base的值瞎搞，它不是一个文件名，而是一段路径，会怎么样？
// 文件名为/dir/file.txt，root为/wwwroot，我怀疑结果将是/wwwroot/dir/file.txt
pathObject = {
    root: "/wwwroot",
    base: "/dir/file.txt"
}

// /wwwroot/dir/file.txt 如我所料
console.log(path.format(pathObject));

// 如果dir存在相对路径呢？比如/wwwroot/dir/..相当于/wwwroot，那么是否会处理？
pathObject = {
    dir: "/wwwroot/dir/..",
    base: "file.txt"
}

// /wwwroot/dir/../file.txt 真的仅仅是拼接一下，其实这事我也能干啊
console.log(path.format(pathObject));

// base与name和ext同时存在的情况
pathObject = {
    dir: "/wwwroot",
    base: "file.txt",
    name: "myfile",
    ext: ".js"
}

// /wwwroot/file.txt 以base为准，name和ext被无视掉了，其它规则跟dir/root一样狗血。
console.log(path.format(pathObject));
```

虽然path.format是如此弱智，但还是有办法补救的，我们还有一个path.normalize可以用来对它做处理。

## path.isAbsolute(path)

判断路径是否为绝对路径，这里同样存在兼容问题，由于API默认以POSIX标准为准，因此如果路径是一个windows规范路径，那么需要使用path.win32.isAbsolute来做判断，所以千万不要随便使用windows规范的路径。

```javascript
var path = require("path");

// POSIX
var url = "/wwwroot/dir";
// true
console.log(path.isAbsolute(url));

// WIN
url = "E:\\wwwroot";
// false
console.log(path.isAbsolute(url));
// true
console.log(path.win32.isAbsolute(url));
```

## path.join([...paths])

将多个路径连接在一起，它相对于path.format来说非常智能，你不用担心它生成的路径会出现"//"，也不用担心各种弱智的表现，唯一要考虑的是你输入的路径应该是个字符串。

没什么特别注意的，只需要知道它很智能，能够识别.以及..之类的相对路径，并且把它们拼装成一个很好的路径即可。这是来自官方手册的例子：

```javascript
path.join('/foo', 'bar', 'baz/asdf', 'quux', '..')
// returns '/foo/bar/baz/asdf'

path.join('foo', {}, 'bar')
// throws TypeError: Arguments to path.join must be strings
```

## path.normalize(path)

在体验过path.format之后，我们需要一个函数对它生成的可能乱七八糟的路径做整理，使其变成一个很规范没有冗余字符的路径，path.normalize就是一个很好的选择。它能够识别.以及..，同时也能够帮你去除多余的/，有点不足的是它不识别windows路径分隔符\。

如果你使用的是win32的normalize，它将会把所有的/转化成\。建议在做处理之前先手工做一下替换，将\统一转成/。

```javascript
var path = require("path");

// 乱七八糟的路径
var url = "/wwwroot\\dir0/dir1//.././myfile.txt";

// /wwwroot\dir0/myfile.txt
console.log(path.normalize(url));

// \wwwroot\dir0\myfile.txt
console.log(path.win32.normalize(url));

// /wwwroot/dir0/myfile.txt 正解！
console.log(path.normalize(url.replace(/\\/g, "/")));
```
## path.parse(path)

将一个合法路径解析成一个pathObject，格式参考format函数。
这个方法同样不怎么智能，建议使用它的时候先做一下处理（请参考path.normalize示例代码）。

下面来看看它工作表现：

```javascript
var path = require("path");

// 乱七八糟的路径
var url = "/wwwroot\\dir0/dir1//.././myfile.txt";

// {
//     root: '/',
//     dir: '/wwwroot\\dir0/dir1//../.',
//     base: 'myfile.txt',
//     ext: '.txt',
//     name: 'myfile'
// }
console.log(path.parse(url));

// {
//     root: '/',
//     dir: '/wwwroot\\dir0/dir1//../.',
//     base: 'myfile.txt',
//     ext: '.txt',
//     name: 'myfile'
// }
console.log(path.win32.parse(url));

// {
//     root: '/',
//     dir: '/wwwroot/dir0/dir1//../.',
//     base: 'myfile.txt',
//     ext: '.txt',
//     name: 'myfile'
// }
console.log(path.parse(url.replace(/\\/g, "/")));

// 正解！
// {
//     root: '/',
//     dir: '/wwwroot/dir0',
//     base: 'myfile.txt',
//     ext: '.txt',
//     name: 'myfile'
// }
console.log(path.parse(path.normalize(url.replace(/\\/g, "/"))));
```

## path.relative(from, to)

获取to参数相对于from参数的相对路径。

这个函数没什么特别好注意的，只需要注意它同样有windows的兼容性问题即可。

以下是官方示例：

```javascript
path.relative('/data/orandea/test/aaa', '/data/orandea/impl/bbb')
// returns '../../impl/bbb'
```

## path.resolve([...paths])

这个函数与path.join非常相似，但它将会得到一个绝对路径，并且该路径是基于磁盘文件系统的，得到的结果与当前工作目录有关。

假设我当前工作目录在/wwwroot/github/bennyzheng.github.io/_posts/2016/（是真的……我的测试脚本就放在这个目录），那么以下代码将会得到相应的结果：

```javascript
var path = require("path");

// /wwwroot/github/bennyzheng.github.io/_posts/2016/myfile
console.log(path.resolve("./", "myfile"));

// 注意第二个参数是以/开头，前边的./dir直接被无视掉了，因为它这里已经直接确定好根目录了
// /myfile
console.log(path.resolve("./dir", "/myfile"));
```

# path模块的属性

## path.delimiter

保存了当前系统下多个路径的分隔符，它的值在windows系统下是;，在POSIX规范的系统下是:。
如果不理解，请查看当前系统的系统变量PATH的值，它就是多个路径连接在一起的。

上一下官方手册的示例代码，它演示了如何利用该属性将系统变量PATH分隔成多个路径，示例中的PATH变量明显是POSIX风格：

```javascript
console.log(process.env.PATH)
// '/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin'

process.env.PATH.split(path.delimiter)
// returns ['/usr/bin', '/bin', '/usr/sbin', '/sbin', '/usr/local/bin']
```

## path.sep

保存了当前系统下使用的目录分隔符，它的值在windows下是\，在POSIX规范的系统下是/。

## path.posix

提供了一套针对遵守posix规范的系统的API支持，从目前的试验来看，我们完全可以使用path.xxx来调用API，因为path.posix就是默认的方式。

## path.win32

提供了一套针对windows系统的API支持，考虑到可移植性，强烈建议不要在你的程序中使用。
