# on-finished源码分析

> version 2.4.1

[on-finished](https://github.com/jshttp/on-finished#readme)是一个小而美的`Node.js`的`http`相关的库，该库的作用是在`http`请求`完成`、`关闭`或`出错`之后触发用户设置的回调函数。使用例子如下：

```js
const fs = require('fs');
const http = require('http');
const onFinished = require('on-finished');
const destroy = require('destroy');

http.createServer(function onRequest (req, res) {
  // 创建可读流读取package.json文件
  const stream = fs.createReadStream('package.json');
  // 将读取的内容以流的方式响应给请求方
  stream.pipe(res);
  // 在流结束/完成/出错之后触发回调来销毁创建的流
  onFinished(res, function () {
    destroy(stream);
  });
});
```

### 主体架构实现

在讲解源码之前，我们先约定两个概念：

- `Node`服务端和客户端的`http`的`请求/响应`统称为`消息`。
- `消息`的`关闭`、`完成`和`出错`都统称为`消息结束`

该库实现都在根目录下的`index.js`文件中，对外暴露两个函数：

```js
'use strict'

module.exports = onFinished
module.exports.isFinished = isFinished

/**
 * 推迟执行回调
 * 优先使用setImmediate控制回调的执行时机在异步IO之后
 */
var defer = typeof setImmediate === 'function'
  ? setImmediate
  : function (fn) { process.nextTick(fn.bind.apply(fn, arguments)) }
  
/**
 * 在消息关闭/完成/出错后执行回调
 */
function onFinished (msg, listener) {
  // 如果msg消息（请求/响应/流）已处于结束状态（完成、关闭、出错）
  if (isFinished(msg) !== false) {
    // 则当前事件循环结束后执行回调
    defer(listener, null, msg)
    return msg
  }

  // msg处于未结束状态，创建监听器
  attachListener(msg, wrap(listener))

  return msg
}
```

`onFinished`的逻辑就是两步：

- 如果`消息`已经`结束`，则直接调用`defer`执行回调函数
- 如果`消息`尚未`结束`，则通过`attachListener`侦听`消息`的`结束`然后触发回调

逻辑图如下所示：

![主体架构](https://note.youdao.com/yws/res/21605/972BD245A9784A93A0D2E799206316D6)

### Node消息回顾

在看`isFinished`函数的实现之前，我们得先回顾`Node`中的消息相关的知识点。`Node`中的消息分为`IncomingMessage`类型和`OutgoingMessage`类型。

- `IncomingMessage`类型的消息有`httpServer`的`req`对象和`httpClient`的`res`对象；
- `OutgoingMessage`类型的消息有`httpServer`的`res`对象和`httpClient`的`req`对象；

![image](https://note.youdao.com/yws/res/21441/CE448122EBFF41D698380BCE8641052C)

`IncomingMessage`消息对象上存在`complete`属性，值为`true`表示已接收并成功解析完整的`HTTP`消息。此属性作为一种确定客户端或服务器是否在连接终止之前完全传输消息的方法特别有用。同时，借用`鸭子类型`的思想，如果消息上存在`complete`属性也可以认为是`IncomingMessage`消息对象。我们来看下面的例子验证下`complete`:

```js
const http = require('http');

const server = http.createServer((req, res) => {
  // 此时req.complete值为false
  debugger;
  req.on('data', chunk => {});
  req.on('end', () => {
    // 此时req.complete值为true
    debugger;
    res.end('res data');
  });
});

server.listen(3000, () => {
  console.log('server running at port 3000.');

  const req = http.request('http://localhost:3000', (res) => {
    // 此时res.complete值为false
    debugger;
    const chunks = [];
    res.on('data', chunk => {
      chunks.push(chunk);
    });
    res.on('end', () => {
      // 此时res.complete值为true
      debugger;
      const data = Buffer.concat(chunks).toString();
      console.log('res data: ', data);
    });
  });
  
  // 发送请求结束
  req.end();
});
```

![image](https://note.youdao.com/yws/res/21489/27A71B224FF449A89D4D02CDABD274DC)

`OutgoingMessage`消息对象上存在`finished`属性，当请求端调用`req.end()`或者服务端调用`res.end()`后，`finished`属性值为`true`。同时，借用`鸭子类型`的思想，如果消息上存在`finished`属性也可以认为是`OutgoingMessage`消息对象。我们看下面的例子来验证一下`finished`：

```js
const http = require('http');

const server = http.createServer((req, res) => {
  // 此时res.finished值为false
  debugger;
  res.end('hello');
  // 此时res.finished值为true
  debugger;
});

server.listen(3000, () => {
  console.log('server running at port 3000.');
  
  // 发送http请求
  const req = http.request('http://localhost:3000', (res) => {
    // ...
  });
  
  // 此时req.finished值为false
  debugger;
  req.end();
  // 此时req.finished值为true
  debugger;
});
```

![image](https://note.youdao.com/yws/res/21493/00D741BA62F145DEAB4A15C753513605)

### 消息结束状态判断

回顾了Node消息的相关知识之后，我们来分析`isFinished`具体的实现：

```js
/**
 * 确定消息是否完成
 *
 * @param {object} 消息对象
 * @return {boolean}
 * @public
 */

function isFinished (msg) {
  // 对底层套接字的引用，即底层依赖的stream.Duplex双工流
  var socket = msg.socket

  /**
   * msg为OutgoingMessage对象
   * 调用msg.end()之前：msg.finished为false
   * 调用msg.end()之后：msg.finished为true
   */
  if (typeof msg.finished === 'boolean') {
    /**
     * msg.finished为false时还依旧判断socket.writable的原因在于
     * 有可能在调用msg.end()之前流被销毁了，比如调用了msg.destroy()
     */
    return Boolean(msg.finished || (socket && !socket.writable))
  }

  /**
   * IncomingMessage对象
   *
   * 注意：如果已接收并成功解析完整的 HTTP 消息，则 message.complete 属性将为 true
   * 该属性作为确认服务端或客户端在连接终止前所有消息是否传输完毕时特别有用
   */
  if (typeof msg.complete === 'boolean') {
    /**
     * 1、msg.upgrade，客户端的ws实际由http发送upgrade事件请求升级协议，
     * 由于Node.js接口限制，
     * 这里认为只要发送的upgrade事件就认为是消息结束了，即使数据还没消费完
     * 2、!socket.readable，只要流不再可写了则说明已经结束，即使没消息尚未完成。
     *  因为存在触发end事件之前先行销毁流的可能或者流出错的可能。
     * 3、最后msg.complete && !msg.readable表示消息完成且流也不再可写了则说明结束了
     */
    return Boolean(msg.upgrade || !socket || !socket.readable || (msg.complete && !msg.readable))
  }

  // 处理未知情况
  // 注意：onFinished内部认为未知情况也是消息结束了
  return undefined
}
```

`isFinished`的实现有很多注意点，首先`msg.socket`是对底层套接字的引用，即底层依赖的`stream`。在前面的知识点我们知道，我们通过判断消息对象是否`finished`属性或者`complete`判断是`OutgoingMessage`对象还是`IncomingMessage`对象，然后根据不同的消息类型进行不同的判断：

- `msg`为`OutgoingMessage`对象
    - 如果`finished`值为`true`，只说明消息已经结束
    - 否则的话判断套接字是否还可写，不可写的则说明消息已经结束

这里为什么要在`finished`为`false`时还要再判断套接字呢？原因就在于我们可能在请求结束之前流被销毁或者关闭了。例如，手动调用了`msg.destroy()`方法主动销毁了流，此时就没必要等到消息结束了，我们认为消息在流销毁的时候就已经结束了。

- `msg`为`IncomingMessage`对象
    - 如果`msg.upgrade`是`true`，则认为消息结束
    - 如果套接字不再可读也认为消息结束了
    - 最后如果消息已完成则消息也是结束了

这里的第二三个条件都好理解，但是为什么第一个条件`msg.upgrade`是`true`也认为是消息结束了呢？原因在于，首先`upgrade`表示请求将已建立的客户端/服务端连接升级为不同的协议，比如将连接从`HTTP1.1`升级到`HTTP2.0`或`HTTP`或`HTTPS`连接到`WebSocket`客户端发送的协议升级请求事件。而我们所熟知的`websocket`本质是`http`连接发送一个`upgrade`请求升级为`ws`。

但是呢，由于`Node.js`接口的限制，只要是`upgrade`消息的消息头的话，则直接认为消息已完成，即使在其消息被读之前。

问题又来了，那么对什么没对`OutgoingMessage`类型的消息处理`upgrade`消息头呢？因为`OutgoingMessage`类型的消息不存在`upgrade`的场景呀！

总结一下，`isFinished`的逻辑图如下所示：

![image](https://note.youdao.com/yws/res/21500/F3D647144F22441E982602CB024D73EE)


### 给消息添加侦听器的实现

当`消息对象`不是处于结束状态时，则是通过`attachListener`函数侦听消息对象的结束状态，然后触发用户设置的结束回调逻辑。`attachListener`的实现逻辑如下所示：

```js
/**
 * 附加侦听器到消息对象上
 *
 * @param {object} msg 消息对象
 * @return {function}
 * @private
 */
function attachListener (msg, listener) {
  /**
   * 获取已创建的侦听器对象
   *
   * attached是拥有一个queue属性的函数
   * queue属性存储了所有的侦听器
   * attached函数执行的结果就是遍历queue内的所有函数依次执行
   */
  var attached = msg.__onFinished

  // 创建侦听器对象
  if (!attached || !attached.queue) {
    // 创建侦听器对象
    attached = msg.__onFinished = createListener(msg)
    // 监听消息对象的结束状态，执行侦听器
    attachFinishedListener(msg, attached)
  }

  // 将用户的回调函数添加到侦听器队列
  attached.queue.push(listener)
}
```

这里的做法是先通过`msg.__onFinished`判断该消息对象有没有已经创建过侦听器对象，即之前有没有已经通过`on-finished`函数添加回调函数了。如果没有创建过侦听器对象，则先调用`createListener`函数创建侦听器对象。在确保侦听器对象存在之后，然后用户设置的回调函数添加到侦听器队列中。逻辑如下图所示：

![image](https://note.youdao.com/yws/res/21609/DC4BEA7D0402497292435DE5DE82A081)

`createListener`创建侦听器对象的逻辑如下：

```js
function createListener (msg) {
  // 侦听器函数，执行后一次调用所有的侦听器
  function listener (err) {
    if (msg.__onFinished === listener) msg.__onFinished = null
    if (!listener.queue) return

    // 获取侦听器队列的引用，并移除原引用
    var queue = listener.queue
    listener.queue = null

    // 依次执行队列中的函数
    for (var i = 0; i < queue.length; i++) {
      queue[i](err, msg)
    }
  }

  // 创建侦听器队列
  listener.queue = []

  return listener
}
```

### 监听消息对象的结束状态

在上面根据消息对象创建了侦听器对象之后，下面看如何侦听消息对象对结束状态然后触发侦听器的？`attachFinishedListener`函数的代码实现如下：

```js
var first = require('ee-first')

/**
 * 监听消息对象的结束状态
 *
 * @param {object} msg
 * @param {function} callback
 * @private
 */
function attachFinishedListener (msg, callback) {
  var eeMsg
  var eeSocket
  // 标记消息状态为未完成
  var finished = false

  // 设置消息状态为完成，并且调用用户的callback
  function onFinish (error) {
    eeMsg.cancel()
    eeSocket.cancel()

    // 设置消息状态为完成
    finished = true
    // 执行用户回调
    callback(error)
  }

  // msg的end或finish事件任意触发一个则触发onFinish
  eeMsg = eeSocket = first([[msg, 'end', 'finish']], onFinish)

  function onSocket (socket) {
    // 套接字不存在的时候监听了socket事件，调用后移除该事件监听
    msg.removeListener('socket', onSocket)

    // 如果是已完成状态则不再响应
    if (finished) return
    if (eeMsg !== eeSocket) return

    // socket套接字触发error或close事件后，触发消息onFinish的回调
    eeSocket = first([[socket, 'error', 'close']], onFinish)
  }

  if (msg.socket) {
    // socket already assigned
    onSocket(msg.socket)
    return
  }

  // 如果套接字不存在则先等待套接字分配完毕之后
  // 再嗲调用onSocket监听相关的error和close事件
  msg.on('socket', onSocket)

  if (msg.socket === undefined) {
    // istanbul ignore next: node.js 0.8 patch
    patchAssignSocket(msg, onSocket)
  }
}
```

这个函数是该库非常关键的一个函数，这里要先了解一下[ee-first](https://github.com/jonathanong/ee-first)库的作用，它的主要作用是目标对象指定的任意一个事件触发就执行回调。所以这里先通过`eeMsg = eeSocket = first([[msg, 'end', 'finish']], onFinish)`监听消息对象的`end`、`finish`的任意一个事件触发了就执行`onFinish`函数，`onFinish`调用后则会调用用户设置的回调函数。注意的是：

- `end`事件是`IncomingMessage`对象**结束**事件
- `finish`事件是`OutgoingMessage`对象**结束**事件

紧接着通过判断套字是否已经被分配完成，如果套接字已经生成则调用`onSocket`函数。`onSocket`函数则是先移除已经添加的`socket`事件，然后通过`ee-first`监听socket对象的`error`或`close`任一事件触发后执行`onFinish`函数。注意的是：

- `error`表示`socket`套接字**出错**事件
- `close`表示`socket`套接字**关闭**事件

最后如果`socket`套接字尚未分配的话，则先监听`socket`套接字分配完成后再监听套接字的`error`或`close`事件。事件触发后，消息结束，执行用户回调函数。关系逻辑图如下：

![image](https://note.youdao.com/yws/res/21601/9292534878A34E7E9B6501B98BE4FC0F)

到这里，本文大部分实现已经讲完了，但是还有一点，就是一开始在`onFinished`函数内部，如果消息没完成，则通过`attachListener`添加消息结束状态监听。但是在传入监听函数时先使用了`wrap`进行包裹:

```js
attachListener(msg, wrap(listener))
```

为什么要使用`wrap`包裹呢？这个`wrap`函数纠结对用户对回调函数做了什么处理呢？

### async_hooks

下面先放出来`wrap`的完整代码如下：

```js
var asyncHooks = tryRequireAsyncHooks()

function tryRequireAsyncHooks () {
  try {
    return require('async_hooks')
  } catch (e) {
    return {}
  }
}

/**
 * Wrap function with async resource, if possible.
 * AsyncResource.bind static method backported.
 * @private
 */
function wrap (fn) {
  var res
  // create anonymous resource
  if (asyncHooks.AsyncResource) {
    res = new asyncHooks.AsyncResource(fn.name || 'bound-anonymous-fn')
  }

  // incompatible node.js
  if (!res || !res.runInAsyncScope) {
    return fn
  }

  // return bound function
  return res.runInAsyncScope.bind(res, fn, null)
}
```

这里的逻辑是通过`asyncHooks.AsyncResource`创建了一个异步资源（名称优先取用户的回调函数名称），并让用户的回调函数`fn`在这个异步资源中执行。前提是当前`Node.js`环境支持`async_hooks`，该模块是`Node.js8`新增的内容。值得注意的是：**即使做这些操作，对实际的用户回调执行是不影响的**。

这里为什么要创建异步资源并让用户的回调在该异步资源中执行呢？作用是在使用`async_hooks`追踪监控异步的时候可以方便的找到`onFinished`的调用链，比如下面使用案例：

```js
const fs = require('fs');
const http = require('http');
const AsyncHooks = require('async_hooks');
const onFinished = require('on-finished');

// 创建async_hooks实例
const asyncHooks = AsyncHooks.createHook({
  init(asyncId, type, tid) {
    // 在终端输出：
    // 异步上下文资源id，异步类型，调用异步资源的上下文id
    fs.writeSync(process.stdout.fd, `${type}: ${asyncId}, tid: ${tid} \n`)
  }
});
// 启用异步追踪
asyncHooks.enable();

// 初始化http服务
const server = http.createServer((req, res) => {
  // 注意回调函数是个具名函数
  onFinished(res, function resOnFinished() {
    console.log('end')
  });
  res.end('a str');
});

server.listen(3000, () => {
  console.log('server running at port 3000.');
});
```

我们请求之后可以在终端看到非常清晰的调用链路，而且因为有了我们根据`回调函数名`创建的异步上下文，非常容易定位位置和区分，如下图所示：

![image](https://note.youdao.com/yws/res/21661/92D47AF6331246918CA9C15F5C22BF91)

更多的是如果我们调用多次`onFinished`时，通过起不同的回调函数名称，那么在我们进行`async_hooks`追踪链路时非常容易区分。

最后提一下，`Node.js`本身单线程异步`I/O`的特性，很多逻辑都是异步的，但是异步也有很多问题，比如异步逻辑`追踪/控制/调试`难等。`async_hooks`的出现就是为了解决该问题，它提供了为所有异步逻辑添加`init/before/after/destroy`等钩子方便使用这控制异步的逻辑。对`async_hooks`感兴趣的小伙伴可以学习一下。



