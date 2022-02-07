
# path-to-regexp源码解析

> version 6.2.0
> 愣锤 2022/02/07

## 使用介绍

> version 6.2.0

[path-to-regexp](https://github.com/pillarjs/path-to-regexp#readme)主要作用是将字符串路径（例如'/user/:name'）转换成对应的正则表达式。下面是使用示例：

```js
const { pathToRegexp, parse, compile } = require('path-to-regexp');

const url = '/user/:id';

const keys = [];
const regexp = pathToRegexp(url, keys);

// /^\/user(?:\/([^\/#\?]+?))[\/#\?]?$/i
console.log(regexp);

/**
 * [
 *  {
 *    name: 'id',
 *    prefix: '/',
 *    suffix: '',
 *    pattern: '[^\\/#\\?]+?',
 *    modifier: ''
 *  }
 * ]
 */
console.log(keys);

// ['/user/10086', '10086', index: 0, input: '/user/10086', groups: undefined]
console.log(regexp.exec('/user/10086'));

// null
console.log(regexp.exec('/notuser/10086'));

const tokens = parse(url);

/**
 * [
 *  '/user',
 *  {
 *    name: 'id',
 *    prefix: '/',
 *    suffix: '',
 *    pattern: '[^\\/#\\?]+?',
 *    modifier: ''
 *  }
 * ]
 */
console.log(tokens);

const toPath = compile(url, {
  encode: encodeURIComponent,
});

const path1 = toPath({
  id: 123,
});

// /user/123
console.log(path1);
```

## 原理解析

该库的所有实现都在src文件夹下的index.ts文件。

![image](https://note.youdao.com/yws/res/18760/6AF0143CD990480EBB106D56E632C96F)

index.ts中主要实现几个函数并对外暴露：

```js
export function parse() {}

export function compile<P extends object = object>() {}

export function tokensToFunction<P extends object = object>() {}

export function match<P extends object = object>() {}

export function regexpToFunction<P extends object = object>() {}

export function regexpToFunction<P extends object = object>() {}

export function tokensToRegexp() {}

export function pathToRegexp() {}
```

### pathToRegexp实现

`pathToRegexp`函数是我们主要使用的，将路径字符串转换成正则对象的实现：

```js
/**
 * Normalize the given path string, returning a regular expression.
 *
 * An empty array can be passed in for the keys, which will hold the
 * placeholder key descriptions. For example, using `/user/:id`, `keys` will
 * contain `[{ name: 'id', delimiter: '/', optional: false, repeat: false }]`.
 */
export function pathToRegexp(
  path: Path,
  keys?: Key[],
  options?: TokensToRegexpOptions & ParseOptions
) {
  /**
   * 一个策略组，根据传入的path参数类型，调用不同的实现
   *  - path是正则对象，则调用regexpToRegexp转换
   *  - path是数组，则调用arrayToRegexp转换
   *  - path是字符串，则调用stringToRegexp转换
   */
  if (path instanceof RegExp) return regexpToRegexp(path, keys);
  if (Array.isArray(path)) return arrayToRegexp(path, keys, options);
  return stringToRegexp(path, keys, options);
}
```

`stringToRegexp`的实现，该方法是对路径字符串的转换逻辑：

```js
/**
 * Create a path regexp from string input.
 */
function stringToRegexp(
  path: string,
  keys?: Key[],
  options?: TokensToRegexpOptions & ParseOptions
) {
  // 先使用parse处理成需要的tokens
  // 然后调用tokensToRegexp将tokens转换成正则对象
  return tokensToRegexp(parse(path, options), keys, options);
}
```

`parse`的实现，先调用lexer对字符串进行词法分析，然后进行语法分析，将语法分析得到的结果输出返回：

```js
/**
 * Parse a string for the raw tokens.
 */
export function parse(str: string, options: ParseOptions = {}): Token[] {
  // 先进行字符分割，也就是词法分析
  const tokens = lexer(str);
  const { prefixes = "./" } = options;
  const defaultPattern = `[^${escapeString(options.delimiter || "/#?")}]+?`;
  const result: Token[] = [];
  let key = 0;
  let i = 0;
  let path = "";

  const tryConsume = (type: LexToken["type"]): string | undefined => {
    if (i < tokens.length && tokens[i].type === type) return tokens[i++].value;
  };

  const mustConsume = (type: LexToken["type"]): string => {
    const value = tryConsume(type);
    if (value !== undefined) return value;
    const { type: nextType, index } = tokens[i];
    throw new TypeError(`Unexpected ${nextType} at ${index}, expected ${type}`);
  };

  const consumeText = (): string => {
    let result = "";
    let value: string | undefined;
    // tslint:disable-next-line
    // 将多个CHAR或者ESCAPED_CHAR类型的token组成一个连续的字符串
    while ((value = tryConsume("CHAR") || tryConsume("ESCAPED_CHAR"))) {
      result += value;
    }
    return result;
  };

  // 将得词法分析到的tokens，进行语法分析
  while (i < tokens.length) {
    const char = tryConsume("CHAR");
    const name = tryConsume("NAME");
    const pattern = tryConsume("PATTERN");

    // 处理NAME或PATTERN类型的token
    if (name || pattern) {
      let prefix = char || "";

      if (prefixes.indexOf(prefix) === -1) {
        path += prefix;
        prefix = "";
      }

      if (path) {
        result.push(path);
        path = "";
      }

      // 添加到解析结果中
      result.push({
        name: name || key++,
        prefix,
        suffix: "",
        pattern: pattern || defaultPattern,
        modifier: tryConsume("MODIFIER") || ""
      });
      continue;
    }

    // 处理CHAR或ESCAPED_CHAR类型的token
    const value = char || tryConsume("ESCAPED_CHAR");
    // 一直匹配到非（CHAR或ESCAPED_CHAR）类型的token停止
    if (value) {
      path += value;
      continue;
    }
    // 将匹配到的结果添加到解析结果中，并且置空本次的匹配结果
    if (path) {
      result.push(path);
      path = "";
    }

    // 处理OPEN和CLOSE类型的token
    // 例如处理 const regexp = pathToRegexp("/:attr1?{-:attr2}?");
    const open = tryConsume("OPEN");
    if (open) {
      const prefix = consumeText();
      const name = tryConsume("NAME") || "";
      const pattern = tryConsume("PATTERN") || "";
      const suffix = consumeText();

      mustConsume("CLOSE");

      result.push({
        name: name || (pattern ? key++ : ""),
        pattern: name && !pattern ? defaultPattern : pattern,
        prefix,
        suffix,
        modifier: tryConsume("MODIFIER") || ""
      });
      continue;
    }

    mustConsume("END");
  }

  return result;
}
```

可以看到`parse`的过程就是先通过`lexer`进行词法分析拿到所有的`tokens`，然后就是消费`tokens`。消费的过程就是迭代所有的`tokens`，消费的依据就是该库期望对外暴露的语法规则。

`lexer`词法分析主要作用就是按规则分割出`tokens`，具体实现如下：

```js
/**
 * Tokenize input string.
 * 词法分割
 */
function lexer(str: string): LexToken[] {
  const tokens: LexToken[] = [];
  let i = 0;

  // 依次迭代每一个字符
  while (i < str.length) {
    // 获取当前字符
    const char = str[i];

    // 如果是星号、加号、问号，则分割为MODIFIER类型的token
    if (char === "*" || char === "+" || char === "?") {
      tokens.push({ type: "MODIFIER", index: i, value: str[i++] });
      continue;
    }

    // 如果是\符号，则分割为ESCAPED_CHAR类型的token
    if (char === "\\") {
      tokens.push({ type: "ESCAPED_CHAR", index: i++, value: str[i++] });
      continue;
    }

    // 如果是左花括号，则分割为OPEN类型的token
    if (char === "{") {
      tokens.push({ type: "OPEN", index: i, value: str[i++] });
      continue;
    }

    // 如果是右花括号，则分割为CLOSE类型的token
    if (char === "}") {
      tokens.push({ type: "CLOSE", index: i, value: str[i++] });
      continue;
    }

    // 如果是冒号，则继续分割冒号后面的字符串
    if (char === ":") {
      let name = "";
      let j = i + 1;

      // 通过字符对应的Unicode值匹配所有的数字、英文大小写、连字符
      // 通过正则匹配也可以，/^[0-9a-zA-Z-]$/，经测试性能并不比Unicode判断差
      while (j < str.length) {
        const code = str.charCodeAt(j);

        if (
          // `0-9`
          (code >= 48 && code <= 57) ||
          // `A-Z`
          (code >= 65 && code <= 90) ||
          // `a-z`
          (code >= 97 && code <= 122) ||
          // `_`
          code === 95
        ) {
          name += str[j++];
          continue;
        }

        break;
      }

      if (!name) throw new TypeError(`Missing parameter name at ${i}`);

      // 将冒号后面匹配到的符合规则的字符串，分割为NAME类型的token
      tokens.push({ type: "NAME", index: i, value: name });
      i = j;
      continue;
    }

    // 如果当前字符是左小括号
    if (char === "(") {
      /**
       * count 左右小括号的计数，是根据栈来判断左右小括号是否匹配的平替方案
       *  - 遇到左括号加一
       *  - 遇到右括号减一
       * 最终根据count的值判断左右小括号是否正确匹配
       */
      let count = 1;
      let pattern = "";
      let j = i + 1;

      if (str[j] === "?") {
        throw new TypeError(`Pattern cannot start with "?" at ${j}`);
      }

      while (j < str.length) {
        // 如果是\开头的字符则获取\加上后面的一个字符,例如
        if (str[j] === "\\") {
          pattern += str[j++] + str[j++];
          continue;
        }

        if (str[j] === ")") {
          // 计数减一
          count--;
          // 如果已完成所有左右小括号的匹配，则停止当前token字符匹配
          if (count === 0) {
            j++;
            break;
          }
        } else if (str[j] === "(") {
          count++;
          // (user(?xxx)) 要求捕获组必须要问号开头
          if (str[j + 1] !== "?") {
            throw new TypeError(`Capturing groups are not allowed at ${j}`);
          }
        }

        // 匹配符合规则的字符串
        pattern += str[j++];
      }

      if (count) throw new TypeError(`Unbalanced pattern at ${i}`);
      if (!pattern) throw new TypeError(`Missing pattern at ${i}`);

      // 将(pattern)内的pattern部分分割为PATTERN类型的token
      tokens.push({ type: "PATTERN", index: i, value: pattern });
      i = j;
      continue;
    }

    // 其他字符分割为类型为CHAR的token
    tokens.push({ type: "CHAR", index: i, value: str[i++] });
  }

  // 最后添加一个类型为END的token
  tokens.push({ type: "END", index: i, value: "" });

  return tokens;
}
```

主要逻辑如下：

- 逐个遍历字符串的字符
- 给不同的字符打上不同的标记，例如`MODIFIER、CHAR`等
- 根据不同的字符进行不同规则的匹配
- 将每个规则匹配到的字符数据存入`tokens`数组
- 将结果返回

需要注意的是：
- 这里的词法分割并没有使用有限状态机，而是遍历后就直接消费了。
- 匹配冒号后面的字符串时用的Unicode值比对方法，平替成正则`/^[0-9a-zA-Z-]$/`也是可以的，经个人测试，性能并不比Unicode判断差
- lexer中判断左右小括号是否正确匹配的逻辑，直接使用的count计数来判断，同样的实现也有栈的方案。
- `i++`自增自减的逻辑，加号在前在后的区别，在普通使用中没有任何区别，但是**在赋值时则是加号在前先自增再赋值，加号在后先赋值再自增**

### arrayToRegexp实现

`pathToRegexp`方法中如果path是数组则调用    arrayToRegexp   来具体实现，其作用是传入数组时生成的正则是可以匹配多个逻辑，也就是**或**的意思:

```js
/**
 * Transform an array into a regexp.
 */
function arrayToRegexp(
  paths: Array<string | RegExp>,
  keys?: Key[],
  options?: TokensToRegexpOptions & ParseOptions
): RegExp {
  const parts = paths.map(path => pathToRegexp(path, keys, options).source);
  return new RegExp(`(?:${parts.join("|")})`, flags(options));
}
```

实现逻辑：

- 遍历所有的`path`并调用`pathToRegexp`获取`path`对应的正则表达式文本
- 将所有的文本用`|`拼接起来
- 重新调用`new RegExp`生成新的正则对象

补充：

- 正则中小括号`()`表示捕获组，就是将匹配到的内容存储起来以供使用
- `(?:)`表示非捕获组，即只进行匹配，不对匹配的结果进行存储