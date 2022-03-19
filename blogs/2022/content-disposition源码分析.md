# content-disposition源码分析

> version 0.5.4
> 愣锤

[content-disposition](https://github.com/jshttp/content-disposition)库是在NodeJs中生成和解析Http Header中`Content-Disposition`字段的库。比如我们在实现Http文件下载的服务端逻辑时，想让浏览器访问请求时自动下载文件而不是读取展示文件，则需要设置该响应头的`Content-Disposition`字段。

目前该库被广泛应用于主流的NodeJS库中，并且每周下载量达到了2千万+次。下面我们先看看`Http`协议中`Content-Disposition`字段的具体信息吧。

### Http协议中Content-Disposition字段

在常规的 HTTP 应答中，`Content-Disposition` 响应头指示回复的内容该以何种形式展示，是以**内联**(`inline`)的形式（即网页或者页面的一部分），还是以**附件**(`attachment`)的形式下载并保存到本地。

在 `HTTP` 场景中，第一个参数或者是 `inline`（默认值，表示回复中的消息体会以页面的一部分或者整个页面的形式展示），或者是 `attachment`（意味着消息体应该被下载到本地；大多数浏览器会呈现一个“保存为”的对话框，将 `filename` 的值预填为下载后的文件名，假如它存在的话）

```bash
Content-Disposition: inline
Content-Disposition: attachment
Content-Disposition: attachment; filename="filename.txt"
```

[Http协议中Content-Disposition字段文档](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Disposition)

下面演示基于`Content-Disposition`字段创建让浏览器自动下载pdf资源的`http`服务。

```js
const http = require('http');
const fs = require('fs');
const mime = require('mime');
const contentDisposition = require('content-disposition');


const server = http.createServer((req, res) => {
  // pdf资源路径
  const pdfPath = __dirname + '/demo.pdf';
  // pdf资源的Content-type值： application/pdf
  const pdfType = mime.getType('pdf');
  const stream = fs.createReadStream(pdfPath);

  res.setHeader('Content-type', pdfType);
  // 设置下载
  // 当浏览器输入http://localhost:3000时会自动弹出下载窗口
  // 如果没有改请求头字段设置，则浏览器展现的是pdf的预览
  res.setHeader('Content-Disposition', contentDisposition(pdfPath));
  stream.pipe(res);
});

server.listen(3000, () => {
  console.log('[pdf download server] running at port 3000.');
});
```

此时在浏览器窗口输入`http://localhost:3000`时则会弹出自动下载窗口，如下图所示：

![image](https://note.youdao.com/yws/res/20121/518BAABE5A544A369A1464E831C5E18E)

如果没有设置`Content-Disposition`的逻辑则会是浏览器预览的效果，如下图所示：

![image](https://note.youdao.com/yws/res/20126/10F724FE8B0E4DF99DC06E1B4DACCC1D)