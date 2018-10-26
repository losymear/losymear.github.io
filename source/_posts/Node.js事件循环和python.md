----
title: Node.js和python的event loop
date: 2018-10-23 16:30:11
tags:
    - blog
    - python
    - Node.js
toc: true

----


> *when A happens, do B*.

  <!-- more -->
## Node.js事件循环
<!-- 一张更好理解的图 https://files.catbox.moe/5b4k5c.jpg -->
> Node uses the Event-Driven Architecture: it has an Event Loop for orchestration and a Worker Pool for expensive tasks.

建议阅读[Event Loop and the Big Picture — NodeJS Event Loop Part 1](https://jsblog.insiderattack.net/event-loop-and-the-big-picture-nodejs-event-loop-part-1-1cb67a182810)系列，绝对是对node.js事件循环解释的最清楚的文章。


### 概念
#### IO
Node.js中的**IO**主要是指[libuv](https://libuv.org/)所支持的和系统磁盘或网络的交互。

#### 阻塞与非阻塞
Node中的调用有两种，阻塞（*bocking*）和非阻塞（*non-blocking*）：
- 阻塞：Node.js执行一段JS代码前必须等待一段非JS操作完成。发生这种情况的原因是，当一个阻塞操作发生时，event loop无法继续执行JS。在Node.js中，如果较差的性能是由于CPU计算密集而不是等待一段非JS操作如IO所导致，那么通常不称之为阻塞。 Node.js标准库中使用libuv的同步方法（如`fs.readSync`）是最常见的阻塞操作。native模块也可能具有阻塞方法。
- Node.js标准库中的IO操作也提供了异步版本，即非堵塞，并接受回调函数，方法名不以`Sync`结尾。比如`fs.readFile(path[, options], callback)`为非阻塞函数，`fs.readFileSync(path[, options])`为阻塞函数。

阻塞代码同步执行，非阻塞代码异步执行。以`fs`模块为例：
同步代码：

```js
const fs = require('fs');
const data = fs.readFileSync('/file.md'); // blocks here until file is read
console.log("文件读取完成");
// morework();
```
非阻塞代码：

```js
const fs = require('fs');
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
});
console.log("文件读取未完成");
// morework();
```
第一段代码看起来比较简单，由于`fs.readFileSync`是堵塞的，在读取文件时无法执行任何JS代码直到文件读取完成。另外一个缺点是如果文件读取错误会抛出异常，如果不捕获程序会crash；`fs.readFile`则由使用者决定是否处理异常：如果读取文件出错回调函数中的参数`err`会传入错误对象，否则为`null`。
第一段代码的`morework()`需要等待文件读取完成才会执行，而非阻塞的第二段代码的`morework()`不需要等待文件读取完成就能继续执行（只不过无法使用读取到的文件内容）。这种**无需等待IO操作即可继续执行JS代码**的核心特性是Node.js能有较高的吞吐量的原因，适合做IO服务器。

<!--
node.js的异步机制是基于事件的，所有的I/O、网络通信、数据库查询都以非阻塞的方式执行，返回结果由事件循环来处理。node.js在同一时刻只会处理一个事件，完成后立即进入事件循环检查后面事件。这样CPU和内存在同一时间集中处理一件事，同时尽量让耗时的I/O等操作并行执行。 -->

#### 并发与吞吐量
Node.js中执行JS是单线程环境，并发是指event loop完成一些工作后执行callback的能力。所以任何想要以并发的方式运行的代码，都必须要在非JS操作（如IO）运行时能让event loop持续工作（即不能堵塞事件循环）。

举个例子，假设一个web服务器需要50ms来完成一个请求，其中的45ms是数据库IO，能被异步执行。使用非堵塞的、异步执行的操作能让剩余的45ms用来接收其它请求，这和使用堵塞代码相比大大提高了吞吐量。
event loop和诸多其它语言的模型不同，比如Java需要创建额外的线程来接收并发请求。

### Node.js Event Loop
本节来自[官方guide](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)。
event loop允许Node.js执行非堵塞IO操作——尽管JS是单线程——通过尽可能地将操作交给系统内核来执行。
Node.js的设计受启发于Ruby的[Event Machine](https://github.com/eventmachine/eventmachine)或者Python的[Twisted](http://twistedmatrix.com/)，不过更进一步。它的event loop是Node.js程序运行时构建而不是作为库（库需要导入）。在上述两者中都有一段堵塞调用来启动event loop，典型行为是在脚本开头定义一个callback然后通过阻塞调用如`EventMachine::run()`来启动服务。Node.js没有这类*start-the-event-loop*调用，**脚本开始执行脚本后就直接进入event loop**；当没有需要执行的callback，Node.js会退出事件循环。这一行为类似浏览器中的JS——event loop对用户隐藏了（不过还是要了解event loop，不然可能写出低效代码）。
**记住，event loop**就是JS代码的执行线程。通过`node index.js`执行代码时，会进入poll阶段（见下一节），然后注册callback。因为现代内核（kernel）是多线程的，能在后台执行多个操作。当其中一个操作执行完成时，内核会告知Node.js，所以对应的callback会添加到`poll`队列中并最终被执行。
Node.js启动时会初始化event loop，然后处理输入的js脚本。js脚本可能会调用异步API、设置定时器，或者调用`process.nextTick()`，然后开始处理event loop。下图简要的表示了event loop的操作顺序：
![Node.js event loop操作顺序](https://files.catbox.moe/nng5e7.jpg)

#### 事件循环阶段
上图每个过程表示event loop的一个阶段。每个队列有一个callback的FIFO队列要执行。当event loop进入某个阶段，它会执行该阶段特定的一些操作，然后执行该阶段队列中的callback，直到队列元素被耗尽或者执行了最大允许数的callback。当元素被耗尽或达到callback限额，event loop会进入下一阶段。
在每轮event loop间，Node.js会检查它是否在等待异步IO或者timer，如果不是则会立即关闭。
<!--Since any of these operations may schedule more operations and new events processed in the poll phase are queued by the kernel, poll events can be queued while polling events are being processed. 因此，-->运行时间长的callback能允许poll阶段运行比timer阈值更长的时间。
#####  **timers**阶段
该阶段会执行由`setTimeout()`和`setInterval()`设置的callback。
一个timer会指定`threshold`，当过了`threshold`毫秒时间后指定的callback会被执行。比如`setTimeout(printCurrentTime, threshold)`。当过了指定的时间，timer回调会尽可能的早地执行，只要它能被调度；不过，操作系统的调度或者其它回调可能会推迟定时器回调的执行。
**注意：** 从技术角度来讲，poll阶段控制timer的执行。

```js
const fs = require('fs');

function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;
  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);


// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();
  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```
示例中有一个定时器设置threshod为100ms，读取`'/path/to/file'`文件需要95ms，读取好后的回调会阻塞10ms。
当event loop进入poll阶段时，它的队列为空（因为`fs.readFile()`）未完成，所以它会等待剩余的时间直到达到timer的阈值。当等待到过了95ms，`fs.readFile()`读取完毕并且它的回调会被加入到`poll`队列并被执行，执行回调需要10ms。回调执行好后，队列没有更多的callback并且已经达到timer的阈值，于是回到timer阶段并执行timer的回调。所以timer从设置到执行共用了105ms。
注意，为了避免poll阶段一直占据event loop，libuv会有一个最大时间（依赖系统），超过该时间停止轮询更多事件。

##### **pending callbacks**阶段
执行推迟到下一轮事件循环的IO回调。
这一阶段用于执行一些系统操作的回调如TCP错误的回调。比如一个TCP socket尝试连接时收到`ECONNREFUSED`异常，一些`*nix`系统会等待异常报告。这些会被加入到pending callbacks阶段的队列中。

##### **idle,prepare**阶段
只在内部使用。

##### **poll**阶段
获取新的IO事件；执行IO相关的callback（几乎所有的callback，除了close callback、定时器设置的callback、`setImmediate()`）。
主要有两个功能：
1. 计算它需要阻塞多久来轮询IO，然后
2. 执行`poll`队列中的事件
如果event loop进入poll阶段且没有设置好的timer，那么会发生下述事情之一：
1. 如果`poll`队列不为空，那么event loop会遍历callback队列，并同步执行，直到队列中的callback被耗尽，或者达到系统相关的限制
2. 如果`poll`队列为空，那么会发生下面两件事：
    1. 如果脚本已被`setImmediate()`调度，那么event loop会结束poll阶段并进入check阶段来执行这些调度好的callback
    2. 如果未被`setImmediate()`调度，那么event loop会等待callback被添加到队列中（如上述例子），并立即执行
当`poll`队列为空时，event loop会检查是否有timer已经到阈值了。如果有一到多个timer到达阈值，就会回到timers阶段并执行这些timer的回调

##### **check**阶段
执行`setImmediate()`设置的callback。
如果poll阶段处于闲置状态并且有callback被`setImmediate()`添加到队列中，那么event loop会进入`check`阶段而不是傻等。

##### **close callbacks**阶段
一些关闭事件的callback，如`socket.on("close", ...)`
如果一个socket或者handle被突然关闭（如`socket.destroy()`），那么`"close"`事件就被发送到这个阶段，否则会由`process.nextTick()`发起。

#### Node.js事件循环的优点
1. 与其他常见的并发模式形成对比。基于OS线程的模式性能相对低效且不易使用。
2. 不需要当心死锁等问题...因为没有锁。
3. Node.js中的函数几乎没有直接操作IO的，因此进程不会由于Node.js的API而堵塞。由于不会堵塞，非常适合在Node上开发可扩展系统。

### `setImmediate()`、`setImmediate()`、`process.nextTick()`、`Promise`

#### `setTimeout()` vs `setImmediate()`
`setImmediate()` 是在当前poll阶段完成时执行，`setTimeout()`是在经过threshold后执行。
> `setImmediate()` will execute code at the end of the current event loop cycle. This code will execute after any I/O operations in the current event loop and before any timers scheduled for the next event loop.

如果不在IO循环中执行如下代码，两个timer的顺序无法确定，取决于处理器性能：

```js
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```
但是如果移到IO循环中执行，则`setImmediate()`会在`setTimeout()`前执行：

```js
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```
使用`setImmediate()`优于`setTimeout()`的地方在于如果在一个IO循环中，`setImmediate()`始终在其它timer前执行。


#### `process.nextTick()`
尽管`process.nextTick()`是一个异步操作，它并不在上面的event loop示图中。因为它不属于event loop中的一部分。相反，`nextTickQueue`会在当前操作后立即执行，不管当前是什么阶段。
如果递归调用`process.nextTick()`会导致event loop无法进入poll阶段。允许这一API的一个原因是“所有的API都可以是异步的，即使它不需要这么做”的设计哲学。
`process.nextTick(cb)`会让`cb`在当前其余的用户代码执行完毕、但在event loop允许继续前执行`cb`。
如下示例代码中，虽然`someAsyncApiCall`有异步的签名形式，但是`callback()`是同步调用的，此时`bar`的值还来不及设置，而如果将`callback()`改造成`process.nextTick(callback);`则能生效。

```js
let bar;

// this has an asynchronous signature, but calls callback synchronously
function someAsyncApiCall(callback) {
    callback();
    //process.nextTick(callback);
}

// the callback is called before `someAsyncApiCall` completes.
someAsyncApiCall(() => {
  // since someAsyncApiCall has completed, bar hasn't been assigned any value
  console.log('bar', bar); // undefined
});

bar = 1;
```

一个具体案例：

```js
const server = net.createServer();
server.on('connection', (conn) => { });

server.listen(8080);
server.on('listening', () => { });
```
`listen`在event loop开始时调用，

文档建议在任何场合都应该使用`setImmediate()`替代`process.nextTick()`，因为它更容易理解，而且与浏览器兼容。


#### `Promise`
官方文档未提到`Promise`，建议阅读[Promises, Next-Ticks and Immediates](https://jsblog.insiderattack.net/promises-next-ticks-and-immediates-nodejs-event-loop-part-3-9226cbe7a6aa)。

Node.js原生`Promise`的回调可以认为是一个微服务（microtask），并会放到微服务队列中，该队列在nexttick队列之后。
![Native Promises](https://files.catbox.moe/o5w68x.jpg)

`Q`、`Bluebird`等库出现于Node.Js提供内置的`Promise`之前，都是使用内置接口如`Process.nextTick()`、`setImmediate()`等。
事实上知道Node.Js的event loop原理以及`Process.nextTick()`、`setImmediate()`这两个API后，也能轻松的实现一个`Promise`。


### Node.js异步示例代码

#### 回调
使用`fs`模块提供的异步API来读取文件，打印，然后删除：

```js
const fs = require('fs');
fs.readFile('/file.md', (readFileErr, data) => {
  if (readFileErr) throw readFileErr;
  console.log(data);
  fs.unlink('/file.md', (unlinkErr) => {
    if (unlinkErr) throw unlinkErr;
  });
});
```
要等到`fs.readFile()`读取到文件内容后再删除它，所以`fs.unlink()`操作在`fs.readFile()`的回调中。如果在删除文件后再创建同名文件夹`file.md`，那么`fs.mkdir()`还需要放置到`fs.unlink()`的回调中... 这就是著名的callback hell，回调链过长会导致代码不易阅读难于维护，如[这个例子](https://files.catbox.moe/7mhvok.jpg)。

#### `Promise`、`async/await`
*`fs`模块未提供`Promise`形式的API，可以使用bluebird包调用`var fs = Promise.promisifyAll(require("fs"));`来转换，这样可以使用`fs.xxxAsync()`返回`Promise`。*

为了避免callback hell，ES6加入了`Promise` API，使用链式调用的方式如
```js
`fs.readFileAsync("./file.md")
   .then(()=>fs.unlinkAsync("./file.md"))
   .then(()=>fs.mkdirAsync("./file.md"))
   .catch(e=>{});
```
不过链式过长还是不易阅读。最优雅的方式是ES8的`async/await`，它是`Promise`的语法糖。
使用`async/await`的示例：

```js
(async function(){
    await fs.readFileAsync("./file.md").catch(e=>{});
    await fs.unlinkAsync("./file.md").catch(e=>{});
    await fs.mkdirAsync("./file.md").catch(e=>{});
})();
```

`async/await`语法的写法很容易被当作是同步堵塞代码，一个技巧是看到`await`表达式就想象代码增加一个回调函数。

### 对event loop的常见误解
#### 误解1：event loop执行在一个单独的线程，用户JS代码在一个单独的线程
Node.js只有一个主线程，用来执行JS代码的线程就是运行event loop的线程。

#### 误解2： 所有的异步事件是由Worker Pool线程池执行
libuv虽然创建了一个线程池，但是当今的操作系统也提供了为IO任务一些异步接口（如Linux的[AIO](http://man7.org/linux/man-pages/man7/aio.7.html)）。只要有可能，libuv就会使用这些异步接口，而不是使用线程池。这同样适用于第三方子系统如数据库，驱动的作者会尽可能的使用异步接口而不是线程池。
只有在没有别的方式时，才会使用线程池来处理异步IO。

#### 误解3： event loop类似一个栈或队列
有些人认为event loop会继续的遍历一个FIFO任务队列，并当有一个任务完成时执行相应的回调。
事实上event loop是由几个阶段构成的，如上文所述。

### [深入Node.js工作原理](https://nodejs.org/en/docs/guides/dont-block-the-event-loop)
Node.js的event loop已经讲述清楚了，这一节可以跳过。
在Node中主要有两种线程：一个event loop（或者叫主线程、事件线程），以及Worker Pool中的多个`Worker`（即线程池）。

#### Worker Pool
Node.js提供了一个[Worker Pool](https://nodejs.org/en/docs/guides/dont-block-the-event-loop)，在libuv中实现，它暴露了一个通用的任务（task）提交API。Node.js使用Worker Pool来处理“昂贵”的操作，这些操作包括操作系统没有提供非堵塞API的IO操作，以及一些CPU密集型任务。Node.js模块用到Worker Pool的API有：
1. IO密集型
    1. [DNS](https://nodejs.org/api/dns.html)： `dns.lookup()`、`dns.lookupService()`
    2. `fs`模块：除了`fs.FSWatcher()`以及同步API的所有API会使用到libuv的线程池
1. CPU密集型
    1. [Crypto](https://nodejs.org/api/crypto.html)： `crypto.pbkdf2()`、`crypto.randomBytes()`、`crypto.randomFill()`
    2. [Zlib](https://nodejs.org/api/zlib.html#zlib_threadpool_usage) 同步API外的所有API会使用到libuv的线程池

对于大多数Node应用上述API是Worker Pool处理的任务的唯一来源。不过使用到[C++ add-on](https://nodejs.org/api/addons.html)的应用或模块也能提交其他任务到Worker Pool。
当在event loop的callback中调用上述API时，event loop会为API进行Node C++绑定，并提交任务到Worker Pool。这需要一定代价，不过和整个任务的消耗比起来是微不足道的。提交任务时，Node会提供一个指针指向对应的C++函数。

#### Node.js如何决定接下来该执行什么代码？
抽象地讲，event loop和Worker Pool分别维护待处理event和待处理task的队列。
但是实际上event loop并没有真正的维护一个event队列，相反，它会有一组文件描述符（file descriptor），要求操作系统通过[epoll](http://man7.org/linux/man-pages/man7/epoll.7.html)(Linux)、[kqueue](https://developer.apple.com/library/content/documentation/Darwin/Conceptual/FSEvents_ProgGuide/KernelQueues/KernelQueues.html)(OSX)、event ports(Solaris)或[IOCP](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365198.aspx)(Windows)等机制来监视。当操作系统告知有一个file descriptor已经准备就绪，event loop会把它翻译成适当的event并调用和该event相关联的callback。如果想要了解更多关于这一过程，可以看[Youtube|Node's Event Loop From the Inside Out by Sam Roberts](https://www.youtube.com/watch?v=P9csgxBgaZ8)。
Worker Pool使用了真正的队列，队列中的元素是要被处理的任务。一个Worker会从队列中弹出一个任务并对它进行处理，当完成时Worker会为event loop发起一个"At least one task is finished"事件。

#### 这种设计意味着...
在“on-thread-per-client”的架构如Apache中，每个执行中的client请求都会有它独自的线程。如果处理某个client的线程堵塞了，那么操作系统就会中断堵塞的线程，然后把线程另外一个client使用。操作系统确保要求少量计算工作的client不会为要求大量计算工作的client（甚至是堵塞）付出代价。
而Node.js给大量的client使用少量的线程，如果一个线程在处理某个client的请求时堵塞了，那么其它未处理好的client请求可能就不会有机会执行，直到线程完成它的工作。因此client的公平对待需要由你的代码来保证。这意味着你不能在一个callback或者task中做过多的工作。


## 浏览器JS的事件循环
和Node.js不同的模型。Node.js的事件循环是由libuv驱动，而浏览器的事件循环是由浏览器驱动，具体见[mdn|event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)。不过两者的表现是差不多的。

## Python的事件循环
python3的`asyncio`提供了异步编程能力。关键字：*task*、*event loop*、*coroutines*。

*event loop*： 一个loop，它使得python3也能异步执行程序。它每一时刻只有一个task在工作，其它的在等待。所以event loop的时间很宝贵，因为有很多task在等待工作中的task。因此如果一个task执行了一个堵塞操作，比如发送请求，那么它就会放出event loop的控制权。event loop的作用简单描述是*when A happens, do B*（A是event， loop是一直执行循环来发现event），比如浏览器中是当你点击按钮时（*when A happens*），它绑定的回调执行（*do B*）。`asyncio`标准库提供了event loop的功能，它的一个作用是当一个socket的I/O可以读写时（*when A happens*）能执行后续操作。除了IO外，event loop也可以用于在另一个线程或子进程执行代码，使用event loop作为调度器。
*task*： 用于在event loop中运行coroutine，具体可见[文档](https://docs.python.org/3/library/asyncio-task.html#task-object)
*coroutine*： 依靠coroutine（协程）来放出event loop的控制权。coroutine是subroutine概念的有状态推广。subroutine即方法或者函数，它可以运行subroutine来执行计算，也可以多次调用，不过两次subroutine间无法保持状态，每次调用都是全新的调用。而coroutine则是有状态的，它类似subroutine，不过能在两次调用间保持状态。也就是说，在两次调用间你可以当做是pause状态，而再次调用可以当做是resume。在python3.5+中，一个coroutine通过`await`来暂停。在一个coroutine中运行`wait anotherCoroutine`时，会暂停该coroutine并将event loop交给`anotherCoroutine`。
示例：

```python
import asyncio

# definition of a coroutine
async def coroutine_1():
    print('coroutine_1 is active on the event loop')

    print('coroutine_1 yielding control. Going to be blocked for 2 seconds')
    await asyncio.sleep(2)

    print('coroutine_1 resumed. coroutine_1 exiting')

# definition of a coroutine
async def coroutine_2():
    print('coroutine_2 is active on the event loop')

    print('coroutine_2 yielding control. Going to be blocked for 5 seconds')
    await asyncio.sleep(5)

    print('coroutine_2 resumed. coroutine_2 exiting')

# 并发执行两个coroutine
async def main1():
   await asyncio.gather(coroutine_1(),coroutine_2())

# 按顺序执行两个coroutine
async def main2():
    await coroutine_1()
    await coroutine_2()

print("并发执行两个coroutine")
asyncio.run(main1())
print("\n\n")
print("按顺序执行两个coroutine")
asyncio.run(main2())
```

注意点：
- `asyncio.sleep(delay)`是asyncio包预定义的coroutine，它不会堵塞event loop，而是延迟当前task的执行，并允许其它task工作。当执行到`await asyncio.sleep`时，它会释放对event loop的控制权，并让loop在指定时间后唤醒它。当过了指定时间后，它会重新获取event loop的控制权，返回该语句的调用结果，并继续执行它的调用者（比如这里的`coroutine_1`和`coroutine_2`）。
- 调用一个`async def`定义的方法不会执行它，而是创建一个coroutine对象。`await`后面跟的是一个coroutine而不是coroutine方法定义
- event loop运行的是task，而不是coroutine。执行到`await coroutine_object`时，coroutine会被包装成task并在当前event loop立即执行。
- 可以使用`asyncio.create_task(coro)`或者`asyncio.create_task(coro)`来把coroutine包装成task。

一个更实际的例子：

```python
import asyncio

# this is a coroutine definition
async def fake_network_request(request):
    print('making network call for request:  ' + request)
    # simulate network delay
    await asyncio.sleep(1)

    return 'got network response for request: ' + request

# this is a coroutine definition
async def web_server_handler():
    # schedule both the network calls in a non-blocking way.

    # ensure_future creates a task from the coroutine object, and schedules it on the event loop
    task1 = asyncio.ensure_future(fake_network_request('one'))

    # another way to do the scheduling
    task2 = asyncio.get_event_loop().create_task(fake_network_request('two'))

    # simulate a no-op blocking task. This gives a chance to the network requests scheduled above to be executed.
    await asyncio.sleep(0.5)

    print('doing useful work while network calls are in progress...')

    # wait for the network calls to complete. Time to step off the event loop using await!
    await asyncio.wait([task1, task2])

    print(task1.result())
    print(task2.result())

asyncio.run(web_server_handler())
```

 想要简单了解python的event loop如何工作可以阅读[How the heck does async/await work in Python 3.5?](https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/)。

## Read More
- Node.js
    - [Event Loop and the Big Picture — NodeJS Event Loop Part 1](https://jsblog.insiderattack.net/event-loop-and-the-big-picture-nodejs-event-loop-part-1-1cb67a182810)
    - [Promises, Next-Ticks and Immediates— NodeJS Event Loop Part 3](https://jsblog.insiderattack.net/promises-next-ticks-and-immediates-nodejs-event-loop-part-3-9226cbe7a6aa)
    - [官方guide](https://nodejs.org/en/docs/guides/#node-js-core-concepts)
    - [Morning Keynote- Everything You Need to Know About Node.js Event Loop](https://www.youtube.com/watch?v=PNa9OMajw9w)
    - [What you should know to really understand the Node.js Event Loop](https://medium.com/the-node-js-collection/what-you-should-know-to-really-understand-the-node-js-event-loop-and-its-metrics-c4907b19da4c)
    - [napajs](https://github.com/Microsoft/napajs) 开源模块，提供了对Node.js多线程支持
    - [libuv: Asynchronous I/O made simple](http://libuv.org/)
- Python3
    - [Python 3's Killer Feature: asyncio](https://eng.paxos.com/python-3s-killer-feature-asyncio)
    - [How the heck does async/await work in Python 3.5?](https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/)
    - [a tale of event loops](https://github.com/AndreLouisCaron/a-tale-of-event-loops)
    - [A simple introduction to Python's asyncio](https://hackernoon.com/a-simple-introduction-to-pythons-asyncio-595d9c9ecf8c)  基本描述清楚python的事件循环
    - [How does asyncio actually work?](https://stackoverflow.com/questions/49005651/how-does-asyncio-actually-work)