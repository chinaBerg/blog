# finalhandler源码分析

> version 1.1.2
> 愣锤 2022/03/22

[finalhandler](https://github.com/pillarjs/finalhandler#readme)是在NodeJs中作为http服务响应的最后一步的处理逻辑。


- 对所有http请求都响应404错误

```js
const http = require('http');
const finalhandler = require('finalhandler');

const server = http.createServer((req, res) => {
  const done = finalhandler(req, res);
  done();
});

server.listen(3000, () => {
  console.log('[server] running at port 3000.');
});
```

- 对http请求返回stream流，并且出错时响应500错误

```js
const http = require('http');
const fs = require('fs');
const finalhandler = require('../index');

// 出错时打印错误日志
function logger(err) {
  console.error(err.stack || err.toString());
}

// 初始化http服务
const server = http.createServer((req, res) => {
  const done = finalhandler(req, res, {
    onerror: logger,
  });

  // 创建一个可读流，并且读取一个不存在的文件
  const stream = fs.createReadStream('not/exist/path/demo.txt');
  stream.on('error', (err) => {
    // 响应500错误
    done(err);
  });
  stream.pipe(res);
});

server.listen(3000, () => {
  console.log('[server] running at port 3000.');
});
```

### 源码解析

源码主体就是导出一个`finalhandler`函数，该函数主要处理参数的初始化以及返回一个函数，返回的函数接受一个从外部传入的err对象。

```js
/**
 * Module exports.
 * @public
 */

module.exports = finalhandler

/**
 * 创建一个函数用于处理response的最后一步逻辑
 *
 * @param {Request} req
 * @param {Response} res
 * @param {Object} [options]
 * @return {Function}
 * @public
 */

function finalhandler (req, res, options) {
    // 参数处理
    // 省略....
    
    // 返回一个函数
    return function (err) {}
}
```

看下完整的`finalhandler`逻辑，参数处理部分主要是获取`env`环境变量，错误发生时的回调函数。

```js
function finalhandler (req, res, options) {
  var opts = options || {}

  // 获取环境变量
  var env = opts.env || process.env.NODE_ENV || 'development'

  // 获取出错时的回调函数
  var onerror = opts.onerror

  // 返回一个函数，也是一个闭包
  return function (err) {
    var headers
    var msg
    var status

    // ignore 404 on in-flight response
    // header头已经被发出去了，则无法响应404
    // 因为在此之前已经有类似writeHead的操作了
    if (!err && headersSent(res)) {
      debug('cannot 404 after headers sent')
      return
    }

    // unhandled error
    if (err) {
      // 从error对象上获取错误状态码
      status = getErrorStatusCode(err)

      if (status === undefined) {
        // 回退到response对象上的状态码
        status = getResponseStatusCode(res)
      } else {
        // 从error对象上获取headers
        headers = getErrorHeaders(err)
      }

      // 生成错误信息
      msg = getErrorMessage(err, status, env)
    } else {
      // 定义404状态码和错误信息
      status = 404
      msg = 'Cannot ' + req.method + ' ' + encodeUrl(getResourceName(req))
    }

    debug('default %s', status)

    // 当err存在时且用户自定义了err的回调，则触发回调
    if (err && onerror) {
      defer(onerror, err, req, res)
    }

    // 实际已经不能响应，则直接销毁req流
    if (headersSent(res)) {
      debug('cannot %d after headers sent', status)
      req.socket.destroy()
      return
    }

    // 调用send方法处理响应发送逻辑
    send(req, res, status, headers, msg)
  }
}
```

返回的函数也是一个闭包，它的主要逻辑是：

- 通过`headersSent(res)`判断如果`header`已经被发送了，则`log`一个信息，不做任何处理
- 存在`err`对象时，例如我们调用`if (err) done(err)`的做法时传入了`err`对象
    - 尝试获取错误状态码
        - 优先从错误对象上获取，取到了则再尝试从`err`对象上获取`headers`。这个`headers`如果取到了则该库最后响应时会携带上。
        - `err`对象上取不到再尝试从`response`对象上获取
        - 还取不到则错误状态码默认是`500`
    - 尝试获取错误信息
- `err`对象不存在时，例如我们调用`done()`
    - 错误状态码设置为`404`
    - 错误信息直接设置为例如`Cannot Get /path/your/req`
- 即使`err`对象存在，但是`header`已经发出去了，也直接销毁`req`流，不再做其他处理
- 如果用户设置了错误处理的`callback`则调用`defer(onerror, err, req, res)`函数处理回调函数的回调
- 否则最后调用`send`函数发送响应逻辑


看下上面的几个工具方法实现：

- `headersSent`函数判断header头是否已经被发出去
- 
```js
/**
 * 判断header头是否已经被发出去
 * @param {object} res
 * @returns {boolean}
 * @private
 */
function headersSent (res) {
  return typeof res.headersSent !== 'boolean'
    ? Boolean(res._header)
    : res.headersSent
}
```

判断逻辑就是`res`对象上是否存在`headersSent`值，该字段值会在`header`被发送之后设为`true`，可以看个例子：

```js
const http = require('http');

const server = http.createServer((req, res) => {
  // false
  console.log('berfor headersSent', res.headersSent);
  res.writeHead(200);
  // true
  console.log('after headersSent', res.headersSent);
  res.end();
});

server.listen(3200);
```

- 获取错误状态码的函数实现

```js
/**
 * 从Error对象上获取错误状态码
 *
 * @param {Error} err
 * @return {number}
 * @private
 */

function getErrorStatusCode (err) {
  // check err.status
  if (typeof err.status === 'number' && err.status >= 400 && err.status < 600) {
    return err.status
  }

  // check err.statusCode
  if (typeof err.statusCode === 'number' && err.statusCode >= 400 && err.statusCode < 600) {
    return err.statusCode
  }

  return undefined
}

/**
 * 从response对象上获取状态码
 *
 * @param {OutgoingMessage} res
 * @return {number}
 * @private
 */

function getResponseStatusCode (res) {
  var status = res.statusCode

  // 如果response上不存在状态码，或者状态码的值不是在400-599范围，
  // 则状态码默认是500
  // default status code to 500 if outside valid range
  if (typeof status !== 'number' || status < 400 || status > 599) {
    status = 500
  }

  return status
}
```

- 从err对象上获取headers的逻辑实现

这里的逻辑就是判断err对象是否存在headers字段，存在的话就拷贝一份对象返回。

```js
/**
 * 从Error对象上获取headers
 *
 * @param {Error} err
 * @return {object}
 * @private
 */

function getErrorHeaders (err) {
  // err上不存在headers或格式不对则返回undefined
  if (!err.headers || typeof err.headers !== 'object') {
    return undefined
  }

  // 拷贝headers上的所有key/value
  var headers = Object.create(null)
  var keys = Object.keys(err.headers)

  for (var i = 0; i < keys.length; i++) {
    var key = keys[i]
    headers[key] = err.headers[key]
  }

  return headers
}
```

- 获取错误信息的实现

这里的错误信息获取逻辑，首先要判断是不是生成环境，因为**考虑到安全的问题，生产环境是不能暴露具体的错误堆栈等信息的**，这样不安全。这点非常重要哦！！！

因此该函数也是只在非生产环境先尝试获取错误的堆栈信息，如果获取不到则再尝试通过`err.toString()`方法获取错误信息，还获取不到的话就通过[statuses](https://github.com/jshttp/statuses)库获取通用的错误信息。比如`500`会返回`Internal Server Error`，`501`会返回`Not Implemented`, `502`会返回`Bad Gateway`等。

在生产环境也仅通过[statuses](https://github.com/jshttp/statuses)库获取通用的错误信息。

```js
/**
 * 从Error对象上获取错误信息，获取不到则根据status获取通用错误信息
 * @param {Error} err
 * @param {number} status
 * @param {string} env
 * @return {string}
 * @private
 */
function getErrorMessage (err, status, env) {
  var msg

  // 非生产环境，尽量获取具体的msg信息
  // 而生成环境则不会暴露具体的错误堆栈等信息，因为不安全
  if (env !== 'production') {
    // 优先使用stack堆栈信息，因为堆栈信息里面包含了message信息
    msg = err.stack

    // 不存在堆栈信息则尝试调用toString得到信息
    if (!msg && typeof err.toString === 'function') {
      msg = err.toString()
    }
  }

  // 生产环境以及开发环境兜底的方法则使用statuses库提供的通用错误信息
  // 比如 500 -> "Internal Server Error"
  return msg || statuses[status]
}
```

- defer的实现

```js
/**
 * 等待当前执行结束后再执行
 *
 * - process.nextTick()属于idle观察者
 * - setImmediate()属于check观察者
 * - 每一轮循环检查顺序：idle观察者 先于 I/O观察者 先于 check观察者
 */
var defer = typeof setImmediate === 'function'
  ? setImmediate
  : function (fn) { process.nextTick(fn.bind.apply(fn, arguments)) }
var isFinished = onFinished.isFinished
```

`defer`的实现要注意`process.nextTick` 是将异步回调放到当前帧的末尾、`io`回调之前，如果`nextTick`过多，会导致`io`回调不断延后,最后`callback`堆积太多；而`setImmediate` 是将异步回调放到下一帧,不影响`io`回调,不会造成`callback` 堆积。因此`defer`优先使用`setImmediate`方法。

接下来看下`send`函数真正的发送响应的处理逻辑

```js
/**
 * 发送response响应
 *
 * @param {IncomingMessage} req 请求
 * @param {OutgoingMessage} res 响应
 * @param {number} status 响应状态码
 * @param {object} headers
 * @param {string} message
 * @private
 */
function send (req, res, status, headers, message) {
  function write () {
    // response的html
    var body = createHtmlDocument(message)

    // 设置响应的状态码和状态码信息
    res.statusCode = status
    res.statusMessage = statuses[status]

    // 如果前述步骤中err对象上存在了headers
    // 则通过此方法将headers对象上的值依次设置到响应头上
    setHeaders(res, headers)

    // 防止XSS攻击的CSP策略，禁止加载任何脚本
    res.setHeader('Content-Security-Policy', "default-src 'none'")
    // 告知接收者禁止嗅探MIME类型，即服务端确认自己的MIME设置无误
    res.setHeader('X-Content-Type-Options', 'nosniff')

    // 设置响应头的Content类型、长度
    res.setHeader('Content-Type', 'text/html; charset=utf-8')
    res.setHeader('Content-Length', Buffer.byteLength(body, 'utf8'))

    // 支持HEAD请求
    if (req.method === 'HEAD') {
      res.end()
      return
    }

    // 设置响应数据
    res.end(body, 'utf8')
  }

  // 当请求出错、关闭、完成时触发
  if (isFinished(req)) {
    write()
    return
  }

  // 断开req上的所有管道连接，
  // 背后实质是req.unpipe()调用，
  // unpipe()没有传递参数是断开所有管道，有参数是断开指定管道
  unpipe(req)

  // 等请求结束后调用write函数发送response
  onFinished(req, write)
  // 将req恢复到流动状态
  // 因为上述调用unpipe之后req就变成了暂停状态
  req.resume()
}
```

`send`的处理逻辑份两种情况：

- `req`已经结束（比如请求关闭、出错、完成）
    - 创建要响应的响应体，即html内容
    - 设置响应码和响应码的信息
    - 如果之前`err`对象上存在`headers`则依次设置响应对象上的响应头相关字段
    - 设置安全相关的响应头
    - 设置`Conetent`相关的响应头
    - 如果是`HEAD`请求则只返回响应头
    - 发送响应数据
    - 处理结束
- `req`请求未结束
    - 调用`unpipe`库结束来断开`req`上的所有管道，注意此举会将`req`设置为暂停状态。
    - 等待请求结束后，再调用上去req请求结束的处理逻辑
    - 最后将req重新设置为流动状态

这里需要注意的是，`req`请求未结束时调用来`unpipe`库来终止`req`上的所有管道连接:

- `unpipe`库背后是调用的`req.unpipe()`
  - 调用`unpipe`方法时如果不传递参数则会断开所有管道连接
  - 调用`unpipe`方法时设置参数就是端口指定的管道连接。

- `req`是可读流，`req.unpipe()`背后会将可读流的状态变更为暂停状态，暂时停止事件的流动，注意不会停止数据的生成。因此在处理完成后又调用了`req.resume()`将流从暂停状态恢复到流动状态。

可参考[node stream unpipe方法](http://nodejs.cn/api/stream.html#readableunpipedestination)理解流相关内容。

最后看下返回的html生成逻辑：

```js
var DOUBLE_SPACE_REGEXP = /\x20{2}/g
var NEWLINE_REGEXP = /\n/g

/**
 * 创建响应的html
 *
 * @param {string} message
 * @private
 */
function createHtmlDocument (message) {
  // 考虑安全问题对message进行escapeHtml编码
  // 然后将换行符转换成<br>标签
  // 最后再处理多个空格能正确显示的问题
  var body = escapeHtml(message)
    .replace(NEWLINE_REGEXP, '<br>')
    .replace(DOUBLE_SPACE_REGEXP, ' &nbsp;')

  return '<!DOCTYPE html>\n' +
    '<html lang="en">\n' +
    '<head>\n' +
    '<meta charset="utf-8">\n' +
    '<title>Error</title>\n' +
    '</head>\n' +
    '<body>\n' +
    '<pre>' + body + '</pre>\n' +
    '</body>\n' +
    '</html>\n'
}
```

这里有个小注意点是，这里的\r处理没必要，因为参数url中不包含换行的情况，另外就是如果要处理换行的问题，正则要考虑不同系统的换行符是不一样的：

```js
// old
var NEWLINE_REGEXP = /\n/g

// new
var NEWLINE_REGEXP = /\r|\n|\r\n/g
```

很重要的一个点是，因为这里的`message`信息包含了用户请求的`url`，该部分来自请求端，因此url中参数是不可信的，有可能包含`XSS`注入，所以要利用`escapeHtml`进行编码再处理。

### 参考

- [Content Security Policy 入门教程](http://www.ruanyifeng.com/blog/2016/09/csp.html) 阮一峰
- [X-Content-Type-Options](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/X-Content-Type-Options) MDN文档X-Content-Type-Optionss
- [X-Content-Type-Options](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/X-Content-Type-Options) MDN文档X-Content-Type-Options