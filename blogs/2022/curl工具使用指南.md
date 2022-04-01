# curl工具使用指南

> 愣锤 2022/04/02

### curl介绍

`curl`是一个命令行工具，用于发送客户端请求。发送客户端请求大家常用的可能是类似`postman`等工具，但是为什么要使用`curl`呢？`curl`等最大优势在于随时随手可以发送，非常方便。比如很多场景下我们只是想快速验证一个请求或接口：

```bash
# 直接在命令行发送一个GET请求
curl https://xxx.com/api/v1/xxx

# 发送POST请求
curl -X POST -d "k1=123&k2=456" https://xxx.com/api/v1/xxx
```

### curl安装

`curl`的安装就是到[官网](https://curl.se/download.html)根据你的系统下载对应的版本进行安装，但是安装好后要配置环境变量。

```bash
# 安装好后重启终端运行，查看版本
curl -V
```

如果能看到如下内容则是安装成功了:

![image](https://note.youdao.com/yws/res/20948/C3B770B7BBF4471682319D15636D0782)

## 发送GET请求

- `curl`后面直接添加url地址即可访问`GET`请求

```bash
curl https://www.baidu.com
```

请求百度网址的效果如下:

![image](https://note.youdao.com/yws/res/20901/4FF07C0971194BE0BE3B97C9F4B80831)

- 发送`GET`请求携带请求参数

```bash
curl https://www.xxx.com/?key=value1&key2=value2
```

### POST请求

`-X POST --data "k1=v1&k2=v2"`发送post请求，并且携带请求数据。下面演示一个接收`POST`请求并返回`POST`数据的的`Node`服务和`CURL`发送`POST`请求示例：

```js
/**
 * http服务，处理post请求并将post数据返回
 */
const http = require('http');

const server = http.createServer((req, res) => {
  if (req.method === 'POST' && hasbody(req)) {
    const buffer = [];
    req.on('data', (chunk) => {
      buffer.push(chunk);
    });
    req.on('end', () => {
      const rawBody = Buffer.concat(buffer).toString();
      res.writeHead(200);
      res.end(rawBody);
    });
  } else {
    res.end('');
  }
});

server.listen(3000, () => {
  console.log('server running at port 3000.');
});

// 判断是否有body请求实体数据
function hasbody (req) {
  return req.headers['transfer-encoding'] !== undefined ||
    !isNaN(req.headers['content-length']);
}
```

`curl`发送`post`请求:

```bash
curl -X POST --data "key1=123&key2=456" http://localhost:3000

# --data可以简写为-d
curl -X POST -d "key1=123&key2=456" http://localhost:3000
```
![image](https://note.youdao.com/yws/res/20843/F1A4CB74CC3B49569B9C3F0C99A3446F)

- 对`post`数据进行`url`编码

```bash
# 注意，--data-urlencode的简写不是-d
# --data的简写是-d
curl --data-urlencode "k1=1&k2=a b"  http://localhost:3000
```

例如下面发送对请求数据中，`a`和`b`之间有个空格，使用`--data-urlencode`会对其进行`encodeURIComponent`编码：

![image](https://note.youdao.com/yws/res/20918/5A90D56EE6BF43B9A85C509C95062CC8)

### 发送HEAD请求

```js
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200);
  res.end();
});

server.listen(3000, () => {
  console.log('server running at port 3000.');
});
```

`-I` 参数可以发送`HEAD`请求:

```bash
curl -I http://localhost:3000
```

![image](https://note.youdao.com/yws/res/20922/0A271A3350EF43968FB07210A4574A63)


### 发送DELETE请求

`-X DELETE`参数可以发送`DELETE`请求：

```bash
curl -X DELETE http://localhost:3000
```

![image](https://note.youdao.com/yws/res/20958/B3BB640353ED440B8D3EE1ADA78251BA)

### 发送PUT请求

下面起一个非常简易的`node`服务，将用户上传的图片保存成图片：

```js
const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
  // 如果是PUT请求且访问的接口是/upload/file
  if (req.method === 'PUT' && req.url === '/upload/file') {
    // 将用户的数据存成图片static/mime.png图片
    const writeStream = fs.createWriteStream(__dirname + '/static/mime.png');
    req.pipe(writeStream).on('finish', () => {
      res.writeHead(200);
      res.end('upload success');
    });
    writeStream.on('error', (err) => {
      res.writeHead(500);
      res.end('server error.');
    });
  }
});

server.listen(3000, () => {
  console.log('server running at port 3000.');
});
```

利用`curl`的`-T`可以发送`PUT`类型请求，同时需要指定上传的资源路径:

```bash
curl -T ./mime.png http://localhost:3000/upload/file
```

![image](https://note.youdao.com/yws/res/20970/8590120F2B674C778BF484CFF0809B04)

同时请求结束后，可以看到上传的图片已经被保存下来了:

![image](https://note.youdao.com/src/53D3B2E9D36E40BEB194D80D965E6405)

### curl下载文件

下载保存文件是加`-o 保存地址`参数。

```bash
# 将baidu的html文件下载到本地
curl -o ./my-download.html https://www.baidu.com
```

`curl`命令执行的效果如下图，而且文件也已经被下载了下来：

![image](https://note.youdao.com/yws/res/20814/E15BE5AE240F4945890285C4068D9072)

### 查看响应头参数

`-i`参数可以返回响应头信息

```bash
curl -i  https://www.baidu.com
```

![image](https://note.youdao.com/yws/res/20817/C82EF13E2A3844278D28B55BDFF6F273)

### 查看完整的报文信息

- `-v`参数查看完整的报文信息

```bash
curl -v https://www.baidu.com
```

![image](https://note.youdao.com/yws/res/20822/C1A2E6B0C18046CD9DADE66A6DF9A955)

- `--trace ./log.txt`查看更详细的信息并将数据写入到指定文件中。

```bash
curl --trace ./log.txt https://www.baidu.com
```

![image](https://note.youdao.com/yws/res/20831/8AD1A95D6C7548229B3295745A190892)

- `--trace-ascii ./log.txt`以ascii编码格式查看更详细的信息并将数据写入到指定文件中。


### 指定请求的user-agent

我们启动一个最简单的`http`服务，并且配置好`vscode`的`debug`用于我们查看`curl`发送的信息：

```js
const http = require('http');

const server = http.createServer((req, res) => {
  // 在此处打上断点，查看req请求对象
  res.end('');
});

server.listen(3000, () => {
  console.log('server running at port 3000.');
});
```

如果直接通过`curl`发送`GET`请求，可以看到`user-agent`为：

![image](https://note.youdao.com/yws/res/20864/98D5DFBF01F5423B9BBCF67C310C8DE5)

`chrome`浏览器的`user-agent`是`Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.109 Safari/537.36`

`--user-agent "xxx"`可以指定发送请求时的`user-agent`，参数简写为`-A`:

```bash
curl --user-agent "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.109 Safari/537.36" http://localhost:3000

# --user-agent简写为-A
curl -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.109 Safari/537.36" http://localhost:3000
```

![image](https://note.youdao.com/yws/res/20853/255A83FC5F864A4FA2F04AD9F0D934E2)

### 指定请求的跳转

`--referer 跳转前url 跳转后url`

```bash
curl --referer http://localhost:3000 http://localhost:3000/newpath
```

此时发送的请求的`req.url`是新的地址`http://localhost:3000/newpath`，并且`headers`中携带了`referer`字段。

![image](https://note.youdao.com/yws/res/20872/6EC1CFE6F6DF45DB832ECADDA7612922)


### 请求时携带cookie参数

- `--cookie "k1=v1&k2=v2"`携带`cookie`参数，`--cookie`可以简写为`-b`:

```bash
curl --cookie "k=1&k2=2" http://localhost:3000

# --cookie简写为-b
curl -b "k=1&k2=2" http://localhost:3000
```

![image](https://note.youdao.com/yws/res/20881/A33D7B02DA2C4D7A9F6F687F8F8563F2)

- `curl`保存服务端的`cookie`到指定文件

为了使用`curl`时能携带服务端设置的`cookie`，我们可以先把服务的`cookie`存到本地，然后后续使用`curl`的适合再携带上。如下，有一个`node`设置`cookie`的例子：

```js
const http = require('http');

const server = http.createServer((req, res) => {
  // Node设置cookie
  res.writeHead(200, {
    'Set-Cookie': 'key1=value1&key2=value2',
  });
  res.end('cookie set success.');
});

server.listen(3000, () => {
  console.log('server running at port 3000.');
});
```

通过`curl`发送请求携带`-c path/to/save`，可以将`node`设置的`cookie`保存到本地:

```bash
curl -c ./cookie http://localhost:3000
```

![image](https://note.youdao.com/yws/res/20893/5F3CA92AEB6E4CF59F8BFE8BBA81CFE2)

- `curl`发送请求时携带`cookie`文件

```bash
curl -b ./cookie http://localhost:3000
```

此时`debug`可以看到`req`上已经携带了我们的`cookie`:

![image](https://note.youdao.com/yws/res/20895/17A9EE38E70E4E37B35758C2D0A6847D)


### 参考

- [curl文档](https://catonmat.net/cookbooks/curl)
- [curl网站开发指南](https://www.ruanyifeng.com/blog/2011/09/curl.html) 阮一峰
- [curl 的用法指南](https://www.ruanyifeng.com/blog/2019/09/curl-reference.html) 阮一峰