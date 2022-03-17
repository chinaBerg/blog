# serve-static源码分析

> version 1.14.2
> 愣锤 2022/03/17

[serve-static](https://github.com/expressjs/serve-static)是在Node提供静态文件托管服务的中间件，背后是对`send`库的中间件封装。该库根据请求的`req.url`映射到对应的资源，当资源不存在时不会直接响应`404`，而是调用`next()`移动到下一个中间件。

# 基本使用

- 在http服务中使用静态文件托管服务

```js
const http = require('http');
const finalhandler = require('finalhandler');
const serveStatic = require('serve-static');

const root = __dirname + '/static';
const fileServer = serveStatic(root, {
  index: ['index.html', 'index.htm'],
});

const server = http.createServer((req, res) => {
  const done = finalhandler(req, res);
  fileServer(req, res, done);
});

server.listen(3200, () => {
  console.log('[file server] running at port 3200.');
});
```

运行服务后通过curl测试，测试前现在脚本的同级目录创建`static/index.html`文件：

```js
# curl请求，可以正常拿到html的内容
curl http://localhost:3200
```

- 演示在http服务中提供资源下载的例子

```js
const http = require('http');
const contentDisposition = require('content-disposition');
const finalhandler = require('finalhandler');
const serveStatic = require('serve-static');

// 初始化文件下载服务
const root = __dirname + '/static';
const fileServer = serveStatic(root, {
  index: false,
  setHeaders: setHeaders,
});

// 设置响应头来强制下载
function setHeaders(res, path) {
  res.setHeader('Content-Disposition', contentDisposition(path));
}

// 初始化http服务
const server = http.createServer((req, res) => {
  const done = finalhandler(req, res);
  fileServer(req, res, done);
});

server.listen(3200, () => {
  console.log('[file server] running at port 3200.');
});
```

通过curl测试下载功能，可以看到文件被正常下载到本地：

```js
# 将服务的static/index.html资源下载到本地的download.html
curl -o ./download.html http://localhost:3200/index.html
```

- 作为express中间件使用

```js
const express = require('express');
const serveStatic = require('serve-static');

const root = __dirname + '/static';
const fileServer = serveStatic(root, {
  index: ['index.html', 'index.htm'],
});

const app = new express();

app.use(fileServer);

app.listen(3200, () => {
  console.log('[koa file server] running at port 3200.');
});
```

# 源码分析

`send`库的实现仅在根目录下的`index.js`文件，核心结构就是导出了一个函数：

```js
// 根目录下的index.js文件
'use strict'

var send = require('send')

/**
 * Module exports.
 * @public
 */

module.exports = serveStatic
module.exports.mime = send.mime

function serveStatic (root, options) {}
```

下面我们就来看serveStatic函数的实现：

```js
function serveStatic (root, options) {
  // 要求必须制定根路径
  if (!root) {
    throw new TypeError('root path required')
  }

  // 根路径值的类型检查
  if (typeof root !== 'string') {
    throw new TypeError('root path must be a string')
  }

  // copy用户传递的参数
  var opts = Object.create(options || null)

  // fall-though 默认值为true
  var fallthrough = opts.fallthrough !== false

  // default redirect 默认值为 true
  var redirect = opts.redirect !== false

  // headers listener
  var setHeaders = opts.setHeaders

  if (setHeaders && typeof setHeaders !== 'function') {
    throw new TypeError('option setHeaders must be function')
  }

  // setup options for send
  opts.maxage = opts.maxage || opts.maxAge || 0
  opts.root = resolve(root)

  // send库directory事件的处理函数
  // 用于处理路径时文件选项时是否进一步重定向
  // 该事件的作用是让用户自定义文件夹路径跳转逻辑
  var onDirectory = redirect
    ? createRedirectDirectoryListener()
    : createNotFoundDirectoryListener()

  // 返回中间件函数
  return function serveStatic (req, res, next) {}
  // ....
}
```

该函数首先是初始化一些默认参数，然后返回一个中间件格式的函数。这里没什么好说的，但是有一个小点提一下，就是**布尔参数默认值的初始化技巧**：

```js
// 设置默认值为true
// 只要用户没显示声明参数为false，则默认值为true
var fallthrough = opts.fallthrough !== false

// 设置默认值为false
// 只要用户没显示声明参数为true，则默认值为false
var redirect = opts.redirect === true
```

接下来我们详细的看这个返回的中间件函数：

```js
// 返回中间件
return function serveStatic (req, res, next) {
  // 处理请求不是GET或HEAD的场景
  if (req.method !== 'GET' && req.method !== 'HEAD') {
    // 如果fallthrough为true，则直接next执行下一个中间件
    if (fallthrough) {
      return next()
    }

    // 否则直接响应405状态告知只允许GET或HEAD请求
    res.statusCode = 405
    res.setHeader('Allow', 'GET, HEAD')
    res.setHeader('Content-Length', '0')
    res.end()
    return
  }

  var forwardError = !fallthrough
  var originalUrl = parseUrl.original(req)
  // 获取pathname路径
  var path = parseUrl(req).pathname

  // make sure redirect occurs at mount
  if (path === '/' && originalUrl.pathname.substr(-1) !== '/') {
    path = ''
  }

  // 实例化send得到流
  var stream = send(req, path, opts)

  // 添加文件夹资源的处理逻辑
  stream.on('directory', onDirectory)

  // 如果用户设置了setHeaders，则自定义响应头函数
  if (setHeaders) {
    stream.on('headers', setHeaders)
  }

  // add file listener for fallthrough
  if (fallthrough) {
    stream.on('file', function onFile () {
      // 如果是读取的文件，则将该变量设置为ture
      // 该变量用于在文件读取报错是否next error
      forwardError = true
    })
  }

  // 监听流出错的钩子
  stream.on('error', function error (err) {
    // 如果用户设置了允许next error逻辑，或者错误状态码大于等于500
    // 则直接next error
    if (forwardError || !(err.statusCode < 500)) {
      next(err)
      return
    }

    next()
  })

  // 将流连接到res流上，即http返回流数据
  stream.pipe(res)
}
```

该部分主要逻辑：

- 处理非`GET | HEAD`的请求
    - 根据配置参数决定是`next()`还是响应`405`错误
- 实例化`send`得到`send`实例`stream`流
- 添加`send`实例的`directory`事件
    - 根据配置参数决定重定向或响应`404`错误
    - `send`库默认的`directory`逻辑是响应`403`错误
- 添加send实例的headers事件让用户可以自定义响应头
- 添加send实例的错误处理事件
    - 如果是文件流出错则直接`next(err)`
    - 如果是错误状态码大于等于500直接`next(err)`
    - 否则根据配置参数决定是`next(err)`还是`next`
- `stream.pipe(res)`返回响应的流数据

最后看下`createRedirectDirectoryListener`的重定向逻辑：

```js
/**
 * Create a directory listener that performs a redirect.
 * 注意该方法虽然是send库directory事件回调
 * 其主要作用就是自定义directory逻辑，即自定义send中的redirectory实现
 * @private
 */

function createRedirectDirectoryListener () {
  return function redirect (res) {
    /**
     * 调用send库内部的hasTrailingSlash方法，
     * 判断是否‘/’结尾的路径。
     * 且没有匹配到资源时404
     */
    if (this.hasTrailingSlash()) {
      this.error(404)
      return
    }

    // 重定向逻辑，重定向到path/，和send库的实现基本一样
    // get original URL
    var originalUrl = parseUrl.original(this.req)

    // append trailing slash
    originalUrl.path = null
    originalUrl.pathname = collapseLeadingSlashes(originalUrl.pathname + '/')

    // reformat the URL
    var loc = encodeUrl(url.format(originalUrl))
    var doc = createHtmlDocument('Redirecting', 'Redirecting to <a href="' + escapeHtml(loc) + '">' +
      escapeHtml(loc) + '</a>')

    // 设置重定向状态码
    res.statusCode = 301
    // 设置重定向的相关请求头
    res.setHeader('Content-Type', 'text/html; charset=UTF-8')
    res.setHeader('Content-Length', Buffer.byteLength(doc))
    res.setHeader('Content-Security-Policy', "default-src 'none'")
    res.setHeader('X-Content-Type-Options', 'nosniff')
    // 重定向到loc的地址
    res.setHeader('Location', loc)
    res.end(doc)
  }
}
```

这个重定向的核心逻辑就是获取要重定向的地址`path/`，然后通过设置响应头进行重定向：

```js
// 设置重定向状态码
res.statusCode = 301
// 设置重定向的相关请求头
res.setHeader('Content-Type', 'text/html; charset=UTF-8')
res.setHeader('Content-Length', Buffer.byteLength(doc))
res.setHeader('Content-Security-Policy', "default-src 'none'")
res.setHeader('X-Content-Type-Options', 'nosniff')
// 重定向到loc的地址
res.setHeader('Location', loc)
res.end(doc)
```