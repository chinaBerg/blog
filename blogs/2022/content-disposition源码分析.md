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
  // pdf文件名称正常的资源路径
  const pdfPath = __dirname + '/demo.pdf';
  // pdf资源名称带iso-8859-1特殊字符的文件名
  // const pdfPath = __dirname + '/demoŸ.pdf';

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

通过上述例子和知识点我们知道一个下载服务如果想让接收端以下载附件的方式响应，则只需要设置`Content-Disposition`格式即可。那么我们还使用`content-disposition`库做什么？目的是什么呢？

原因就在于，比如我们指定下载的`filename`包含了`ISO-8859-1`字符集之外的字符的时候（例如一些西欧语言的文字符号）是否要自动进行`ISO-8859-1`格式的转码，以及`Content-Disposition`是否要支持`RFC 5987`标准。

```js
// 一个正常命名的demo.pdf的Content-Disposition
'attachment; filename="demo.pdf";'

// 带iso-8859-1特殊字符的demoŸ.pdf的Content-Disposition
'attachment; filename="demo?.pdf"; filename*=UTF-8''demo%C5%B8.pdf'
```

因此在讲解`content-disposition`库的源码之前，我们要先简单聊聊不同编码概念和区别。

### 浅谈计算机中常见编码及区别

作为一个程序员，在计算机中不同的编码想必大家多少都有接触到，比如`Unicode`，`GBK`，`ASCII`，`utf8`，`utf16`，`ISO8859-1`等。所谓字符集编码其实就是将字符和计算机中的数字（二进制存储）进行一一的映射起来。

下面看看这些不同的编码概念及区别吧，这些编码概念些许有些枯燥却是后续源码理解的基础：

- ASCII编码

最初计算机由美国发明出来，因此编码只需要应对`26`个字母、数字和一些常见标点字符即可，`ASCII`应运而生。标准`ASCII` 码使用`7`位二进制数，最高位恒为`0`, 可以表示`128`个字符。例如在`JS`中字符和`ASCII`的互转：

```js
// 字符转十进制的ASCII
var str = 'A';
str.charCodeAt(); // 65

// 十进制的ASCII转字符
String.fromCharCode(65) // 'A'
```

ASCII码对照表可以[查阅这里](https://www.habaijian.com/) https://www.habaijian.com/

- ISO8859-1编码

随着世界各地都开始使用计算机，不同语言对字符编码提出了新的需求，原`ASCII`的`128`个字符已经不够，因此扩展了ASCII最高位变成了`256`个字符的编码，这就是`ISO8859-1`。

> `ISO-8859-1`编码是单字节编码，向下兼容`ASCII，`其编码范围是`0x00-0xFF`，`0x00-0x7F`之间完全和`ASCII`一致，`0x80-0x9F`之间是控制字符，`0xA0-0xFF`之间是文字符号。此字符集支持部分于欧洲使用的语言。
>
> ISO-8859-1标准中0x80-0xFF为控制字符。ISO-8895-15去除了0x80-0xFF中的控制字符，在0x80-0xFF加入了œ、Œ、Ÿ 、Š、š、Ž、ž等字母和欧元（€）、单引号（‘’）、双引号（“”）、斜体f（ƒ）、省略号（…）、商标（™）、千分号（‰）等常用符号
>
> ----来自于百度百科

- GBK编码

计算机进入中国后，中国文字非常多，常用汉字就有`6000`多，`ASCII/ISO8859-1`的单字节是无法满足汉字编码需求的。因此我们便使用`2字节`表示汉字及字符，小于`127`的与原`ASCII`意义相同，这样便有了支持`7000`多个汉字编码的`GBK2312`编码。

但是汉字太多了，`GBK2312`编码依旧不够用，我们因此继续扩展多个字节，`3字节`甚至`更多字节`，只要第一个字节满足大于`127`，我们就认为是汉字编码，这种扩展后的编码称为`GBK`。

- Unicode编码

比如中国搞出了GBK，那么其他国家也扩展各自的编码，这就导致同样的编码却表示不同的意思。因此`ISO`（国际标谁化组织）制定了全球统一的字符集编码方案，叫`UCS`(`Universal Character Set`)，俗称`Unicode`。`Unicode`统一采用16位二进制编码字符, 是一个很大的集合，现在的规模可以容纳100多万个符号。

`Unicode`的问题在于：计算机无法知道多个字节到底是表示一个字符还是分别表示多个字符；第二个问题是`Unicode`如果规定所有字符都用三四个字节表示，那么对于英文这些只需要一个字节的来说会造成存储的大量浪费。

- UTF8编码

`UTF-8`是使用最广的`Unicode`实现，它的最大特点是可变长的编码方式，可以使用`1~4`个字节表示一个符号，根据不同的符号变化字节长度。解读 `UTF-8`编码非常简单。如果一个字节的第一位是`0`，则这个字节单独就是一个字符；如果第一位是`1`，则连续有多少个`1`就表示当前字符占用多少个字节。

- URL的编解码

在开发中，我们经常会对URL使用`encodeURIComponent | decodeURIComponent`进行编解码，如下述代码：

```js
// 对url进行编码
// 'http%3A%2F%2Flocalhost%3A3000%2F%3Fquery%3D%E4%BD%A0%E5%A5%BD'
encodeURIComponent('http://localhost:3000/?query=你好')
```

可以看到`:`被编码成`%3A`，`?`被编码成`%3F`等，那么这些`%`后面的值是哪来的呢？其实就是十六进制的`utf8`值。解码就是去掉`%`然后利用`utf8`解码成字符。

好了，有了这些不同编码规范的基础，下面让我正式进入`content-disposition`库的原理解析吧。

### Content-Disposition生成原理

下面我们来看具体的源码处理吧，`content-disposition`库源码只有一个文件，对外导出了两个方法，主体结构如下：

```js
'use strict'

/**
 * 对外到导出的模块
 * @public
 */

module.exports = contentDisposition
module.exports.parse = parse

function contentDisposition (filename, options) {}

function parse (string) {}
```

下面我们看下`contentDisposition`函数的实现，该函数最终就是生成我们需要的`Content-Disposition`的值。

```js
/**
 * 创建 Content-Disposition header
 *
 * @param {string} [filename]
 * @param {object} [options]
 * @param {string} [options.type=attachment]
 * @param {string|boolean} [options.fallback=true]
 * @return {string}
 * @public
 */

function contentDisposition (filename, options) {
  var opts = options || {}

  // 定义type值，默认值attachment
  // 告知接收者以何种方式展示响应的数据，
  // attachment表示以附件下载的方式保存到本地
  var type = opts.type || 'attachment'

  // 根据filename和fallback获取要生成header头相关字段的对象格式的参数
  var params = createparams(filename, opts.fallback)

  // 将对象参数格式化成header的字符串格式
  return format(new ContentDisposition(type, params))
}
```

主要逻辑就是定义默认参数，根据`filename`和`fallback`获取要生成`header`头相关字段的对象格式的参数，最后将对象格式的参数格式化成真正的`header`头的字符串格式值并返回。

看下`createparams`的实现：

```js
/**
 * 匹配Latin1字符集以外的字符的正则对象
 * @private
 */
var NON_LATIN1_REGEXP = /[^\x20-\x7e\xa0-\xff]/g

// RFC 2616标准下的TEXT语法字符范围的正则对象
var TEXT_REGEXP = /^[\x20-\x7e\x80-\xff]+$/

// 匹配百分号编码格式的正则对象
var HEX_ESCAPE_REGEXP = /%[0-9A-Fa-f]{2}/

/**
 * 从filename和fallback参数创建header参数对象
 *
 * @param {string} [filename]
 * @param {string|boolean} [fallback=true]
 *   如果filename值中存在ISO-8859-1字符集以外的字符，
 *   是否将filename值自动升级为ISO-8859-1版本
 * @return {object}
 * @private
 */

function createparams (filename, fallback) {
  if (filename === undefined) {
    return
  }

  var params = {}

  if (typeof filename !== 'string') {
    throw new TypeError('filename must be a string')
  }

  // fallback 默认值为true
  if (fallback === undefined) {
    fallback = true
  }

  if (typeof fallback !== 'string' && typeof fallback !== 'boolean') {
    throw new TypeError('fallback must be a string or boolean')
  }

  // 如果fallback是字符串，则必须符合ISO-8859-1字符集
  // 选项为字符串说明就是filename不符合ISO-8859-1时不自动生成，而是用fallback的值
  if (typeof fallback === 'string' && NON_LATIN1_REGEXP.test(fallback)) {
    throw new TypeError('fallback must be ISO-8859-1 string')
  }

  // 文件名称，Eg： demo.txt
  var name = basename(filename)

  // determine if name is suitable for quoted string
  // 是否是符合RFC 2616字符集规则的文本
  var isQuotedString = TEXT_REGEXP.test(name)

  // generate fallback name
  /**
   * 获取filename的name的回退后的值
   * - 如果fallback是true，则将name编码成ISO-8859-1版本
   * - 如果是字符串，则直接取fallback值替代自动转换ISO-8859-1的逻辑
   */
  var fallbackName = typeof fallback !== 'string'
    ? fallback && getlatin1(name)
    : basename(fallback)
  /**
   * 判断是否回退了，即是否有不符合ISO-8859-1的值进行了ISO-8859-1编码
   * - fallback为false，则未回退
   * - fallback为true，则判断回退后的值fallbackName和原值name是否相等
   * - fallback为字符串，则判断fallback值和原值name是否相等
   */
  var hasFallback = typeof fallbackName === 'string' && fallbackName !== name

  // set extended filename parameter
  // 如果编码了ISO-8859-1，或者包含RFC 2616标准的字符集规则外的文本，或者包含%格式的转移符，
  // 则默认使用值为 filename* 的header 字段
  if (hasFallback || !isQuotedString || HEX_ESCAPE_REGEXP.test(name)) {
    params['filename*'] = name
  }

  // set filename parameter
  // 如果是符合RFC 2616标准的字符集规则的，则根据hasFallback
  // 选择fallbackName或者name
  if (isQuotedString || hasFallback) {
    params.filename = hasFallback
      ? fallbackName
      : name
  }

  return params
}
```

此处主要的逻辑是：

- 进行`fallback`等参数的类型校验
- 获取要下载的文件的文件名
- 根据文件名的值以及`fallback`等参数，决定是否要对文件名进行`ISO-8859-1`进行编码
- 确定是否要进行`RFC 2616`标准的支持
- 最终返回参数对象

接下来我们看下`getlatin1`是如何将unicode转码成
`ISO-8859-1`码的:

```js
/**
 * 匹配Latin1字符集以外的字符的正则对象
 * @private
 */
var NON_LATIN1_REGEXP = /[^\x20-\x7e\xa0-\xff]/g

/**
 * Get ISO-8859-1 version of string.
 * 简单的将字符串从Unicode编码转换到ISO-8859-1编码格式
 * @param {string} val
 * @return {string}
 * @private
 */

function getlatin1 (val) {
  // simple Unicode -> ISO-8859-1 transformation
  // 就是将ISO-8859-1以外的字符全部替换成?
  return String(val).replace(NON_LATIN1_REGEXP, '?')
}
```

这里要补充一下`ISO-8859-1`码的知识点：

- `ISO-8859-1`编码是单字节编码，向下兼容`ASCII`，其编码范围是`0x00-0xFF`
    - `0x00-0x7F`之间完全和`ASCII`一致
    - `0x80-0x9F`之间是控制字符
    - `0xA0-0xFF`之间是文字符号
- `Latin1`是`ISO-8859-1`的别名，有些环境下写作`Latin-1`

接下来我们看下`format`的逻辑是如何将参数对象转换成`header`的字符串格式的:

```js
/**
 * 将object格式化成Content-Disposition的http header字符串
 *
 * @param {object} obj
 * @param {string} obj.type
 * @param {object} [obj.parameters]
 * @return {string}
 * @private
 */

function format (obj) {
  // 参数对象
  var parameters = obj.parameters
  // 类型
  var type = obj.type

  if (!type || typeof type !== 'string' || !TOKEN_REGEXP.test(type)) {
    throw new TypeError('invalid type')
  }

  // 拼接Content-Disposition值的开头部分
  // 例如 attachment;
  var string = String(type).toLowerCase()

  // 拼接Content-Disposition的key/value字符串部分
  if (parameters && typeof parameters === 'object') {
    var param
    var params = Object.keys(parameters).sort()

    for (var i = 0; i < params.length; i++) {
      param = params[i]

      // 取值逻辑：
      // 如果是key*格式的则编码成RFC5987标准的http字符集
      // 否则则是直接最字符串双引号转译然后前后加双引号
      var val = param.substr(-1) === '*'
        ? ustring(parameters[param])
        : qstring(parameters[param])

      // 最终得到attachment;filename="a.txt";格式的字符串
      string += '; ' + param + '=' + val
    }
  }

  return string
}
```

`format`的逻辑如下所示：

- 遍历参数对象的所有key
- 根据参数的类型进行不同的字符串拼接逻辑
    - 如果是`*`结尾的，例如`filename*`则调用`ustring`将`value`编码成`RFC5987`标准的格式
    - 否则调用`qstring`仅对`value`的双引号进行转译然后首尾加上双引号
- 最终编译的通用格式，例如为`attachemnt;filename=demo.txt`

我们再看下`ustring`是如何将`value`转译成`RFC 5987`标准的：

```js
/**
 * 匹配encodeURIComponent之后的url所有特殊字符的正则对象，不包含百分号
 * @private
 */
var ENCODE_URL_ATTR_CHAR_REGEXP = /[\x00-\x20"'()*,/:;<=>?@[\\\]{}\x7f]/g

/**
 * Encode a Unicode string for HTTP (RFC 5987).
 * 编码成RFC 5987标准的格式 @see https://datatracker.ietf.org/doc/html/rfc5987
 * @param {string} val
 * @return {string}
 * @private
 */

function ustring (val) {
  var str = String(val)

  // percent encode as UTF-8
  // 对str进行encodeURIComponent编码，然后将特殊字符转译成十六进制的ascii的格式
  // ENCODE_URL_ATTR_CHAR_REGEXP正则匹配url除去%外的特殊字符
  var encoded = encodeURIComponent(str)
    .replace(ENCODE_URL_ATTR_CHAR_REGEXP, pencode)

  return 'UTF-8\'\'' + encoded
}
```

这里可以看到首先使用`encodeURIComponent`进行编码，然后对url中的除`%`之外的特殊字符进行转译成十六进制的`ascii`码格式。再看下`qstring`是如何转译`value`的:

```js
/**
 * 匹配双引号的正则对象
 * @private
 */
var QUOTE_REGEXP = /([\\"])/g

/**
 * Quote a string for HTTP.
 * 对字符串内的双引号加上转译符，同时首尾加上双引号，用于http
 * @param {string} val
 * @return {string}
 * @private
 */
function qstring (val) {
  var str = String(val)

  // 这么这行的逻辑首先是首尾加上双引号
  // replace逻辑是对QUOTE_REGEXP匹配到的字符串内的双引号替换成'\\$1'
  // '\\$1'即表示替换成\\"
  return '"' + str.replace(QUOTE_REGEXP, '\\$1') + '"'
}
```

### Content-Disposition反向解析原理

`Content-Disposition`既然可以生成，那自然是可以反向解析出type和其他参数对象的。对应在`content-disposition`则是通过使用`parse`逻辑

```js
const pdfPath = __dirname + '/demo.pdf';
const params = contentDisposition.parse(
  contentDisposition(pdfPath),
);
```

![image](https://note.youdao.com/yws/res/20245/84E74FB518A64AD5AF7FAEFA0AB6954D)

下面我们来看看parse函数是如何反向解析的吧：

```js
/**
 * 解析Content-Disposition header字符串
 * @param {string} string
 * @return {object}
 * @public
 */

function parse (string) {
  if (!string || typeof string !== 'string') {
    throw new TypeError('argument string is required')
  }

  // 匹配到的type部分，含空格、分号等
  var match = DISPOSITION_TYPE_REGEXP.exec(string)

  if (!match) {
    throw new TypeError('invalid type format')
  }

  // normalize type
  var index = match[0].length
  // match的第一个子表达式匹配到的结果表示type的值
  var type = match[1].toLowerCase()

  var key
  var names = []
  var params = {}
  var value

  // calculate index to start at
  // 参数开始的位置为匹配到type之后的位置
  index = PARAM_REGEXP.lastIndex = match[0].substr(-1) === ';'
    ? index - 1
    : index

  // 利用正则开始匹配后续的key=value的参数
  while ((match = PARAM_REGEXP.exec(string))) {
    if (match.index !== index) {
      throw new TypeError('invalid parameter format')
    }

    index += match[0].length
    // 获取key
    key = match[1].toLowerCase()
    // 获取value
    value = match[2]

    if (names.indexOf(key) !== -1) {
      throw new TypeError('invalid duplicate parameter')
    }

    names.push(key)

    // 如果key是*结尾的，例如filename*
    if (key.indexOf('*') + 1 === key.length) {
      // decode extended value
      // key截取*之前的部分
      key = key.slice(0, -1)
      // value进行解码
      value = decodefield(value)

      // overwrite existing value
      params[key] = value
      continue
    }

    // 已存在的忽略
    if (typeof params[key] === 'string') {
      continue
    }

    // 如果是编码时对value进行了双引号转码以及首尾加了双引号
    if (value[0] === '"') {
      // remove quotes and escapes
      // 移除收尾的双引号，对value内转译后的双引号解码回来
      value = value
        .substr(1, value.length - 2)
        .replace(QESC_REGEXP, '$1')
    }

    params[key] = value
  }

  if (index !== -1 && index !== string.length) {
    throw new TypeError('invalid parameter format')
  }

  return new ContentDisposition(type, params)
}
```

解析逻辑如下：

- 利用正则匹配到`type`的部分字符串
- 再从`type`后面的部分利用正则开始匹配类似`;key=value`的参数
- 对匹配到的key去除末尾的*
- 对匹配到的value去除首尾的双引号，对value本身进行解码等

下面我们看下是如何匹配到type的这个正则对象：

```js
/**
 * RegExp for various RFC 6266 grammar
 *
 * disposition-type = "inline" | "attachment" | disp-ext-type
 * disp-ext-type    = token
 * disposition-parm = filename-parm | disp-ext-parm
 * filename-parm    = "filename" "=" value
 *                  | "filename*" "=" ext-value
 * disp-ext-parm    = token "=" value
 *                  | ext-token "=" ext-value
 * ext-token        = <the characters in token, followed by "*">
 * @private
 */
/**
 * 匹配Content-Disposition的type的字段
 * - 匹配逻辑为1-n个第一个小括号内的字符
 * - 后面跟0-n个制表符或空格
 * - 再后面匹配分号或者前面的规则一直匹配到结尾
 */
var DISPOSITION_TYPE_REGEXP = /^([!#$%&'*+.0-9A-Z^_`a-z|~-]+)[\x09\x20]*(?:$|;)/
```

根据`RFC 6266`标准的语法写成对应的正对对象，正则对象的逻辑为：

- 匹配逻辑为`1-n`个第一个小括号内的字符
- 后面跟`0-n`个制表符或空格
- 再后面匹配分号或者前面的规则一直匹配到结尾

还有就是匹配类似`;key=value`的正则对象，是根据`RFC 2616`标准创建的，这个规则还是非常复杂的，具体规范及标准如下：

```js
/**
 * RFC 2616版本标准的语法
 *
 * parameter     = token "=" ( token | quoted-string )
 * token         = 1*<any CHAR except CTLs or separators>
 * separators    = "(" | ")" | "<" | ">" | "@"
 *               | "," | ";" | ":" | "\" | <">
 *               | "/" | "[" | "]" | "?" | "="
 *               | "{" | "}" | SP | HT
 * quoted-string = ( <"> *(qdtext | quoted-pair ) <"> )
 * qdtext        = <any TEXT except <">>
 * quoted-pair   = "\" CHAR
 * CHAR          = <any US-ASCII character (octets 0 - 127)>
 * TEXT          = <any OCTET except CTLs, but including LWS>
 * LWS           = [CRLF] 1*( SP | HT )
 * CRLF          = CR LF
 * CR            = <US-ASCII CR, carriage return (13)>
 * LF            = <US-ASCII LF, linefeed (10)>
 * SP            = <US-ASCII SP, space (32)>
 * HT            = <US-ASCII HT, horizontal-tab (9)>
 * CTL           = <any US-ASCII control character (octets 0 - 31) and DEL (127)>
 * OCTET         = <any 8-bit sequence of data>
 * @private
 */
var PARAM_REGEXP = /;[\x09\x20]*([!#$%&'*+.0-9A-Z^_`a-z|~-]+)[\x09\x20]*=[\x09\x20]*("(?:[\x20!\x23-\x5b\x5d-\x7e\x80-\xff]|\\[\x20-\x7e])*"|[!#$%&'*+.0-9A-Z^_`a-z|~-]+)[\x09\x20]*/g
```

关于的标准可以查看这里的[rfc2616标准文档](https://datatracker.ietf.org/doc/html/rfc2616) https://datatracker.ietf.org/doc/html/rfc2616

最后我们再看看`decodefield`函数的解码逻辑：

```js
/**
 * 解码RFC 5987的值
 * @param {string} str
 * @return {string}
 * @private
 */

function decodefield (str) {
  var match = EXT_VALUE_REGEXP.exec(str)

  if (!match) {
    throw new TypeError('invalid extended field value')
  }

  var charset = match[1].toLowerCase()
  var encoded = match[2]
  var value

  // to binary string
  // 将百分号格式编码的16进制ascii转译成字符格式
  var binary = encoded.replace(HEX_ESCAPE_REPLACE_REGEXP, pdecode)

  switch (charset) {
    case 'iso-8859-1':
      value = getlatin1(binary)
      break
    case 'utf-8':
      value = Buffer.from(binary, 'binary').toString('utf8')
      break
    default:
      throw new TypeError('unsupported charset in extended field')
  }

  return value
}
```

看下匹配解析value的正则表达式:

```js
/**
 * RFC 5987 标准的语法规则
 *
 * ext-value     = charset  "'" [ language ] "'" value-chars
 * charset       = "UTF-8" / "ISO-8859-1" / mime-charset
 * mime-charset  = 1*mime-charsetc
 * mime-charsetc = ALPHA / DIGIT
 *               / "!" / "#" / "$" / "%" / "&"
 *               / "+" / "-" / "^" / "_" / "`"
 *               / "{" / "}" / "~"
 * language      = ( 2*3ALPHA [ extlang ] )
 *               / 4ALPHA
 *               / 5*8ALPHA
 * extlang       = *3( "-" 3ALPHA )
 * value-chars   = *( pct-encoded / attr-char )
 * pct-encoded   = "%" HEXDIG HEXDIG
 * attr-char     = ALPHA / DIGIT
 *               / "!" / "#" / "$" / "&" / "+" / "-" / "."
 *               / "^" / "_" / "`" / "|" / "~"
 * @private
 */

var EXT_VALUE_REGEXP = /^([A-Za-z0-9!#$%&+\-^_`{}~]+)'(?:[A-Za-z]{2,3}(?:-[A-Za-z]{3}){0,3}|[A-Za-z]{4,8}|)'((?:%[0-9A-Fa-f]{2}|[A-Za-z0-9!#$&+.^_`|~-])+)$/
```

更多关于`RFC5987`可以查看里的文档[rfc5987](https://datatracker.ietf.org/doc/html/rfc5987) https://datatracker.ietf.org/doc/html/rfc5987

### 参考

- [unicode字符集列表](http://www.tamasoft.co.jp/en/general-info/unicode.html) http://www.tamasoft.co.jp/en/general-info/unicode.html
- [ascii码列表](https://www.habaijian.com/) https://www.habaijian.com/
- [rfc5987标准](https://datatracker.ietf.org/doc/html/rfc5987) https://datatracker.ietf.org/doc/html/rfc5987
- [ISO-8859-1百科](https://baike.baidu.com/item/ISO-8859-1/7878872?fr=aladdin) https://baike.baidu.com/item/ISO-8859-1/7878872?fr=aladdin
- [聊聊计算机中的编码（Unicode，GBK，ASCII，utf8，utf16，ISO8859-1等）以及乱码问题的解决办法](https://blog.csdn.net/renwotao2009/article/details/51295766)