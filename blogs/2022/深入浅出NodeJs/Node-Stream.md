# Node Stream模块

下面演示一个读取文件内容并作为http请求资源返回的例子：

- 使用文件模块读取

如果读取的文件内容巨大，则在响应大量用户并发请求时会消耗大量内存导致用户连接缓慢。

```js
const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
  fs.readFile(__dirname + '/demo.txt', (err, body) => {
    res.end(body);
  });

});

server.listen(3200, () => {
  console.log('server running at port 3200.');
});
```

- 使用流读取

文件内容数据会一段一段的源源不断的发送给用户。

```js
const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
  const stream = fs.createReadStream(__dirname + '/demo.txt');
  stream.pipe(res);
});

server.listen(3200, () => {
  console.log('server running at port 3200.');
});
```

### 流的类型

- Readable - 可读流
- Writable - 可写流
- Duplex - 双工流，即Readable 和 Writable 流
- Transform - 转换流，即可以在写入和读取数据时修改或转换数据的 Duplex 流


### Readable

- 可读流的主要功能是作为上游，提供数据给下游（可写流、双工流、转换流）
- 可读流通过`push`方法生产数据，当调用`push(null)`则宣告生产数据结束
- 创建流，按需生产数据与非按需生产数据

大多情况下，我们更期望在需要数据时才生产数据，以此避免大量缓存数据。
```js
const { Readable } = require('stream');

// 非按需生产数据，在push数据时数据就添加到缓存中了
const readable = Readable();
readable.push('a');
readable.push('b');
readable.push('c');
// 宣告生产数据结束
readable.push(null);

// 将可读流与可写流process.stdout连接，供process.stdout消费
// 注意：在process.stdout消费数据之前，数据已经添加到缓存了
readable.pipe(process.stdout);


// 可读流按需生产数据
const data = ['a', 'b', 'c'];
const readable2 = Readable({
  read() {
    this.push(data.shift() || null);
  }
});

// 只有在process.stdout消费时才按需生产推送数据
readable2.pipe(process.stdout);
```

- 可读流有两种读取模式（消费可读流）：
    - 流动模式。数据自动从底层获取，通过接口事件尽快返回应用程序
    - 暂停模式。必须显示调用`.read()`方法从流中读取数据块

下面演示可读流的流动模式数据获取:

```js
const { Readable } = require('stream');

const source = ['a', 'b', 'c', 'd'];

// 定义可读流
const readable = Readable({
  read() {
    // 可读流实例通过push生产数据，遇到null时宣告生产结束，否则下游一直等待
    const data = source.shift() || null;
    this.push(data);
  }
});

// 监听data事件，会从缓存中获取数据进行消费
// 监听data事件自动进入'flowing'模式
readable.on('data', (data) => {
  console.log('data', data);
});

// 可读流数据被消耗完时触发end事件
readable.on('end', () => {
  console.log('end');
});
```

下面演示可读流暂停模式数据的读取:

```js
const { Readable } = require('stream');

const source = ['a', 'b', 'c', 'd'];

// 定义可读流
const readable = Readable({
  read() {
    // 可读流实例通过push生产数据，遇到null时宣告生产结束，否则下游一直等待
    const data = source.shift() || null;
    this.push(data);
  }
});

// readable事件表示流中产生了新数据或者到了流的尽头
readable.on('readable', () => {
  let data;
  // readable.read(n)从缓存中尝试读取n字节的数据；n不传递则从缓存一次性读取全部的数据
  while(data = readable.read()) {
    // <Buffer 61 62>
    // <Buffer 63>
    // <Buffer 64>
    console.log(data);
  }
});
```

所有可读流均以暂停模式开始，但可以通过以下方式之一切换到流动模式：
- 添加 `data` 事件句柄
- 调用 `.resume()` 方法
- 调用 `.pipe()` 方法将数据发送到可写流

可读流使用以下模式切换回暂停模式：
- 如果没有管道目标，则通过调用 `.pause()` 方法
- 如果有管道目标，则删除所有管道目标后通过调用 `.unpipe()` 方法删除多个管道目标

添加 `readable` 事件句柄会自动使流停止流动，并且必须通过 `readable.read()` 来消费数据。 如果删除了 `readable` 事件句柄，则如果有 
`data` 事件句柄，流将再次开始流动

可读流示例：

- 客户端上的 HTTP 响应
- 服务器上的 HTTP 请求
- 文件系统读取流
- 压缩流
- 加密流
- TCP 套接字
- 子进程的标准输出和标准错误
- process.stdin

### 可写流

可写流是作为下游消费上游提供的数据。

```js
const { Writable } = require('stream');

const writable = Writable({
  write(data, _, next) {
    console.log(data);
    process.nextTick(next);
  }
});


writable.on('finish', () => {
  console.log('finish');
});

writable.write('a');
writable.write('bc');
writable.end();

// 输出结果如下
// <Buffer 61>
// <Buffer 62 63>
// finish
```

可写流示例：

- 客户端上的 HTTP 请求
- 服务器上的 HTTP 响应
- 文件系统写入流
- 压缩流
- 加密流
- TCP 套接字
- 子进程标准输入
- process.stdout、process.stderr

### 连接可读流与可写流

- 通过pipe连接，上游是可读流，下游是可写流

```js
const { Readable, Writable} = require('stream');

function createReadable() {
  const source = ['a', 'b', 'c'];
  return Readable({
    read() {
      process.nextTick(this.push.bind(this), source.shift() || null);
    },
  });
}

function createWritable() {
  return Writable({
    write(data, _, next) {
      console.log('write data: ', data);
      next();
    },
  });
}

const readable = createReadable();
const writeable = createWritable();
readable.pipe(writeable).on('finish', () => {
  console.log('finish');
});
```

- 基于事件的方式连接可读流和可写流

```js
const readable = createReadable();
const writeable = createWritable();

readable.on('data', (data) => {
  writeable.write(data);
});

readable.on('end', () => {
  writeable.end();
});

writeable.on('finish', () => {
  console.log('finish');
});
```

注意，pipe的方式相比事件方式，pipe内部做了控制内存的优化：

- 可写流中缓存队列的长度已经到达了临界值时，会暂停可读流的输出
- 等待可写流处理完缓存后再继续flowing可读流继续输出

### pipeline

`pipe`方法返回对目标流的引用，从而可以建立管道流链。创建多个流后，用`pipe`将其连起来便形成了`pipeline`：

```js
function toUpperCase() {
  return Transform({
    transform(bufer, _, next) {
      next(null, bufer.toString().toUpperCase());
      // 上述写法等同于下面
      // this.push(bufer.toString().toUpperCase());
      // next(null);
    }
  });
}


const readable = createReadable();
const writeable = createWritable();

// pipelie输出结果如下:
// write data:  <Buffer 41>
// write data:  <Buffer 42>
// write data:  <Buffer 43>
// finish
readable.pipe(toUpperCase()).pipe(writeable).on('finish', () => {
  console.log('finish');
});
```

需要注意的是中间环节的流必须是可读可写的，即`Duplex`和`Transform`。