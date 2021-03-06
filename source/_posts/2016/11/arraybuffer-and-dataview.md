---
title: 缓冲数组以及数据视图
categories: javascript 
date: 2016-11-20 00:00:00
tags:
  - TypeArray
  - Node.js
  - ArrayBuffer
  - Binary
---

Javascript在数据的处理上一直不是强项，比如数字不分整型和浮点数统一使用了64位浮点数，如果涉及到二进制运算则显得非常无力，在数据传输上也非常浪费带宽。在ES6针对Javascript二进制数据处理上的无力引入了原始缓冲区ArrayBuffer，并且还提供了多种位数的int类型数组以及数据视图来处理数据。

<!-- more -->

## ArrayBuffer

ArrayBuffer是ES6新引入的用于处理二进制数组的缓冲区对象，它不提供对数据的操作能力，提供的是二进制数据的存储。

ArrayBuffer跟Array比较相似，它是一个定长的数组，在初始化后就无法改变长度。每个元素都是一个字节（8bit），相当于C语言中的char，所以如果它的长度是10，就是10字节，绝无例外。

在这里要特别提一下，我们平时编程所需的所有的数据都可以用不同数量的byte表示，比如32位系统中的int类型长度是4，其实它就是拿4个连续的字节拼凑而成的一段空间。

ArrayBuffer就是以字节为单位的缓冲区，再配上对应的API（TypeArray or DataView）来做数据的操作。

ArrayBuffer需要使用new关键词进行实例化，参数为缓冲区的大小：

```javascript
var buffer = ArrayBuffer(1024); // 1024个字节，即1kb
```

它提供了一个属性以及一个方法： 

* byteLength - 只读属性，获取本缓冲数组的长度，由于一个元素就是1字节，其实这里也可以当成该缓冲区申请了多少字节的内存空间。
* slice(begin[, end]) - 返回一个新的ArrayBuffer，并使用当前ArrayBuffer从begin到end之间的数据填充，如果end省略则是从begin到缓冲区结束。

同时它还提供了一个静态方法：

* ArrayBuffer.isView(arg) - 如果参数是TypeArray或者DataView，则返回true，否则返回false。

## 数据视图

之前提到过ArrayBuffer仅仅是提供了对数据的存储，相当于动态申请了一段内存（当然你不需要像C/C++一般去释放它，或者在不需要的时候也可以将其设置为null由gc自动释放），对于这段缓冲区的操作则是由一组TypeArray以及灵活度更高的DataView对象来提供。

### TypeArray

类型数组提供了几种强类型数组对ArrayBuffer中的数据进行读写，其中包括：

* Int8Array - 8位的整型数组，由于ArrayBuffer每个元素就是1字节，也就是8位，所以Int8Array的长度就是ArrayBuffer的长度。
* Uint8Array - 8位的无符号整型数组，int8使用了7位来保存数据，1位来当符号位以标识数据的正负，而无符号整型则将8位全部用来保存数据，只能保存非负整数，同样它的长度是ArrayBuffer的长度。
* Uint8ClampedArray - 8位的无符号整型数组，它与Uint8Array在处理超出范围的数据上的做法略有不同。
* Int16Array - 16位的整型数组，使用2个字节凑成一个int16，也就是ArrayBuffer(1024)可以被当成一个长度为512的int16数组使用。
* Uint16Array - 16位的无符号整型数组，与Uint8Array类似，仅是每个元素由两个字节组成。
* Int32Array - 32位的整型数组，使用4个字节凑成一个int32，也就是ArrayBuffer(1024)可以被当成一个长度为256的int32数组使用。
* Uint32Array - 32位的无符号整型数组
* Float32Array - 32位的IEEE浮点数数组（单精度float）
* Float64Array - 64位的IEEE浮点数数组（双精度float)

吐槽一下：这些类型数组以及ES6新引入的类、TypeScript什么的都感觉像是在打以前吹揍着"脚本语言就应该简单，弱类型是多么的方便"的大神们的脸。

曾几何时，我还在流着冷汗看着试卷想着int在32位系统上应该是占了几位，long又是占了几位，没想到当了前端还是要面对它们，感谢ECMA。

类型数组提供的初始化方式、API都是一模一样的，除了元素类型。

#### 初始化

初始化一个TypeArray有多种方法：

* new TypeArray(length)  
指明元素个数来创建一个类型数组：

```javascript
// 直接创建一个1024个元素的
var length = 1024;

// 占了1024 * 1个字节，即1024个字节
var int8Array = new Int8Array(length);
console.log(int8Array.byteLength); // 1024

// 占了1024 * 2个字节，即2048个字节
var int16Array = new Int16Array(length);
console.log(int16Array.byteLength); // 2048
```

* new TypeArray(typedArray)  
从一个已经存在的类型数组创建新的类型数组，要特别注意的是它们仅仅是复制关系。

```javascript
var int8Array = new Int8Array(1024);
console.log(int8Array.byteLength); // 1024

var int16Array = new Int16Array(int8Array);
console.log(int16Array.byteLength); // 2048

var newInt8Array = new Int8Array(int16Array);
console.log(newInt8Array.byteLength); // 1024
```

这里需要考虑到一个情况，一个范围较大的类型可以安全地存放范围较小的类型值，可是反过来的话就不行了，比如int8的范围是-128到127，int16是-32768到+32767。下边示例则证明了如果发生了数据溢出，则会将数据的一部份截断，比如int16是2个字节，而int8是1个字节，那么将会保留一个字节，然后抛弃另一个字节，导致数据变化。

```javascript
var int16Array = new Int16Array(2);

// 超出int8的数值范围了，int8的数字范围是-128到127
int16Array[0] = 1290;
int16Array[1] = 1024;

var int8Array = new Int8Array(int16Array);

// 在我的机子的运行结果是 "1290 1024 10 0"
// 这里为什么会得到10和0，后边会解释
console.log(int16Array[0], int16Array[1], int8Array[0], int8Array[1]);
```

* new TypeArray(object)  
从一个对象中创建类型数组，对象类型我只试出Array可以，应该还有其它的。

```javascript
var int16Array = new Int16Array([1, 2]);
// Int16Array [ 1, 2 ]
console.log(int16Array);
```

* new TypeArray(buffer [, byteOffset [, length]])  
从一个已经存在的ArrayBuffer对象中创建类型数组，同时可以指定开始的位置以及长度来针对ArrayBuffer的某个区域进行操作。

要特别注意的是，byteOffset是指从第几个字节算起，而length参数是指TypeArray拥有多少个元素。

```javascript
var buffer = new ArrayBuffer(1024);
var int8Array = new Int8Array(buffer, 2, 2);
var int16Array = new Int16Array(buffer, 2, 2);

// 2 2
console.log(int8Array.length, int8Array.byteLength);
// 2 4 由于是int16，所以2个int16占了4个字节
console.log(int16Array.length, int16Array.byteLength);
```

#### 静态成员

* TypeArray.BYTES_PER_ELEMENT - 返回该种TypeArray（注意，不要直接写TypeArray，应该是Int8Array之类的对象）每个元素占的字节数。
* TypeArray.from(source[, mapFn[, thisArg]]) - 从一个可遍历的对象(Set、Array、String之类)中创建一个类型数组。  
*这里要特别注意当你需要传第三个参数用来给mapFn当this的时候，请不要把mapFn写成箭头函数。*

```javascript
var array = [1, 2, 3];
var obj = { a : 2 };

// Int8Array [ 1, 2, 3 ]
console.log(Int8Array.from(array));

// Int8Array [ 2, 4, 6 ]
console.log(Int8Array.from(array, (x) => (x + x)));

// {}
// {}
// {}
// Int8Array [ 2, 4, 6 ]
// 请不要使用箭头函数！因为箭头函数的this是不可改变的！
console.log(
    Int8Array.from(array, x => {
        console.log(this);
        return x * 2;
    }, obj)
);

// { a: 2 }
// { a: 2 }
// { a: 2 }
// Int8Array [ 3, 4, 5 ]
// 不使用箭头函数后，obj生效了！
console.log(
    Int8Array.from(array, function(x) {
        console.log(this);
        return x + this.a;
    }, obj)
);
```
* TypeArray.of(element0[, element1[, ...[, elementN]]]) - 将所有参数拼装成一个类型数组。要特别注意的是如果元素不合法，则会得到0.

```javascript
// Int8Array [ 1, 2, 3, 0 ]
console.log(
    Int8Array.of(1, 2, 3, [4, 5])
);
```

#### 成员变量

* buffer - TypeArray内部使用的ArrayBuffer对象，如果将一个外部ArrayBuffer当参数构造了一个TypeArray，那么该buffer就是那个外部ArrayBuffer。如果为同一个ArrayBuffer建立两个TypeArray，当使用了其中一个TypeArray的API对数据做修改，另一个也会生效，因为它们使用了同一个数据源。

```javascript
var buffer = new ArrayBuffer(2);
var int8Array = new Int8Array(buffer);
var int16Array = new Int16Array(buffer);

// Int16Array [ 0 ] Int8Array [ 0, 0 ]
console.log(int16Array, int8Array);
int16Array[0] = 123456;
// Int16Array [ -7616 ] Int8Array [ 64, -30 ]
// 二者都变了，因为ArrayBuffer变了
console.log(int16Array, int8Array);
```

* byteLength  - 获取TypeArray使用的ArrayBuffer的字节长度，与length不同的是它的值是以字节计算的，所以不会因为TypeArray的类型不同而发生变化。

```javascript
var buffer = new ArrayBuffer(1024);
var int8Array = new Int8Array(buffer);
var int16Array = new Int16Array(buffer);

// 1024 1024
console.log(int8Array.byteLength, int16Array.byteLength);
```

* byteOffset - 保存了当前TypeArray操作的缓冲区域的起始位置，它是按字节算的

```javascript
var buffer = new ArrayBuffer(1024);
var int8Array = new Int8Array(buffer, 1);
var int16Array = new Int16Array(buffer);

// 1 0 
console.log(int8Array.byteOffset, int16Array.byteOffset);
```

* length - 保存了当前TypeArray的元素个数，该属性跟Array一致，在这里特地列出来是为了跟byteLength做对比。

```javascript
var buffer = new ArrayBuffer(10);
var int8Array = new Int8Array(buffer);
var int16Array = new Int16Array(buffer);

// 10 5
console.log(int8Array.length, int16Array.length);
```

#### 成员方法

TypeArray实现了大部份Array的方法，所以你可以调用类似map、every之类的方法处理数据。这方面的方法与属性就不在这里数说了，仅列出Array没有的。

* set(array|typeArray [,offset])  
将一个数组或者类型数组复制到TypeArray指定的位置中，如果不指定位置则是从头覆盖。如果数据超出范围，则会将数据截断。

```javascript
var buffer = new ArrayBuffer(10);
var int8Array = new Int8Array(buffer);

// 注意，它必须是一个Array或者TypeArray！
int8Array.set([5], 2);
// Int8Array [ 0, 0, 5, 0, 0, 0, 0, 0, 0, 0 ]
console.log(int8Array);
// Int8Array [ 64, 0, 5, 0, 0, 0, 0, 0, 0, 0 ] 被截断了
int8Array.set([123456]);
console.log(int8Array);
```
* subarray([begin [,end]])  
返回指定开始以及结束位置之前的元素组成的新同类型TypeArray。  
这里要注意以下两点：
  + 返回的新TypeArray与当前TypeArray是共享缓冲区的，也就是修改了其中一个的值，也同样会影响另一个。
  + 返回的新TypeArray不包含end下标元素

```javascript
var int8Array = new Int8Array([1, 2, 3, 4, 5]);
var subarray = int8Array.subarray(1, 3);

// Int8Array [ 2, 3 ]
// 对应着int8Array[1]以及int8Array[2]，不包含int8Array[3]
console.log(subarray);

int8Array[2] = 20;
// Int8Array [ 2, 20 ] 注意，subarray[1]被更改了
console.log(subarray);
```

### DataView

DataView提供了相对TypeArray更为灵活的方式用于操作数据，从TypeArray的各种示例上看，TypeArray其实就对应着C语言的数组（定长、元素同类型），而DataView则可以操作类似结构体的东西，它允许你随时切换不同的类型读写数据。

#### 初始化

new DataView(buffer [, byteOffset [, byteLength]])

DataView只可以使用ArrayBuffer作为操作对象，同时允许指定操作区域，这个跟TypeArray是一致的。

```javascript
var buffer = new ArrayBuffer(10);
var view = new DataView(buffer);

// DataView {
//     byteLength: 10,
//     byteOffset: 0,
//     buffer: ArrayBuffer { byteLength: 10 }
// }
console.log(view);
```

#### 成员变量

与TypeArray一模一样，DataView拥有buffer、byteOffset以及byteLength三个成员变量，它们的作用也是一样的，所以这里不再详说。

#### 成员方法

DataView的成员方法提供了一系列的get、set方法，用法基本上一致： 

* get 在byteOffset指定的位置读取相应类型的值，littleEndian指定数据存储是高位优先还是低位优先
  + getInt8(byteOffset)
  + getUint8(byteOffset)
  + getInt16(byteOffset [, littleEndian])
  + getUint16(byteOffset [, littleEndian])
  + getInt32(byteOffset [, littleEndian])
  + getUint32(byteOffset [, littleEndian])
  + getFloat32(byteOffset [, littleEndian])
  + getFloat64(byteOffset [, littleEndian])
* set 在byteOffset指定的位置写入相应类型的值，littleEndian指定数据存储是高位优先还是低位优先
  + setInt8(byteOffset, value)
  + setUint8(byteOffset, value)
  + setInt16(byteOffset, value [, littleEndian])
  + setUint16(byteOffset, value [, littleEndian])
  + setInt32(byteOffset, value [, littleEndian])
  + setUint32(byteOffset, value [, littleEndian])
  + setFloat32(byteOffset, value [, littleEndian])
  + setFloat64(byteOffset, value [, littleEndian])

```javascript
var buffer = new ArrayBuffer(2);
var view = new DataView(buffer);
view.setInt8(0, 10);
// 10
console.log(view.getInt8(0));
```

关于Endian方面请看后边章节。

## 数据的存储方式
说说这些类型数组元素的数据表示方式吧！当然，我不打算在这里全部讲完，特别是IEEE规定的单精度浮点数和双精度浮点数是怎么用二进制表示的估计能另起一篇文章。

一个整型数字写成2进制之后以书写习惯来说左边是高位，右边是低位。举个例子，int8的5使用二进制是这么表示的：

```text
0000 0101
```

最左边的0是符号位，表示非负数，101则是5的二进制写法。

负数无法使用非负数的规则来表示，比如-1用非负数的规则表示是：

```text
1000 0001
```

考虑到-1如果加上1值应该为0，那么该值加上1却会变成-2：

```text
1000 0010
```

也就是在计算方式上出现问题了，所以二进制是采用了补码的方式来表示，补码可以完全不考虑符号位，它的规则如下：

* 非负数直接用正常的二进制数表示
* 负数是绝对值二进制取反加1

按这种规则，-1的绝对值二进制是：

```text
0000 0001
```

取反后是：

```text
1111 1110
```

再加1则是：

```text
1111 1111
```

我们再尝试给它加上1，看看值是多少：

```text
1 0000 0000
```

明显高位溢出了，于是将高位多余的1截掉变成：

```text
0000 0000
```

它的值如我们所料是0，这就是补码。uint8由于没有正负之分，它的所有值都是非负数，所以就不需要考虑补码。

## 数据溢出截断

首先需要了解一下int16用二进制是如何表示的，以3850为例子，它的二进制表示方式是这样子的：

```text
00001111 00001010
```

int16占了两个字节，因此它被截成两断，我们把左边的字节称为高位字节，右边的字节称为低位字节。在内存中它是如何存放的呢？建立一个Int8Array来看一下：

```javascript
var buffer = new ArrayBuffer(2);
var int16Array = new Int16Array(buffer);
var int8Array = new Int8Array(buffer);

int16Array[0] = 3850;

// Int16Array [ 3850 ]
console.log(int16Array);

// Int8Array [ 10, 15 ]
console.log(int8Array);
```

从运行结果看，3850被截断成两个int8值：

```text
int8Array[0] = 10 = 00001010
int8Array[1] = 15 = 00001111
```

看起来很反人类是吧？高位字节不是放在左边下标为0的字节上，却放在右边下标为1的字节上，跟我们的书写习惯反过来了！

事实上，数据在内存中存储并没有硬性规则高位字节必须放在左边（虽然它更符合人类阅读习惯），具体实现也是根据当前CPU的实现。大多数计算机是以高位优先的顺序存放数据（即高位在左，低位在右），但基于Intel CPU的计算机则是反过来以低位优先存放数据，我的本子就是Intel CPU的，因此我的运行结果就是Int8Array [ 10, 15 ]，或许换个电脑就会变成Int8Array [ 15, 10 ]。

那么当数据溢出截断又是怎么处理的？尝试着给一个int8字节赋值3850，看看最终得到的值是什么:

```javascript
var int8Array = new Int8Array([3850]);
//Int8Array [ 10 ]
console.log(int8Array);
```

很明显，它把15抛弃了，也就是保存了低位字节，抛弃高位。事实上，溢出处理在各种CPU上都是保留低位能保留的字节，把高位的截断，这个与数据存储是按高位优先还是低位优先没什么关系。

## 字节序处理

请先看以下示例代码在我机子上跑的情况：

```javascript
var buffer = new ArrayBuffer(2);
var view = new DataView(buffer);
var int16Array = new Int16Array(buffer);
view.setInt8(0, 10);
// 10
console.log(view.getInt8(0))
// 2560 用二进制表示是：00001010 00000000，
console.log(view.getInt16(0));

int16Array[0] = 10;
// 2560!
console.log(view.getInt16(0));
```

仔细看结果，很明显，view使用的是高位优先的读写方式，而TypeArray使用的是低位优先的读写方式。在内存中两个字节的数据在setInt8(0, 10)的时候会变成

```text
00001010 00000000
```

按照我计算机的读写规则应该是低位优先，也就是实际上这里应该被解读成

```text
00000000 00001010
```

也就是得到10，但实际上使用view却得到了2560，也就是高位优先。

DataView默认是使用big-endian方式，也就是高位优先进行读写。但在读写多字节数据的时候，可以通过传入值为true的little-endian参数来要求使用低位优先规则读写数据。
