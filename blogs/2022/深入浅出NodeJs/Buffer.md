# 精读《深入浅出NodeJs》- Buffer篇

Buffer是类Array的对象，主要作用是操作子节。Node场景下，在网络流和文件的操作中，需要处理大量二进制数据，字符串远不能满足需求，因此Buffer应运而生。

Node中Buffer是`js`与`c++`结合的模块，性能部分由c++实现，非性能部分由js实现。

Buffer是一个全局模块，在node进程启动时便已经加载。

### Buffer对象

Buffer对象类似于数组，每一项都是16进制（即0-255）的两位数：

```js
const buffer = Buffer.from('hello buffer');
// 输出 <Buffer 68 65 6c 6c 6f 20 62 75 66 66 65 72>
console.log(buffer)
```

Buffer默认使用utf-8编码，该编码下中文文字占三个元素，英文和半角标点占一个元素：

```js
const bufferChinese = Buffer.from('你好');
// 输出 <Buffer e4 bd a0 e5 a5 bd>
console.log(bufferChinese);
```

### 内存

Buffer内存分配是在Node的C++层面实现内存的申请，而不是在V8的堆内存。原因在于处理大量子节数据时，如果需要一点内存就向操作系统申请一点内存，会造成大量内存申请的系统调用，对操作系统造成压力。

Node在内存的使用上是在c++申请内存，在js中分配内存。采用slab动态内存管理机制。slab是一块申请好的固定大小的内存区域，具有3种状态：

- 完全分配状态
- 部分分配状态
- 未被分配状态

Node以8k作为界限区分Buffer是大对象还是小对象。对于小对象则使用slab预先申请和事后分配，大对象则直接使用c++层面提供的内存，不再细致分配。

### 转换、拼接

- 字符串与Buffer的互转:

```js
// 字符串转Buffer
const buffer = Buffer.from('hello');
// <Buffer 68 65 6c 6c 6f>
console.log(buffer);

// Buffer转字符串
const str = buffer.toString();
// hello
console.log(str);
```

- Buffer拼接

Buffer在使用场景中，经常是一段一段传输的：

```js
const fs = require('fs');

// 创建可读流
const readStream = fs.createReadStream('./demo.txt');

let str = '';
readStream.on('data', chunk => {
  str += chunk;
  // 等同于下面这段，因为隐藏了toString操作
  // str = str.toString() + chunk.toString();
});

readStream.on('end', () => {
  // 既许一人以偏爱，愿尽余生之慷慨。
  console.log(str);
});
```

Buffer的读取对于英文这种单字节数据没有问题 ，但是对于中文这种宽字节则会出现乱码的情况。宽字节在utf8下占三个元素，所以如果读取时存在了截断情况，则会出现乱码：

```js
// y以上面的例子，我们设置参数，让每次读取时只读取5个字节
const readStream = fs.createReadStream('./demo.txt', {
  highWaterMark: 5,
});

// 此时输出如下
// 既��一���以偏��，���尽余��之���慨。
```

原因是中文占三个元素，每次读取5字节，第一次读取的5个字节，前三个是一个中文，后两个字节则无法解析，因此出现了乱码。解决办法如下：

```js
const readStream = fs.createReadStream('./demo.txt', {
  highWaterMark: 5,
});
// 设置编码格式则可以解决
// 因为setEncoding方法背后指定宽字节在utf8下是三个字节，
// 因此在第一次读取5个字节时，遇到后两个字节无法转换，此时会把这两个字节保存在内部，
// 在下一次读取时拿出来拼接到下一次的五个字节再重新转换
rs.setEncoding('utf8');
```

上面这种方式，能解决大部分场景的乱码问题，但并不能从根上解决问题。

正确的读取方式，利用Buffer.concat将每次读取的小buffer合并成一个大的buffer，利用iocnv-lite再进行转换：

```js
let size = 0;
const chunks = [];

const readStream = fs.createReadStream('./demo.txt', {
  highWaterMark: 5,
});
readStream.on('data', chunk => {
  chunks.push(chunk);
  size += chunk.length;
});
readStream.on('end', () => {
  const buffer = Buffer.concat(chunks, size);
  const str = iconv.decode(buffer, 'utf8');
  // 既许一人以偏爱，愿尽余生之慷慨。
  console.log(str);
});
```

- Buffer是二进制数据，与字符串本质不同
- Buffer与字符串存在编解码关系
- highWaterMark的值设置要注意，过小会导致系统调用次数过多，值越大读取速度越快