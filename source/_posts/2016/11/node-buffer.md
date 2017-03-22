---
title: Node.js - buffer模块
categories: Node.js学习笔记
date: 2016-11-21 00:00:00
tags:
  - Buffer
  - Binary
  - Node.js
---
刚把Javascript本身新引入的TypeArray以及ArrayBuffer好了解了一下，Node.js针对二进制数据提供了buffer模块，手册中提到Buffer是Int8Array的另一种更适合Node.js使用的实现方式（我看是因为之前TypeArray还没实现结果先出了Buffer，现在准备一条道走到黑吧……不过确实用起来挺好用）。在Node.js的数据交换过程中，buffer模块经常会被使用到，比如数据流。
<!-- more -->
# Buffer的初始化

使用buffer模块不需要使用require引入模块，全局提供了一个Buffer对象（可见它使用频率有多频繁）。虽然Buffer可以使用new关键词创建一个实例，但手册上明确表示它已经被弃用，我们应该使用alloc之类的API创建。

需要特别注意的是：

* Buffer是固定大小的（fixed-size），从实例化后就无法再改变它的内存大小。
* Buffer是动态申请的内存，它并不在V8引擎的堆里。

## Buffer.form(array)  
从数组中创建Buffer，需要注意的是这里的数组是指普通的Array、TypeArray以及类数组的的对象。需要注意的是从数组或类数组对象创建的Buffer是新申请内存并且把数组内容复制到Buffer中。

```javascript
// 普通数组
var buffer = Buffer.from([1, 2, 3]);
// <Buffer 01 02 03>
console.log(buffer);

// TypeArray
buffer = Buffer.from(new Int8Array([2, 3, 4]));
// <Buffer 02 03 04>
console.log(buffer);

// 试一下TypeArray
var arrayBuffer = new ArrayBuffer(10);
var int8Array = new Int8Array(arrayBuffer);
// 从TypeArray创建
buffer = Buffer.from(int8Array);
int8Array[0] = 12;

// <Buffer 00 00 00 00 00 00 00 00 00 00>
// 第一个字节没有变，证明这里是复制了一个副本
console.log(buffer);

// 如果想让它们共享内存，可以从TypeArray的缓冲区创建：
buffer = Buffer.from(int8Array.buffer);
// <Buffer 0c 00 00 00 00 00 00 00 00 00>
console.log(buffer);
```

## Buffer.from(arrayBuffer[, byteOffset [, length]])  
从ArrayBuffer中创建Buffer，同时可以指定开始的位置byteOffset以及字节数length。需要注意的是从ArrayBuffer创建的Buffer是与ArrayBuffer共享内存的，随便改哪个都会导致另一个更新。

```javascript
var arrayBuffer = new ArrayBuffer(10);
var int8Array = new Int8Array(arrayBuffer);
var buffer = Buffer.from(arrayBuffer);

// <Buffer 00 00 00 00 00 00 00 00 00 00> 初始值
console.log(buffer);
int8Array[0] = 10;

// <Buffer 0a 00 00 00 00 00 00 00 00 00>
// 第一个字节变成了10，证明从arrayBuffer创建的是共享内存的
console.log(buffer);
```
## Buffer.alloc(size[, fill[, encoding]])  
直接申请一段内存，内存大小为size个字节。新申请的内存每个字节默认将值填为0，也可以通过fill设置填充选项。注意这里相当于调用了buffer的fill方法，填充编码默认为utf-8。

```javascript
// <Buffer 61 61 61 61 61 61 61 61 61 61>
// 使用'a'填充每个字节
var buffer = Buffer.alloc(10, 'a');
console.log(buffer);

// 使用'ab'填充整个buffer，由于buffer长度为10，ab会被不断循环填充。
// 同时指明编码格式为utf-8
buffer = Buffer.alloc(10, 'ab', "utf-8");
console.log(buffer);
```
## Buffer.allocUnsafe(size)  
直接申请一段内存，内存大小为size个字节。与alloc不同的是，它标明了是"unsafe"，其实就是申请内存时不使用0将原有内存内容擦除因此不安全，但也因此它相对于alloc来说更快。

```javascript
var buffer = Buffer.allocUnsafe(10);
// 每次buffer的内容都是不可预知的
console.log(buffer);
```

## Buffer.allocUnsafeSlow(size)  
申请一段内存，内存大小为size个字节。这个函数使用了Slow这个词，它的意思是说这块内存将是独享的，不允许Buffer按照规则决定是否从预申请的内存池中分配内存（因此它是缓慢的，大概就是这个意思）。事实上，所有不符合内存池分配规则的Buffer都是独享的，只不过这里特地要求不考虑与其它Buffer共享一块内存池罢了，没有什么大的区别。**注意，这个方法平时不需要考虑使用，它更多的是提供给Node.js源码开发者使用的，普通开发者应该使用Buffer**。

# 数据的读写

Buffer针对各种数据类型提供了多种API用于读写操作，如果数据类型不是int8还需要区分字节序是高位优先还是低位优先，以BE(BIG_ENDIAN)和LE(LITTLE_ENDIAN)结尾区分。

Buffer的读写操作跟Javascript的TypeArray提供的读写操作非常类似，唯一需要注意的是noAssert参数，请谨慎使用该参数。noAssert如果被设置为true，意味着在写数据的时候不会检测写入的数据是否会超出当前Buffer分配给数据存储的内存区域！这可能会导致灾难性地问题。

Buffer允许把它当数组一般使用下标读写某个字节，但同时也提供了大量的方法方便读写：

## 读操作

* readInt8(offset[, noAssert]) - 读取一个8位的整型
* readUInt8(offset[, noAssert]) - 读取一个8位的无符号整型
* readInt16BE(offset[, noAssert]) - 高位优先读取16位整型
* readInt16LE(offset[, noAssert]) - 低位优先读取16位整型
* readUInt16BE(offset[, noAssert]) - 高位优先读取16位无符号整型
* readUInt16LE(offset[, noAssert]) - 低位优先读取16位无符号整型
* readInt32BE(offset[, noAssert]) - 高位优先读取32位整型
* readInt32LE(offset[, noAssert]) - 低位优先读取32位整型
* readUInt32BE(offset[, noAssert]) - 高位优先读取32位无符号整型
* readUInt32LE(offset[, noAssert]) - 低位优先读取32位无符号整型
* readFloatBE(offset[, noAssert]) - 高位优先读取32位浮点数
* readFloatLE(offset[, noAssert]) - 低位优先读取32位浮点数
* readDoubleBE(offset[, noAssert]) - 高位优先读取64位浮点数
* readDoubleLE(offset[, noAssert]) - 低位优先读取64位浮点数
* readIntBE(offset, byteLength[, noAssert]) - 高位优先读取一个指定字节数的整型，最大48位
* readIntLE(offset, byteLength[, noAssert]) - 低位优先读取一个指定字节数的整型，最大48位
* readUIntBE(offset, byteLength[, noAssert]) - 高位优先读取一个指定字节数的无符号整型，最大48位
* readUIntLE(offset, byteLength[, noAssert]) - 低位优先读取一个指定字节数的无符号 整型，最大48位

## 写操作

* writeInt8(value, offset[, noAssert]) - 写入一个8位整型
* writeUInt8(value, offset[, noAssert]) - 写入一个8位无符号整型
* writeInt16BE(value, offset[, noAssert]) - 高位优先写入一个16位整型
* writeInt16LE(value, offset[, noAssert]) - 低位优先写入一个16位整型
* writeUInt16BE(value, offset[, noAssert]) - 高位优先写入一个16位无符号整型
* writeUInt16LE(value, offset[, noAssert]) - 低位优先写入一个16位无符号整型
* writeInt32BE(value, offset[, noAssert]) - 高位优先写入一个32位整型
* writeInt32LE(value, offset[, noAssert]) - 低位优先写入一个32位整型
* writeUInt32BE(value, offset[, noAssert]) - 高位优先写入一个32位无符号整型
* writeUInt32LE(value, offset[, noAssert]) - 低位优先写入一个32位无符号整型
* writeFloatBE(value, offset[, noAssert]) - 高位优先写入一个32位浮点数
* writeFloatLE(value, offset[, noAssert]) - 低位优先写入一个32位浮点数
* writeDoubleBE(value, offset[, noAssert]) - 高位优先写入一个64位浮点数
* writeDoubleLE(value, offset[, noAssert]) - 低位优先写入一个64位浮点数
* write(string[, offset[, length]][, encoding]) - 将一个字符串写入指定的位置，可以通过length限制长度以及通过encoding设置编码，编码默认是UTF-8。
* writeIntBE(value, offset, byteLength[, noAssert]) - 高位优先将value值写入指定的位置，需要明确指定value值占的位数
* writeIntLE(value, offset, byteLength[, noAssert]) - 低位优先将value值写入指定的位置，需要明确指定value值占的位数

# 序列化

Buffer提供了几种方法可以将内容序列化：

* toString([encoding[, start[, end]]])  
以指定的编码返回指定范围的字符串，默认是使用UTF-8，返回全部内容。

```javascript
var buffer = Buffer.from("hello world");
// world 注意，d的下标是10，这里如果设置end，返回的字符串是不包含end下标的字符的。
console.log(buffer.toString("utf-8", 6, 11));
```

* toJSON()  
将Buffer以{"type":"Buffer","data":[1,2,3,4,5]}的方式返回一个JSON对象，个人感觉它的格式很死板。

```javascript
var buffer = Buffer.from("hello world");
// { type: 'Buffer',
//     data: [ 104, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100 ] }
console.log(buffer.toJSON());
```

# 字节序处理

涉及到多字节数据的读写的API都得考虑字节序的问题，在Buffer提供的API中，读写操作基本上都以BE以及LE后缀结尾。当你明确知道当前数据是以高位优先或者低位优先存储，却希望将其做转换的话可以使用以下三个API进行数据处理：

* swap16() - 将当前Buffer的数据每两个字节视为一个16数字，并反向重排字节。
* swap32() - 将当前Buffer的数据每两个字节视为一个32数字，并反向重排字节。
* swap64() - 将当前Buffer的数据每两个字节视为一个64数字，并反向重排字节。

以16位数字（int16/uint16)为例：

```javascript
// 8个字节，可以存放4个int16
var buffer = Buffer.allocUnsafe(8);

buffer.writeInt16BE(0x0102, 0);
buffer.writeInt16BE(0x0304, 2);
buffer.writeInt16BE(0x0506, 4);
buffer.writeInt16BE(0x0708, 6);

// <Buffer 01 02 03 04 05 06 07 08>
// 高位优先，符合人类阅读习惯
console.log(buffer);

// 我认为我现在需要低位优先方式存储，调用swap16进行整理
buffer.swap16();

// <Buffer 02 01 04 03 06 05 08 07>
// 它以2个字节为单位将字节顺序调换
console.log(buffer);

// 重置一下
buffer.writeInt16BE(0x0102, 0);
buffer.writeInt16BE(0x0304, 2);
buffer.writeInt16BE(0x0506, 4);
buffer.writeInt16BE(0x0708, 6);

// 看看swap32会有啥结果
buffer.swap32();

// <Buffer 04 03 02 01 08 07 06 05>
// 它认为4个字节是一个整体，以4个字节为单位将字节顺序调换
console.log(buffer);
```

# 缓冲池与性能

Buffer对象有一个属性叫poolSize，用于设置缓冲池的大小。以下是这个缓冲池的关键性代码，来自Node.js(v6.9.1/lib/buffer.js)源码：

```javascript
Buffer.poolSize = 8 * 1024;

// ....

function createPool() {
  poolSize = Buffer.poolSize;
  allocPool = createUnsafeBuffer(poolSize);
  poolOffset = 0;
}

createPool();

// ....

function allocate(size) {
  if (size <= 0) {
    return new FastBuffer();
  }
  if (size < (Buffer.poolSize >>> 1)) {
    if (size > (poolSize - poolOffset))
      createPool();
    var b = allocPool.slice(poolOffset, poolOffset + size);
    poolOffset += size;
    alignPool();
    return b;
  } else {
    // Even though this is checked above, the conditional is a safety net and
    // sanity check to prevent any subsequent typed array allocation from not
    // being zero filled.
    return createUnsafeBuffer(size);
  }
}

```

createPool函数从内存中申请了8kb的内存，并将其作为当前Buffer的缓冲池。

allocate函数用于分配内存，以下是我对它的注解：

```javascript
function allocate(size) {
  // 当外部申请的内存小于等于0，则直接返回一个FastBuffer
  // FastBuffer是使用Uint8Array实现的Buffer，这个不在我们关注的话题内
  if (size <= 0) {
    return new FastBuffer();
  }
  // 当申请的内存大小小于缓冲池大小的一半时就开始尝试从已经存在的缓冲区中分配内存
  if (size < (Buffer.poolSize >>> 1)) {
    // 如果申请的内存比当前空闲的缓冲池内存（有一部份被用掉给别的Buffer了）还大时
    // 直接放弃当前缓冲池重建一个新的！
    if (size > (poolSize - poolOffset))
      createPool();
    var b = allocPool.slice(poolOffset, poolOffset + size);
    poolOffset += size;
    alignPool();
    return b;
  } else {
    // Even though this is checked above, the conditional is a safety net and
    // sanity check to prevent any subsequent typed array allocation from not
    // being zero filled.
    // 直接申请新的内存返回
    return createUnsafeBuffer(size);
  }
}
```

从上边的源码解析中可以看到几个点：

* 为了节省申请内存消耗，Buffer直接申请了一块8kb的内存空间当成缓冲池随时用于存放小量数据
* 如果新建Buffer缓冲区大小小于当前设置的poolSize的一半时（4kb），则会使用缓冲池里的空闲内存分配，省掉申请内存消耗
* 如果新建Buffer需要的缓冲区大小满足缓冲池分配内存的原则，但当前缓冲池的空余内存又不够用，当前缓冲池会被直接浪费掉，直接申请新的8kb缓冲池。

这其实就是用空间换时间的做法，Node.js没有在缓冲池的内存管理上花什么心思，就是不够了就浪费掉申请新的，但很多时候数据量都不会超过4kb，也就是我们大部份操作都不会导致Buffer动态申请内存，提高了性能。

这里特别要注意的是，由于8kb的缓冲池是一次性申请的，只有缓冲池上所有Buffer都设置为null，否则整块缓冲池都不会被释放，也就是内存泄露。
