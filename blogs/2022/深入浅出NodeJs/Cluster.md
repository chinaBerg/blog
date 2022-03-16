# 精读《深入浅出NodeJs》- Cluster篇

cluster集群模块主要是node中利用多核系统处理负载问题。基本使用例子如下：

```js
// cluster.js
const cluster = require('cluster');
const os = require('os');

cluster.setupMaster({
  exec: __dirname + '/worker.js',
});

os.cpus().forEach(() => {
  cluster.fork();
});

// worker.js
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200);
  res.end('hello pid ' + process.pid);
});

server.listen(3200, () => {
  console.log(process.pid + ' start up.');
});
```

终端启动cluster.js

```bash
# 终端运行
node cluster.js

# mac运行结果如下：
42035 start up.
42034 start up.
42036 start up.
42037 start up.

# curl请求测试
curl http://localhost:3200
# 结果如下
hello pid 42036
```

`cluster`集群的实现原理正如进程模块所讲的，是利用`net`模块和`child_process`模块实现的。在`cluster`启动时内部启动`net`的`tcp`服务器，在`cluster.fork()`子进程时，会将tcp服务器的文件描述符通过句柄传递给子进程，子进程通过`SO_REUSEADDR`端口重用实现共享端口。

注意，在`cluster`应用中，一个主进程只能管理一组工作进程（例如只能管理一组`worker.js`，不同同时管理`worker.js`和`worker2.js`）。