# live-server源码解析

> version 1.1.2
> 愣锤 2022/02/22

[live-server](https://github.com/tapio/live-server#readme)是一个支持实时刷新功能的开发环境服务器。全局安装后可以作为命令行使用，例如：

```bash
# 终端输入
live-server
```

此时会使开启一个服务，并自动打开浏览器访问当前静态资源。同时监听当前目录下的静态文件内容发生变化，并实时刷新浏览器。


# 源码解析

从`package.json`文件中的`bin`字段可以看的，当前脚本的入口文件是`live-server.js`文件:

```json
{
  "bin": {
    "live-server": "./live-server.js"
  },
}
```

### 脚本入口`live-server.js`的实现

首先第一的代码是定义脚本的执行环境为`node`：

```js
#!/usr/bin/env node

var path = require('path');
var fs = require('fs');
var assign = require('object-assign');
var liveServer = require("./index");
```

紧接着都是从我们输入的node命令中，解析出命令相关参数，比如我们输入如下命令:

```bash
# 终端输入
live-server --port=3000 --host=http://localhost
```

具体解析逻辑如下：

```js
var opts = {
	host: process.env.IP,
	port: process.env.PORT,
	open: true,
	mount: [],
	proxy: [],
	middleware: [],
	logLevel: 2,
};

// 获取系统账户根目录文件夹 (等同于os.homedir()) 下的.live-server.json文件
var homeDir = process.env[(process.platform === 'win32') ? 'USERPROFILE' : 'HOME'];
var configPath = path.join(homeDir, '.live-server.json');

// 如果文件存在则读取文件json内容，进行参数的合并
if (fs.existsSync(configPath)) {
	var userConfig = fs.readFileSync(configPath, 'utf8');
	assign(opts, JSON.parse(userConfig));
	if (opts.ignorePattern) opts.ignorePattern = new RegExp(opts.ignorePattern);
}

/**
 * 解析终端命令参数
 * argv第一个参数是node的执行上下文，第二个参数是执行的脚本地址
 * 第三个参数及以后是命令参数
 */
for (var i = process.argv.length - 1; i >= 2; --i) {
  var arg = process.argv[i];
  // 解析端口号
  if (arg.indexOf("--port=") > -1) {
	var portString = arg.substring(7);
	var portNumber = parseInt(portString, 10);
	  if (portNumber === +portString) {
		opts.port = portNumber;
		process.argv.splice(i, 1);
	  }
  }
  // 解析host地址
  else if (arg.indexOf("--host=") > -1) {
    opts.host = arg.substring(7);
	process.argv.splice(i, 1);
  }
  // 省略其他else if代码
  // 该部分和上述一样，都是解析命令的其他参数
  // ......
}
```

- 首先判断用户根目录下有无`.live-server.json`文件，有则解析json内容作为默认配置
- 从`process.argv`读取所有的命令参数，与默认参数合并。该值是一个数组，数组第一项是node的执行上下文，第二个项是执行的脚本地址，后面的项都是后续的所有参数。
- 得到默认参数后，开始调用server真正的实现，并把参数传递进入，如下：

```js
liveServer.start(opts);
```

### liveServer的实现

`liveServer`的实现是在`index.js`内：

```js
// 读取injected.html的内容
// 内容实际为一段websocket代码，用于和本服务通信的
var INJECTED_CODE = fs.readFileSync(path.join(__dirname, "injected.html"), "utf8");
```

首先读取了该库根目录下`injected.html`文件的内容，并把内容赋值给一个变量等待后面使用。该文件内容就是存储的一段websocket代码，该代码的作用是在访问html等资源时要注入进去的代码，注入进去执行就可以在html文件运行时与服务进行ws连接和通信。先看下该`injected.html`的内容:

```html
<!-- Code injected by live-server -->
<script type="text/javascript">
	// <![CDATA[  <-- For SVG support
	if ('WebSocket' in window) {
		(function() {
			function refreshCSS() {
				var sheets = [].slice.call(document.getElementsByTagName("link"));
				var head = document.getElementsByTagName("head")[0];
				for (var i = 0; i < sheets.length; ++i) {
					var elem = sheets[i];
					head.removeChild(elem);
					var rel = elem.rel;
					if (elem.href && typeof rel != "string" || rel.length == 0 || rel.toLowerCase() == "stylesheet") {
						var url = elem.href.replace(/(&|\?)_cacheOverride=\d+/, '');
						elem.href = url + (url.indexOf('?') >= 0 ? '&' : '?') + '_cacheOverride=' + (new Date().valueOf());
					}
					head.appendChild(elem);
				}
			}
			var protocol = window.location.protocol === 'http:' ? 'ws://' : 'wss://';
			var address = protocol + window.location.host + window.location.pathname + '/ws';
			var socket = new WebSocket(address);
			socket.onmessage = function(msg) {
				if (msg.data == 'reload') window.location.reload();
				else if (msg.data == 'refreshcss') refreshCSS();
			};
			console.log('Live reload enabled.');
		})();
	}
	// ]]>
</script>
```

可以看到，该文件的内容就是一段js脚本，脚本主要做了如下事情：

- 根据url地址生成要连接的ws服务地址
- 初始化ws连接
- 监听ws服务推送的消息：
    - `reload`消息则刷新当前页面
    - `refreshcss`消息则做无感css刷新

无感css刷新的做法是遍历`head`标签中所有的样式表的`link`标签，然后逐个删除，然后重新插入，插入时生成一个新的时间戳字段用于去掉缓存效果。

接下来就是服务的主体实现:

```js
var LiveServer = {
	server: null,
	watcher: null,
	logLevel: 2
};

LiveServer.start = function(options) {}

LiveServer.shutdown = function() {}

module.exports = LiveServer;
```

就是定义一个对象，然后添加了start和shutdown两个方法。先看start的实现：

```js
// 根据options参数启动服务器
LiveServer.start = function(options) {
    options = options || {};
	// host地址
	var host = options.host || '0.0.0.0';
	// 端口号
	var port = options.port !== undefined ? options.port : 8080; // 0 means random
	// 脚本的入口，也就是要启动的资源服的入口
	var root = options.root || process.cwd();
	
	// 其他默认参数设置
	// .....
	
	// 初始化一个connect服务
	var app = connect();
	
	// ... 省略其他日志逻辑等与主体逻辑无关代码
	
	// 加载了一些中间件，例如
	// 添加cors跨域处理的中间件
	if (cors) {
		app.use(require("cors")({
			origin: true, // reflecting request origin
			credentials: true // allowing requests with credentials
		}));
	}
	
	// 加载静态文件托管服务中间件等
	app.use(staticServerHandler)
	
	var server, protocol;
	// 如果用户设置了https的配置
	if (https !== null) {
		var httpsConfig = https;
		// https参数是字符串时，则作为配置文件路径
		// 然后加载文件内容作为https请求的参数配置
		if (typeof https === "string") {
			httpsConfig = require(path.resolve(process.cwd(), https));
		}
		// 创建https服务器
		server = require(httpsModule).createServer(httpsConfig, app);
		protocol = "https";
	} else {
		// 否则默认使用http服务
		server = http.createServer(app);
		protocol = "http";
	}
}
```

- 首先进行各种参数的默认赋值
- 通过connect库实例化一个中间件服务
- 加载cors中间件、静态文件托管服务中间件等
- 根据用户参数选择创建http/https服务
- http/https服务加载中间件模型

核心代码如下：

```js
// 初始化一个connect中间件服务
var app = connect();

// 加载很多中间件
app.use(mideware1);
app.use(mideware2);
app.use(mideware3);

// 初始化http服务并使用中间件
var server = http.createServer(app);
```

#### 静态文件托管服务如何实现的？

```js
// 创建静态文件托管服务的中间件
var staticServerHandler = staticServer(root);

// 加载静态文件托管服务中间件等
app.use(staticServerHandler) // Custom static server
	.use(entryPoint(staticServerHandler, file))
	.use(serveIndex(root, { icons: true }));
```

由此可知具体的静态文件托管服务在`staticServer`中实现：

```js
// 静态文件托管服务
function staticServer(root) {
	var isFile = false;
	try { // For supporting mounting files instead of just directories
		// 判断指定的路径是否是文件
		isFile = fs.statSync(root).isFile();
	} catch (e) {
		if (e.code !== "ENOENT") throw e;
	}
	// 返回一个中间件
	return function(req, res, next) {
		// 仅处理GET和HEAD请求
		if (req.method !== "GET" && req.method !== "HEAD") return next();
		// 获取域名后面的路径部分，例如x.com/abc/def获取的是/abc/def
		// 如果isFile为true，直接为空
		var reqpath = isFile ? "" : url.parse(req.url).pathname;
		var hasNoOrigin = !req.headers.origin;
		var injectCandidates = [ new RegExp("</body>", "i"), new RegExp("</svg>"), new RegExp("</head>", "i")];
		var injectTag = null;

		// 利用send库把静态资源作为http的请求结果返回
		send(req, reqpath, { root: root })
			.on('error', error)
			.on('directory', directory)
			.on('file', file)
			.on('stream', inject)
			.pipe(res);
	};
}
```

- staticServer函数是一个创建函数，用于创建一个中间件函数
- 如果是非`GET | HEAD`请求则直接调用`next()`执行下一个中间件
- 利用send库把静态资源作为http请求的结果返回
    - 参数1是当前请求对象
    - 参数2是请求的资源路径
    - 参数3指定了请求资源的相对路径是当前脚本根路径或者用户可以指定root
- 利用send库进行监听事件
    - 请求资源是文件夹时，调用directory处理函数
    - 请求资源是文件时，调用file处理函数
    - 请求的流开始时，调用inject处理逻辑。该部分是最关键的，就是在该部分进行ws代码的注入

下面结束send各个事件的具体处理逻辑：

- 文件夹

当访问文件时，直接在后面拼接`/`，然后进行资源重定向

```js
// 当请求的是一个目录时的处理函数
function directory() {
	var pathname = url.parse(req.originalUrl).pathname;
	res.statusCode = 301;
	res.setHeader('Location', pathname + '/');
	res.end('Redirecting to ' + escape(pathname) + '/');
}
```

- 文件的处理函数

```js
// 当请求的是一个文件时的处理函数
function file(filepath /*, stat*/) {
	var x = path.extname(filepath).toLocaleLowerCase(), match,
			possibleExtensions = [ "", ".html", ".htm", ".xhtml", ".php", ".svg" ];
	if (hasNoOrigin && (possibleExtensions.indexOf(x) > -1)) {
		// TODO: Sync file read here is not nice, but we need to determine if the html should be injected or not
		var contents = fs.readFileSync(filepath, "utf8");
		for (var i = 0; i < injectCandidates.length; ++i) {
			match = injectCandidates[i].exec(contents);
			if (match) {
				injectTag = match[0];
				break;
			}
		}
	}
}
```

主要处理逻辑就是根据请求的文件路径的后缀名，判断是否是`.html | .htm | .xhtml`等文件类型，是的话则读取文件内容，通过正则查找文件内容是否包含`</body> | </head>`等字符，如果包含则说明该文件是可以进行注入ws代码的。这里只是给`injectTag`变量打个标记，真正的注入是在`stream`事件中实现。

- stream流开始事件

```js
// 在读取的目标文件流中注入socket脚本
function inject(stream) {
	if (injectTag) {
		// We need to modify the length given to browser
		var len = INJECTED_CODE.length + res.getHeader('Content-Length');
		res.setHeader('Content-Length', len);
        // 保存原pipe的引用
		var originalPipe = stream.pipe;
        // 重写原pipe方法
		stream.pipe = function(resp) {
            // 重新调用pipe方法，并且理由event-stream模块，对流的内容进行注入内容
            // 注入的内容为websocket通信的部分
			originalPipe.call(stream, es.replace(new RegExp(injectTag, "i"), INJECTED_CODE + injectTag)).pipe(resp);
		};
	}
}
```

处理逻辑主要通过重写pipe方法，然后对读取的流的内容进行替换，把`</body>`字符替换成要注入的`ws代码+</body>`,然后把res返回的响应头中的`Content-Length`值更新为替换后的内容长度。

### 服务监听和打开浏览器

```js
// Handle successful server
server.addListener('listening', function(/*e*/) {
	LiveServer.server = server;

	var address = server.address();
	// @see https://www.cnblogs.com/wenwei-blog/p/12114184.html
	var serveHost = address.address === "0.0.0.0" ? "127.0.0.1" : address.address;
	var openHost = host === "0.0.0.0" ? "127.0.0.1" : host;

	var serveURL = protocol + '://' + serveHost + ':' + address.port;
	// 打开的应用的url服务地址
	var openURL = protocol + '://' + openHost + ':' + address.port;

    // 省略日志的部分
    // ......


	// Launch browser
	// 利用open库唤起应用打开对于路径
	// 用户没有单独设置的情况下，是唤起浏览器
	if (openPath !== null) {
		if (typeof openPath === "object") {
			openPath.forEach(function(p) {
				open(openURL + p, {app: browser});
			});
		} else {
			open(openURL + openPath, {app: browser});
		}
	}
});

// Setup server to listen at port
// 监听端口号和host
server.listen(port, host);
```

- 通过listening事件监听http/https服务启动成功
- 拼接要到的资源的地址，即openPath
- 利用open库唤起应用，默认是唤起浏览器
- 监听端口号和host，开始运行服务

### 页面资源和服务通信连接

上面的分析中我们知道，我们会在流资源的http请求返回时注入ws代码，ws代码会自动尝试和我们的server服务开始连接。此时会触发server的握手事件，那么我们就可以在握手时初始化ws的服务，并建立客户端和ws的一对一连接。

这里之所以建立一对一的连接，主要是为了通信方便和数据互相隔离，同时也做到了由客户端发起连接时才初始化ws服务，因为有些资源是不会注入ws服务的，也就不需要连接。

立一对一的连接通过[faye-websocket](https://github.com/faye/faye-websocket-node)来实现。

```js
// WebSocket
var clients = [];
// 监听握手事件，每一个socket连接对应一个socket服务
// 利用faye-websocket库实现一对一的连接关系
server.addListener('upgrade', function(request, socket, head) {
	var ws = new WebSocket(request, socket, head);
	// ws初始化成功后，向连接的ws客户端发生一条消息
	// 虽然这条消息客户端没有使用
	ws.onopen = function() {
	    ws.send('connected');
	};

    // 监听到客户端关闭时，移除其缓存实例
	ws.onclose = function() {
		clients = clients.filter(function (x) {
			return x !== ws;
		});
	};

    // 缓存客户端实例
	clients.push(ws);
});
```

### 如何监听资源内容发生变化

在客户端和服务端建立了ws连接之后，那么就要监听静态资源内容是否发生了变化，我们需要在变化后通知客户端资源进行刷新：

```js
// Setup file watcher
LiveServer.watcher = chokidar.watch(watchPaths, {
	ignored: ignored,
	ignoreInitial: true
});

// 资源发生变化的处理函数
function handleChange(changePath) {
	var cssChange = path.extname(changePath) === ".css" && !noCssInject;

	clients.forEach(function(ws) {
		if (ws) {
			ws.send(cssChange ? 'refreshcss' : 'reload');
		}
	});
}

// 监听相关的变化事件
LiveServer.watcher
	.on("change", handleChange)
	.on("add", handleChange)
	.on("unlink", handleChange)
	.on("addDir", handleChange)
	.on("unlinkDir", handleChange)
	.on("error", function (err) {
		console.log("ERROR:".red, err);
	});
```

- 利用chokidar库进行文件内容变更的监听
- 文件内容变化话，判断是否是css文件发生变化
    - css发生变化，ws发生`refreshcss`消息
    - 否则ws发送`reload`消息
- 客户端根据消息作出不同的响应，`reload`或者无感刷新css

### 关闭服务

```js
// 主要就是关闭watcher的资源内容监听和关闭server服务
LiveServer.shutdown = function() {
	var watcher = LiveServer.watcher;
	if (watcher) {
		watcher.close();
	}
	var server = LiveServer.server;
	if (server)
		server.close();
};

// shutdown方法会在server的error事件中触发
server.addListener('error', function(e) {
	if (e.code === 'EADDRINUSE') {
		var serveURL = protocol + '://' + host + ':' + port;
		console.log('%s is already in use. Trying another port.'.yellow, serveURL);
		setTimeout(function() {
			server.listen(0, host);
		}, 1000);
	} else {
		console.error(e.toString().red);
		LiveServer.shutdown();
	}
});
```


### 该库功能的核心实现

简单抽取该库最主要的核心实现，基本如下100行代码：

```js
const http = require('http');
const fs = require('fs');
const path = require('path');
const url = require('url');
const open = require('open');
const send = require('send');
const eventStream = require('event-stream');
const fayeWebsocket = require('faye-websocket');
const chokidar = require('chokidar');

const config = {
  host: 'http://127.0.0.1',
  port: 3000,
  root: process.cwd(),
}

const injectContent = fs.readFileSync('./injected.html');

const server = http.createServer((req, res) => {
  let isInject = false;
  const reqPath = url.parse(req.url).pathname;

  function handleFile(filepath) {
    const ext = path.extname(filepath).toLocaleLowerCase();
    const targetFiles = possibleExtensions = [ '', '.html', '.htm', '.xhtml' ];
    const fileContent = fs.readFileSync(filepath, 'utf8');
    if (!req.headers.origin && targetFiles.includes(ext)) {
      const regexp = /<\/body>/g;
      if (regexp.exec(fileContent)) {
        isInject = true;
      }
    }
  }

  function inject(stream) {
    if (!isInject) return;

    // We need to modify the length given to browser
    const len = injectContent.length + res.getHeader('Content-Length');
    // 保存原pipe的引用
    const originalPipe = stream.pipe;

    res.setHeader('Content-Length', len);
    // 重写原pipe方法
    stream.pipe = function(resp) {
      // 重新调用pipe方法，并且理由event-stream模块，对流的内容进行注入内容
      // 注入的内容为websocket通信的部分
      originalPipe.call(stream, eventStream.replace(/<\/body>/g, injectContent + '</body>')).pipe(resp);
    };
  }

  send(req, reqPath, {
    root: config.root,
  }).on('stream', inject)
    .pipe(res);

});

let clients = [];
server.addListener('upgrade', (request, socket, head) => {
  const ws = new fayeWebsocket(request, socket, head);
  ws.onopen = function() {
    ws.send('connected');
  };
  ws.onmessage = function(e) {
    console.log('receive:', e.data);
    ws.send(e.data)
  }
  ws.onclose = function() {
    clients = clients.filter(function (x) {
      return x !== ws;
    });
  };
  clients.push(ws);
});

server.listen(3000, () => {
  const openPath = `${config.host}:${config.port}`;
  console.log(`[live-server] server is running at: ${openPath}`);

  // 服务启动成功后打开浏览器
  open(openPath, {
    app: null,
  });
});

const wathcer = chokidar.watch([config.root], { ignoreInitial: true });

wathcer.on('change', handleChange)
  .on('add', handleChange)
  .on('unlink', handleChange)
  .on('addDir', handleChange)
  .on('unlinkDir', handleChange)
  .on('error', (err) => {});

function handleChange(changePath) {
  console.log('file change');
  // 判断是否是css文件内容发生变化
  const cssChange = path.extname(changePath) === '.css';
  clients.forEach(function(ws) {
    if (ws) {
      ws.send(cssChange ? 'refreshcss' : 'reload');
    }
  });
}
```

# 其他

该库主要涉及的几个点：

- node脚本的书写
- http/https服务创建，并通过[connect](https://github.com/senchalabs/connect#readme)库支持中间件模型
- 利用[send](https://github.com/pillarjs/send#readme)库创建静态资源服务，并对流内容通过[event-straem](https://github.com/dominictarr/event-stream)库进行修改
- 利用[open](https://github.com/sindresorhus/open#readme)库打开浏览器或者其他应用
- 利用[chokidar](https://github.com/paulmillr/chokidar)进行文件内容变更的监听
- 利用[faye-websocket](https://github.com/faye/faye-websocket-node)在客户端连接时才初始化ws服务并建立一对一的连接