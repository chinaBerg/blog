# send源码分析

> version 1.0.0-beta.1
> 愣锤 2022/03/07

[send](https://github.com/pillarjs/send#readme)是一个用于从文件系统以流的方式读取文件作为http响应结果的库。

下面演示一个对于所有http请求都返回根目录下的static/index.html文件资源的例子：

```js
const http = require('http');
const path = require('path');
const send = require('send')

// 初始化一个http服务
const server = http.createServer(function onRequest (req, res) {
  send(req, './index.html', {
    // 指定返回资源的根路径
    root: path.join(process.cwd(), 'static'),
  }).pipe(res);
});

server.listen(3000, () => {
  console.log('server is running at port 3000.');
});
```

### 源码分析

`send`库对外暴露一个`send`方法，该方法内初始化一个`SendStream`类，`SendStream`类继承`Stream`模块，同时实现`pipe`等实例方法。

```js
var path = require('path')
var Stream = require('stream')

/**
 * Path 模块一些方法的快捷引用
 */
var extname = path.extname
var join = path.join
var normalize = path.normalize
var resolve = path.resolve
var sep = path.sep

/**
 * 对外暴露的send函数
 * 没有直接暴露SendStream类的原因主要是去掉new的调用
 * @public
 */

module.exports = send

/**
 * 对外暴露的send方法，接收req请求，返回`SendStream`得到的文件流
 * @param {object} req http模块等req请求
 * @param {string} path 要匹配的静态资源路径
 * @param {object} [options] 可选参数
 */
function send (req, path, options) {
  return new SendStream(req, path, options)
}

function SendStream (req, path, options) {
  // ES5方式继承Stream模块
  Stream.call(this)

  var opts = options || {}

  this.options = opts
  this.path = path
  this.req = req

  // ... 其他一些初始化参数赋值的操作

  this._root = opts.root
    ? resolve(opts.root)
    : null
}

SendStream.prototype.pipe = function pipe (res) {}

// ES5方式继承Stream模块
util.inherits(SendStream, Stream)
```

在使用`send`库时，主要是通过调用`send`函数得到的实例的`pipe`方法，下面看下`pipe`的实现：

```js
SendStream.prototype.pipe = function pipe (res) {
  // 根路径
  var root = this._root

  // 保存res引用
  this.res = res

  // 对path进行decodeURIComponent解码
  var path = decode(this.path)
  // 解码失败直接返回res
  if (path === -1) {
    this.error(400)
    return res
  }

  // null byte(s)
  if (~path.indexOf('\0')) {
    this.error(400)
    return res
  }

  var parts
  if (root !== null) {
    // 将path规范化成./path
    if (path) {
      path = normalize('.' + sep + path)
    }

    // malicious path
    if (UP_PATH_REGEXP.test(path)) {
      debug('malicious path "%s"', path)
      this.error(403)
      return res
    }

    // 根据路径符合分割path
    parts = path.split(sep)

    // join / normalize from optional root dir
    // 将根路径拼接起来
    path = normalize(join(root, path))
  } else {
    // ".." is malicious without "root"
    if (UP_PATH_REGEXP.test(path)) {
      debug('malicious path "%s"', path)
      this.error(403)
      return res
    }

    // normalize用于规范化path，可以解析..或.等路径符合
    // sep提供特定于平台的路径片段分隔符
    // parts得到的是根据路径分隔符分割到的字符串数组
    parts = normalize(path).split(sep)

    // resolve the path
    // 系列化为绝对路径
    path = resolve(path)
  }

  // 处理点开通的文件，例如.cache
  if (containsDotFile(parts)) {
    debug('%s dotfile "%s"', this._dotfiles, path)
    switch (this._dotfiles) {
      case 'allow':
        break
      case 'deny':
        this.error(403)
        return res
      case 'ignore':
      default:
        this.error(404)
        return res
    }
  }

  // 处理pathname以"/"结尾的情况
  if (this._index.length && this.hasTrailingSlash()) {
    this.sendIndex(path)
    return res
  }

  this.sendFile(path)
  return res
}
```

- `pipe`方法主要作用是根据用户参数格式化`path`参数
- 根据path参数的值：
    - 以`/`结尾则调用`sendIndex`方法
    - 否则调用`sendFile`方法处理

`sendIndex`方法的主要逻辑是根据要匹配的`path`参数为`/`结尾时，尝试匹配`path/index.html`或以用户设置的`index`值优先。

```js
/**
 * 尝试从path转换成index值
 * Eg：path/ => path/index.html
 * @param {String} path
 * @api private
 */
SendStream.prototype.sendIndex = function sendIndex (path) {
  var i = -1
  var self = this

  function next (err) {
    // 如果用户设置的所有index值都没有匹配到，则抛出错误
    // index默认值是["index.html"]，即当访问path/时，指定到path/index.html
    if (++i >= self._index.length) {
      if (err) return self.onStatError(err)
      return self.error(404)
    }

    // path拼接index
    var p = join(path, self._index[i])

    debug('stat "%s"', p)
    // 判断新的index路径是否存在
    fs.stat(p, function (err, stat) {
      // 不存在则继续尝试下一个index
      if (err) return next(err)
      // 如果新的index路径是文件夹，继续尝试下一个index
      if (stat.isDirectory()) return next()
      // 如果是文件，则emit file事件
      self.emit('file', p, stat)
      // 调用send返回流数据
      self.send(p, stat)
    })
  }

  next()
}
```

sendIndex内部在尝试拼接path/index后，如果资源存在，则判断是文件夹还是文件资源：

- 文件夹资源则继续根据`index`值尝试拼接`path`路径
- 若是文件资源，则调用实例的`send`方法继续处理资源，同时`emit`一个`file`事件

下面看send方法内部的处理：

```js
SendStream.prototype.send = function send (path, stat) {
  var len = stat.size
  var options = this.options
  var opts = {}
  var res = this.res
  var req = this.req
  var ranges = req.headers.range
  var offset = options.start || 0

  // 无法发送的抛错处理
  if (res.headersSent) {
    // impossible to send now
    this.headersAlreadySent()
    return
  }

  debug('pipe "%s"', path)

  // 设置res的headers请求头相关字段
  this.setHeader(path, stat)

  // 设置请求头的Content-Type值
  this.type(path)

  // conditional GET support
  if (this.isConditionalGET()) {
    if (this.isPreconditionFailure()) {
      this.error(412)
      return
    }

    if (this.isCachable() && this.isFresh()) {
      this.notModified()
      return
    }
  }

  // adjust len to start/end options
  len = Math.max(0, len - offset)
  if (options.end !== undefined) {
    var bytes = options.end - offset + 1
    if (len > bytes) len = bytes
  }

  // Range support
  if (this._acceptRanges && BYTES_RANGE_REGEXP.test(ranges)) {
    // parse
    ranges = parseRange(len, ranges, {
      combine: true
    })

    // If-Range support
    if (!this.isRangeFresh()) {
      debug('range stale')
      ranges = -2
    }

    // unsatisfiable
    if (ranges === -1) {
      debug('range unsatisfiable')

      // Content-Range
      res.setHeader('Content-Range', contentRange('bytes', len))

      // 416 Requested Range Not Satisfiable
      return this.error(416, {
        headers: { 'Content-Range': res.getHeader('Content-Range') }
      })
    }

    // valid (syntactically invalid/multiple ranges are treated as a regular response)
    if (ranges !== -2 && ranges.length === 1) {
      debug('range %j', ranges)

      // Content-Range
      res.statusCode = 206
      res.setHeader('Content-Range', contentRange('bytes', len, ranges[0]))

      // adjust for requested range
      offset += ranges[0].start
      len = ranges[0].end - ranges[0].start + 1
    }
  }

  // clone options
  for (var prop in options) {
    opts[prop] = options[prop]
  }

  // set read options
  opts.start = offset
  opts.end = Math.max(offset, offset + len - 1)

  // 设置Content-Length
  res.setHeader('Content-Length', len)

  /**
   * 支持HEAD请求
   * HEAD请求也是用于请求资源，但是服务器不会返回请求资源的实体数据，只会传回响应头，也就是元信息
   */
  if (req.method === 'HEAD') {
    res.end()
    return
  }

  // 调用stream方法返回文件流数据
  this.stream(path, opts)
}
```

send方法内设置了返回资源的请求头相关字段：

- 根据用户参数设置`Cache-Control`、`Last-Modified`等
- 设置`Content-Type`字段，如果返回的资源已经包含了`Content-Type`则使用原有的，否则根据文件后缀名，通过mime库获取`Content-Type`

同时，`send`内部支持了`HEAD`请求，对应`HEAD`请求则只返回请求头相关信息，不返回资源的实体数据：

```js
/**
 * 支持HEAD请求
 * HEAD请求也是用于请求资源，但是服务器不会返回请求资源的实体数据，只会传回响应头，也就是元信息
 */
if (req.method === 'HEAD') {
  res.end()
  return
}
```

最后调用`stream`方法返回文件流数据：

```js
SendStream.prototype.stream = function stream (path, options) {
  // TODO: this is all lame, refactor meeee
  var finished = false
  var self = this
  var res = this.res

  /**
   * 创建一个可读流
   * emit一个stream事件，让外部可以在该事件钩子中继续处理stream
   */
  var stream = fs.createReadStream(path, options)
  this.emit('stream', stream)
  // 将流传递给res响应
  stream.pipe(res)

  // response finished, done with the fd
  // 响应结束，销毁流
  onFinished(res, function onfinished () {
    finished = true
    destroy(stream)
  })

  // 错误处理，销毁流
  stream.on('error', function onerror (err) {
    // request already finished
    if (finished) return

    // clean up stream
    finished = true
    destroy(stream)

    // error
    self.onStatError(err)
  })

  // 流读取结束
  stream.on('end', function onend () {
    self.emit('end')
  })
}
```

stream内部的实现才是本库的核心部分，首先通过fs模块创建一个可读流，同时对外暴露一个stream事件，让外部有机会在创建流后做一些处理逻辑：

```js
/**
 * 创建一个可读流
 * emit一个stream事件，让外部可以在该事件钩子中继续处理stream
 */
var stream = fs.createReadStream(path, options)
this.emit('stream', stream)
// 将流传递给res响应
stream.pipe(res)
```

最后在流出错或者响应结束时销毁流，在流读取结束时暴露一个`end`事件。


下面我们回到`pipe`方法内部，对于`path`不是`/`结尾的调用`sendFile`逻辑:

```js
SendStream.prototype.pipe = function pipe (res) {
  // ... 省略前面的代码
    
  // 处理pathname以"/"结尾的情况
  if (this._index.length && this.hasTrailingSlash()) {
    this.sendIndex(path)
    return res
  }

  this.sendFile(path)
  return res
}
```

下面看下`sendFile`逻辑:

```js
SendStream.prototype.sendFile = function sendFile (path) {
  var i = 0
  var self = this

  debug('stat "%s"', path)
  fs.stat(path, function onstat (err, stat) {
    // 如果文件资源不存在，且没有文件后缀名，则调用next方法拼接.html等后缀名继续尝试尝试
    if (err && err.code === 'ENOENT' && !extname(path) && path[path.length - 1] !== sep) {
      // not found, check extensions
      return next(err)
    }
    if (err) return self.onStatError(err)
    // 如果是文件夹，则重定向
    if (stat.isDirectory()) return self.redirect(path)
    // 如果是文件，则emit file事件，
    self.emit('file', path, stat)
    // 利用send方法返回流
    self.send(path, stat)
  })

  function next (err) {
    if (self._extensions.length <= i) {
      return err
        ? self.onStatError(err)
        : self.error(404)
    }

    var p = path + '.' + self._extensions[i++]

    debug('stat "%s"', p)
    fs.stat(p, function (err, stat) {
      if (err) return next(err)
      if (stat.isDirectory()) return next()
      self.emit('file', p, stat)
      self.send(p, stat)
    })
  }
}
```

这时的主要做法是判断`path`对应的资源是否存在：

- 如果不存在，且不存在文件后缀名，则尝试拼接后缀名再查看资源是否存在。
- 如果资源存在，则判断是文件夹还是文件，是文件夹则继续尝试匹配，是文件则调用`send`做后续处理，逻辑同之前的`send`


### 总结

`send`库的核心还是在于根据`path`路径映射的资源，通过`fs.createReadStream`进行读取流，然后通过`stream.pipe(res)`进行消费流。

另一个比较有意思的点就是实现了`HEAD`请求，只返回请求头，不返沪请求的实体数据。