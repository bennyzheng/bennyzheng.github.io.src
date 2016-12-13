---
title:  Node.js - url模块
date: 2016-11-20 00:00:00
categories: Node.js学习笔记 
tags:
  - Node.js
  - URL
  - Path
---
URL是每一位网虫都使用过的东西，我记得第一次上网吧玩的时候，网吧老板帮我输了一个地址，在那时候很出名的聊天室网站"碧海银沙"，那时候我觉得上网就是上这个网站聊天，这是我第一次接触到URL。  
<!-- more -->
# URL的结构

Node.js手册非常形象的使用了下方这张图片作为解读URL结构的示例，图中很清楚地为我们构造出一个完整的URL应该是长什么样的。

![Node.js的URL结构图](/images/2016/url.png)

Node.js的url模块能够让工程师将一个URL字符串解析成一个对象，它拥有以下属性：

* href - 保存了完整的url地址。要特别注意的是它会将协议以及域名中的大写字母转成小写字母，至于其它位置的字符则会保持原样。这是因为协议以及域名是大小写不分的，为了方便直接全部都转成小写。
* protocol - 小写字母组成的协议类型，比如http:。要特别注意:也是协议的一部份（一直觉得很囧）。
* slashes - 一个布尔值，一般是true，它注明url的协议后边是否拥有两个/字符，有些奇葩URL是不带//的。
* host - 小写字母组成的域名+端口，二者之间有个冒号隔开。如果直接使用IP访问，那么host值就是IP地址，否则就是域名。另外，如果URL上没有写端口那么这里的值跟hostname一致。举个例子：www.host.com:8080。
* auth - URL中附带的认证信息，这个很少使用，毕竟不是很安全。当我们访问一个要求认证的网页时，服务器将会返回http状态码401要求浏览器提供用户认证信息，输入用户名和密码之后它将会在Http请求报文头中以Base64的方式写在Authorization字段中。你也可以使用类似"http://user:pass/host.com/a.php"的方式直接将用户名和密码（密码可省略，照样弹出窗口让你输入）写在URL中。
* hostname - 域名，跟host不同的是它不带端口。
* port - 端口，如果没有该值会使用该协议默认端口，比如http(80)、https(443)、ftp(21)、sftp(22)。
* pathname - 请求路径名，它不包含search以及hash。
* search - 查询字符串，在URL表现为类似"?id=1"这种形式，要特别注意的是它包含问号(?)。
* path - 请求路径，它包含pathname以及search部份，比如"/a/b/c.php?id=1"。许多浏览器并不把URL上的hash部份带到服务器，hash部份一般用于标识本页的某个锚点，因此path这一块是会被附在请求头上送到服务器端。
* query - 查询参数，它与search不同之处在于它不带问号，比如"id=1&name=2"。要特别注意的是，使用url.parse解析一个url的时候可以通过设置参数来决定是否对search做解析，如果将parseQueryString参数设置为true，query将会是一个对象，它用键值对的方式保存了search里包含的数据，比如{ id: 1, name: 2 }
* hash - 锚点，在URL处于末尾，由#号开始。在平时的网页URL中，浏览器一般不会将其发送到服务器端，它更多的是用在客户端。本意是用于做锚点，可以跳到页面某个特定位置，实际上现在更多的是用来做单页面的路由功能。举例："#hash"。

以下是一个完整的URL对象：

```javascript
var urlObject = {
    href: "http://user:pass@host.com:8080/p/a/t/h?query=string#hash",
    protocol: "http:",
    slashes: true,
    auth: 'user:pass',
    host: 'host.com:8080',
    port: '8080',
    hostname: 'host.com',
    hash: '#hash',
    search: '?query=string',
    query: { query: 'string' }, // 或者 "query=string"
    pathname: '/p/a/t/h',
    path: '/p/a/t/h?query=string',
    href: 'http://user:pass@host.com:8080/p/a/t/h?query=string#hash'
}
```

# URL模块API

URL模块仅提供了3个函数，没有提供类或者其它东西：

* url.format - 将一个包含必要字段的urlObject格式化成一个合法的URL字符串。
* url.parse - 将一个合法的URL字符串转成urlObject，注意URL并不需要包含所有的字段，比如"./"也算是一个URL字符串。
* url.resolve - 拥有from以及to两个参数，自动计算出to相对于from的绝对地址。如果from参数省略，则是将to转换为绝对路径。

这三个函数除了url.resolve都比较容易理解，至于resolve，手册给出了相应的例子，也比较形象：

```javascript
url.resolve('/one/two/three', 'four')         // '/one/two/four'
url.resolve('http://example.com/', '/one')    // 'http://example.com/one'
url.resolve('http://example.com/one', '/two') // 'http://example.com/two'
```

# 小贴士

* 弄清绝对路径的根目录在哪  
  
  在构建一个页面时为了方便可能会使用绝对路径，绝对路径是以/开头的路径，对绝对路径不大了解的前端可能会就发现直接在服务器上查看HTML文件时正常，但直接双击打开html文件则发生路径错误。这时需要认清这绝对路径的根目录到底是哪里。  
  
  假设我们的站点放在/wwwroot/mysite，访问地址是http://mysite.com，现在我们需要访问/index.html，它引用了一个路径为/css/site.css的样式文件，它被存放在/wwwroot/mysite/css/site.css。  
  
  这时候访问http://mysite.com/index.html，样式能够正常显示，而直接打开/wwwroot/mysite/index.html，则会说找不到/css/site.css。    
  
  这是因为在使用域名访问index.html时，绝对路径是指http://mysite.com/，映射到磁盘上的地址是/wwwroot/mysite/，而直接打开index.html时，index.html的地址是/wwwroot/mysite/index.html，这时候的绝对路径根目录是/，这时候磁盘上的/css/site.css是不存在的，真正的地址应该是/wwwroot/mysite/css/mysite.css。

* 查询字符串的编码  
  
  请求一个URL时许多人弄不清哪里需要编码哪里不需要编码，甚至还重复编码，这里特地做一下说明。另外，服务器端接收到你的值是不需要解码的，因为服务器本身就会帮你解码了。

```javascript
// 在发起请求的时候，queryString每个字段以及它的值都应该被编码，比如"/file?键1=值1&键2=值2"应该这么写（也可以使用encodeURI，但url不能带search和hash部份）：  
var url = "/file?" + encodeURIComponent("键1") + "="
    + encodeURIComponent("值1") + "&"
    + encodeURIComponent("键2") + "="
    + encodeURIComponent("值2");
// 当然，如果你确认你的key和value都不会有非ASCII字符以及需要转义的字符，你也可以不编码（如果不知道是否需要编码，一定要编码，有利无害）。另外，如果值是一个url,该url也需要做编码：  
// value本身是一个url，请对它的key和value编码。由于key的值是"key"，我确认它绝对不需要编码（因为encodeURIComponent("key")值同样是key），所以我就省掉了。
var value = "http://www.site.com/?key=" + encodeURIComponent("这是中文值");
// url的值为value的值，value的值也需要编码。
var url = "http://mysite.com/?url=" + encodeURIComponent(value);
```
