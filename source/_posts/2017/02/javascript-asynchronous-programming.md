---
title: Javascript异步编程
categories: javascript
date: 2017-03-29 00:00:00
tags:
    - 异步编程
    - Promise
    - await
    - async
---

前几年碰上一个需求，内容是一个注册表单，其中有几个字段需要调用服务器的接口进行验证才允许提交，当时自己写了一个异步队列封装用来实现这个功能，当然为了赶进度仅是实现了功能，没有什么复用性，而且代码组织也比较差。

随后我想想这种需求肯定一堆人会碰上，要不找找有没有什么好的解决方案，当时让我找到了Promise/A+规范以及when.js。很幸运的，Promise/A+进入了制定Web标准的那帮家伙的眼里，并且得到认可，目前在不少浏览器中已经可以直接使用Promise对象，甚至是可以使用async/await关键词将异步操作变得非常简单起来，当然这也是需要配合Promise。若是考虑到兼容性问题，我们还有[when.js](https://github.com/cujojs/when)之类的开源库可以使用。

关于Promise/A+的规范内容可以查阅[《Promise/A+》](https://segmentfault.com/a/1190000002452115)或者它的英文原文[《Promises/A+》](https://promisesaplus.com/)。

<!-- more -->

## Promise

Promise是属于ES6的新增内容，在许多新版本的浏览器可以被使用。Promise的实例代表着一个异步行为，它拥有三种状态：

* pending - 初始状态，它表示该行为未得出结果
* fulfilled - 行为已经完成，并且成功
* rejected - 行为失败

其中pending可以转化为fulfilled或者rejected，也就是从初始状态变成成功或者失败。

以下代码创建了一个Promise实例：

```javascript
let promise = new Promise((resolve, reject) => {
    let startTime = new Date().getTime();
    
    setTimeout(() => {
        let finishTime = new Date().getTime();
        resolve(finishTime - startTime);
        // reject("我计算不出来！");
    }, 1000);
});
```

### Promise原型

Promise构建函数拥有两个参数resolve以及reject，resolve表示操作成功并且可以将结果当成参数传递作为结果，而reject表示操作失败，它的参数是失败的原因。

Promise.prototype拥有两个重要的方法：

* then(onFulfilled, onRejected) - 为promise实例添加成功或者失败回调，当promise的状态被设置为fulfilled时（即调用resolve函数），则调用onFulfilled回调。若是被设置为rejected时（即调用了reject函数）则调用onRejected回调。then将返回一个新的promise对象，它将以onFulfilled或者onRejected的值填充。onRejected可以忽略，那么当原promise对象的状态是rejected时，新的promise对象也同样是rejected，并且值是原promise对象的值。新的promise对象同样可以调用then方法设置新的onFulfilled、onRejected回调函数。
* catch(onRejected) - 为promise对象添加onRejected回调，回调函数将会默认以返回值执行resolve操作。若返回一个新的Promise实例则会以该实例的结果进行resolve操作。

说真的，听起来非常绕，可以通过实例来理解：

```javascript
let promise = new Promise((resolve, reject) => {
    // 第一步
    resolve(1234);
}).then(v => {
    // 第二步的onFulfilled
    console.log(`上个操作传递过来的值是${v}`);
    return 4321;
}).then(v => {
    // 第三步的onFulfilled
    console.log(`上个操作传递过来的值是${v}`);
}).then(v => {
    // 第四步的onFulfilled
    console.log(`上个操作传递过来的值是${v}`);
});
```

输出结果是：

```text
上个操作传递过来的值是1234
上个操作传递过来的值是4321
上个操作传递过来的值是undefined
```

这里由于promise对象的then方法同样返回一个新的promise对象，它的值被设置为onFulfilled或onRejected的返回结果，所以支持链式调用，要特别注意这里的promise变量得到的是最后一次then方法调用的结果。

第一步调用了resolve(1234)，将new Promise构造出来的promise对象状态设置为fulfilled，值为1234。随后调用了promise对象的then方法，为它设置了一个onFulfilled回调函数，于是onFulfilled回调函数被调用，并且将1234作为参数v传递进去，因此第二步的onFulfilled函数输出了"上个操作传递过来的值是1234"。

第二步的onFulfilled函数返回了4321，而then函数返回了一个新的promise对象，它的状态被设置为fulfilled，并且值为4321。新的promise对象的then方法又被调用，4321作为值传递给它的onFulfilled回调函数作为参数v，因此第三步的onFulfilled函数输出了上个操作传递过来的值是4321。

第三步的onFulfilled没有return语句，一个没有return语句的普通函数返回结果当然是undefined，所以then函数返回了一个状态为fulfilled值为undefined的promise对象，它的then又被调用，将undefined作为第四步的onFulfilled的参数v，因此最后输出了"上个操作传递过来的值是undefined"。

代码中声明的promise变量的值就是最后的then返回的promise对象，最终它的状态是fulfilled，值是undefined。

如果onFulfilled或onRejected返回了一个promise对象，那么then返回的promise对象的结果将会以该promise对象的结果填充：

```javascript
let promise = new Promise((resolve, reject) => {
    // 第一步
    resolve(1234);
}).then(v => {
    // 第二步的onFulfilled
    console.log(`第一步传过来的值是${v}`);
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(4321);
        }, 100);
    });
}).then(v => {
    // 第三步的onFulfilled
    console.log(`隔了100ms后得到第二步传过来的值${v}`);
});
```

运行结果是：

```text
第一步传过来的值是1234
隔了100ms后得到第二步传过来的值4321
```

也就是then方法的onFulfilled或onRejected若是想做异步操作，则返回一个新的promise对象，并在promise对象的回调中进行resolve或者reject操作填充一个。而then则会以onFulfilled或onRejected返回的promise对象值返回一个新的promise对象。

onRejected回调是在promise对象状态被设置为rejected时被调用，then同样会以它的返回值填充一个promise对象：

```javascript
let promise = new Promise((resolve, reject) => {
    // 第一步
    reject("调用失败了！")
}).then(v => {
    // 第二步的onFulfilled
    console.log("这里不会被调用");
    return 0;
}, msg => {
    // 第二步的onRejected
    console.error(`第二步的onRejected被调用了：${msg}`);
    return 1;
}).then(v => {
    // 第三步的onFulfilled
    console.log(`成功，第三步得到了${v}`)
}, msg => {
    // 第三步的onRejected
    console.log(`失败，第三步得到了${msg}`);
});
```

输出结果是：

```text
第二步的onRejected被调用了：调用失败了！
成功，第三步得到了1
```

可以看到，由于第一步调用了reject设置了状态为rejected、失败原因为"调用失败了！"，因此第二步的onFulfilled并没有被调用，而是调用了onRejected回调，并把reject函数设置的错误原因作为参数msg传递给它。由于onRejected返回了1，因此第二步的then返回的promise对象的状态被设置为fulfilled，值为1，第三步的onFulfilled被调用，并且得到了1。

若是一个promise对象状态被设置为rejected，但它的then方法没有设置onRejected回调，那么then方法将会返回一个同样的promise对象，该对象若是then方法被调用，则会以同样的逻辑处理：

```javascript
let promise = new Promise((resolve, reject) => {
    // 第一步
    reject("调用失败了！")
}).then(v => {
    // 第二步的onFulfilled
    console.log("这里不会被调用");
    return 0;
}).then(v => {
    // 第三步的onFulfilled
    console.log(`成功，第三步得到了${v}`)
}, msg => {
    // 第三步的onRejected
    console.log(`失败，第三步得到了${msg}`);
});
```

输出结果为：

```text
失败，第三步得到了调用失败了！
```

这里第一步将promise对象状态设置为rejected、原因为"调用失败了！"，它的then方法被调用，但却没有传入onRejected回调，于是then方法返回了一个同样状态为rejected、原因为"调用失败了！"的promise对象，新的promise对象的then方法被调用了，而该方法传入了onRejected回调，于是输出了"失败，第三步得到了调用失败了！"。

catch方法跟then一模一样，但它仅能够设置onRejected回调函数：

```javascript
let promise = new Promise((resolve, reject) => {
    // 第一步
    reject("调用失败了！")
}).then(v => {
    // 第二步的onFulfilled
    console.log("这里不会被调用");
    return 0;
}).catch(msg => {
    // 第二步的onRejected
    console.log(`失败，第三步得到了${msg}`);
});
```

输出结果跟上一个例子一模一样：

```text
失败，第三步得到了调用失败了！
```

可以用稍微易懂的文字来描述这种：当某一步失败的时候，会调用后边碰到的第一个onRejected回调函数并将结果传递给它。

要特别注意，catch跟then的区别仅仅在于无法设置onFulfilled回调，它同样是返回一个新的promise对象，并将onRejected的结果（返回值或返回的promise对象的结果）填充到新的promise对象中：

```javascript
let promise = new Promise((resolve, reject) => {
    reject("调用失败了！");
}).catch(msg => {
    console.error(`失败原因是：${msg}`);
    return new Promise((resolve, reject) => {
        resolve(1234)
    });
}).then(v => {
    console.log(`catch传递过来的是${v}`);
});
```

输出结果是：

```text
失败原因是：调用失败了！
catch传递过来的是1234
```

所以，千万不要以为catch完就无法继续往下走了。

### Promise方法

Promise提供了四个方法：

* Promise.all(iterable) - iterable是一个可遍历的迭代器对象，比如Array、Map、Set之类的。iterable里每个项都是一个子Promise（或一个确定的值），当all执行后返回一个promise对象，如果iterable所有项都得到fulfilled结果时调用promise对象的onFulfilled回调，并将所有结果以数组的形式传入onFulfilled回调，并且顺序是跟iterable元素顺序一致。若是某个项被置为rejected状态时会马上调用onRejected回调，并将失败原因当成参数传给它。

```javascript
let promise = Promise.all([
    new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(1)
        }, 100);
    }),
    new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(2)
        }, 200);
    }),
    1234
]).then(values => {
    // 于300ms后输出结果为 [1, 2, 1234]
    console.log(values);
});
```

这里要注意的是，如果iterable的元素是一个非Promise实例，它本身就作为值并且状态为fulfilled；若是一个Promise实例，那么它的值就是Promise实例的结果。

以下是触发onRejected的示例： 

```javascript
let promise = Promise.all([
    new Promise((resolve, reject) => {
        setTimeout(() => {
            reject('这是失败的原因')
        }, 100);
    }),
    new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(2)
        }, 200);
    }),
    1234
]).then(values => {
    // 不会走到这
    console.log(values);
}, msg => {
    // 输出了"这是失败的原因"
    console.error(msg);
});
```

* Promise.race(iterable) - 跟Promise.all类似，但它不会等到所有项得到结果才调用onFulfilled或onRejected回调，而是只要有一个项得到结果马上调用相应的回调。

```javascript
let promise = Promise.race([
    new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(1)
        }, 100);
    }),
    new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(2)
        }, 200);
    }),
    1234
]).then(value => {
    console.log(value);
});
```

这个示例将马上输出1234，因为它是一个确定的值，相当于第一个得到fulfilled状态值为1234的子Promise对象。若是iterable里没有1234这一项将会在100ms后输出1，因为理论上第一个子Promise对象比第二个子Promise对象更早得到结果。

因此可以看出Promise.race与Promise.all不同的地方就在于Promise.race的iterable参数里每个项都处于竞争关系，谁更早确定结果谁的结果就会被作为参数传对相应的回调函数。

* Promise.resolve(value) - 返回一个状态为fulfilled的Promise对象，并将value作为值。

```javascript
Promise.resolve(1234)
    .then(v => {
        // 输出了1234
        console.log(`得到的是${v}`);
    });
```

要特别注意的是：**如果value是一个Promise实例，则会以该Promise实例的最终状态和结果来填充返回的Promise对象**。

```javascript
Promise.resolve(new Promise((resolve, reject) => {
    setTimeout(() => {
        reject('网络请示失败');
    }, 100);
})).then(v => {
    console.log(`输出的是${v}`);
}, msg => {
    console.error(`失败原因是${msg}`);
});
```

虽然调用的是Promise.resolve对象，但由于参数是个Promise实例，并且它的最终状态是rejected，因此最终还是调用了onRejected回调，输出了"失败原因是网络请示失败"。

* Promise.reject(reason) - 跟Promise.resolve类似，它会返回一个状态为rejected，失败原因为reason的Promise实例。

```javascript
Promise.reject('网络失败')
    .catch(msg => {
        console.error(`失败原因是${msg}`);
    });
```

要注意的是：**Promise.reject的reason参数哪怕是一个Promise实例都不会像Promise.resolve一般以该Promise实例的最终状态和结果为准，也就是说这种情况Promise.reject返回的Promise实例的状态依然是rejected，并且失败原因是传入的Promise实例参数**。

## 异步函数

### async与await

通过async关键词可以声明一个异步函数，异步函数将会返回一个Promise对象，可以通过then或catch来得到它的运行结果。

首先来看一个示例，有两个异步操作fn1以及fn2，它们将通过异步计算得出两个值，现在想求这两个值的和：

```javascript
let fn1 = () => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(10);
        }, 100);
    });
}

let fn2 = () => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(20);
        }, 200);
    });
}

Promise.all([fn1(), fn2()]).then(values => {
    return values[0] + values[1];
}).then(v => {
    console.log(`运算出fn1()+fn2()为${v}`);
});
```

使用async函数实现如下：

```javascript
let fn1 = () => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(10);
        }, 100);
    });
}

let fn2 = () => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(20);
        }, 200);
    });
}

async function fn() {
    return await fn1() + await fn2();
}

fn().then(v => {
    console.log(`运算出fn1()+fn2()为${v}`);
});
```

await在这里起了一个非常关键的作用，它只允许被使用在async函数中，当它被添加在一个Promise对象之前时（注意fn1和fn2都是返回了一个Promise对象），async函数将会暂停，等到Promise对象结果确定时再继续往下执行。也就是说await正如其名，让异步函数等待。

事实上正如上方示例，没有async和await也同样可以使用Promise实现，但有了它们我们的代码逻辑会显得更加扁平化，看起来就仿如一个同步操作，非常好理解。

异步函数在其中某个异步操作失败的时候会马上填充返回的Promise对象的状态为rejected，并且将该操作的失败原因当成失败原因，可以通过onRejected回调函数去处理：

```javascript
let fn1 = () => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            reject("网络连接失败");
        }, 100);
    });
}

let fn2 = () => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(20);
        }, 200);
    });
}

async function fn() {
    return await fn1() + await fn2();
}

fn().then(v => {
    console.log(`运算出fn1()+fn2()为${v}`);
}, msg => {
    console.error(`操作失败原因是：${msg}`);
});
```

由于fn1的结果是rejected，因此fn函数返回的Promise会在100ms时状态被置为rejected，并且调用了onRejected回调函数，输出了"操作失败原因是：网络连接失败"。

### 执行效率的优化

MDN上提供了一个有趣的例子可以给我们敲一下警钟：什么时候用await可是有讲究的：

```javascript
function resolveAfter2Seconds(x) {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve(x);
    }, 2000);
  });
}

async function add1(x) {
  var a = resolveAfter2Seconds(20);
  var b = resolveAfter2Seconds(30);
  return x + await a + await b;
}

add1(10).then(v => {
  console.log(v);  // prints 60 after 2 seconds.
});

async function add2(x) {
  var a = await resolveAfter2Seconds(20);
  var b = await resolveAfter2Seconds(30);
  return x + a + b;
}

add2(10).then(v => {
  console.log(v);  // prints 60 after 4 seconds.
});
```

add1和add2都会得出同样的结果，但是add1花了2秒，而add2花了4秒，这里关键在于add1在调用resolveAfter2Seconds时没有加await，也就是两个resolveAfter2Seconds几乎同时开始工作，它们将在2秒后一起确定结果。而add2在第一次调用resolveAfter2Seconds时已经加上await关键词，这时候add2会暂停，两秒后得到a的值才开始调用第二次resolveAfter2Seconds，然后再等2秒。

所以在编辑异步函数使用await关键词的时候，一定要注意一点：多个异步操作之间到底是并行关系还是顺序关系。

