# @electron/get源码笔记

> version 0.0.0-development
> 愣锤 2022/02/14

[@electron/got](https://www.npmjs.com/package/@electron/get)库主要作用是用于下载electron的，并且可以设置下载时的镜像资源地址等。

## 基本使用

```js
import { downloadArtifact } from '@electron/get';

// NB: Use this syntax within an async function, Node does not have support for
//     top-level await as of Node 12.
const zipFilePath = await downloadArtifact({
  version: '4.0.4',
  platform: 'darwin',
  artifactName: 'electron',
  artifactSuffix: 'symbols',
  arch: 'x64',
});
```

## 源码分析

该库提供`downloadArtifact`和`download`两个方法用于下载electron的资源，并且返回下载后的绝对路径。主要实现逻辑如下图所示：

![image](https://note.youdao.com/yws/res/19002/963182F5FD5640FC8BF643FBACF0418F)

- 格式化`platform`、`arch`、`version`等下载参数信息
- 根据`artifactName`、`version`、`platform`等参数生成要下载文件名及后缀名
- 根据用户设置的文件名、版本号、镜像资源路径参数等参数获取真正的electron的remote资源地址
- 初始化缓存实例，如果之前下载过相同资源，则缓存中会存在记录
- 判断electron的缓存资源是否存在（如果以前下载过则会有缓存记录），则判断是否使用缓存。缓存存在则返回缓存资源的绝对路径
- 否则调用用户自定义的下载函数或该库默认的下载函数，
将资源下载到临时目录
- 将临时目录中的electron资源移动到缓存路径上
- 下载结束

### electron的remote资源地址获取

根据用户设置的镜像资源地址，最终在`process.env`上获取`mirror`的地址，因为可以在多处设置，因此有如下取值顺序：

- NPM_CONFIG_ELECTRON_MIRROR
- npm_config_electron_mirror
- npm_package_config_electron_mirror
- ELECTRON_MIRROR
- 用户参数中设置的mirror值，即opts.mirror
- github上electron的资源地址

具体拼接的逻辑则为：mirror + customDir + fileanme.ext。实现逻辑如下：

```js
/**
 * process.env上设置的镜像相关的值
 */
function mirrorVar(
  name: keyof Omit<MirrorOptions, 'resolveAssetURL'>,
  options: MirrorOptions,
  defaultValue: string,
): string {
  // Convert camelCase to camel_case for env var reading
  const lowerName = name.replace(/([a-z])([A-Z])/g, (_, a, b) => `${a}_${b}`).toLowerCase();

  return (
    process.env[`NPM_CONFIG_ELECTRON_${lowerName.toUpperCase()}`] ||
    process.env[`npm_config_electron_${lowerName}`] ||
    process.env[`npm_package_config_electron_${lowerName}`] ||
    process.env[`ELECTRON_${lowerName.toUpperCase()}`] ||
    options[name] ||
    defaultValue
  );
}

/**
 * 拼接要下载electron时的remote资源地址，
 * 主要用于镜像等参数的设置
 */
export async function getArtifactRemoteURL(details: ElectronArtifactDetails): Promise<string> {
  const opts: MirrorOptions = details.mirrorOptions || {};
  /**
   * 获取process.env上设置的electron相关镜像值，取值顺序依次为：
   *  - NPM_CONFIG_ELECTRON_MIRROR
   *  - npm_config_electron_mirror
   *  - npm_package_config_electron_mirror
   *  - ELECTRON_MIRROR
   *  - 用户参数中设置的mirror值，即opts.mirror
   *  - github上electron的资源地址
   */
  let base = mirrorVar('mirror', opts, BASE_URL);
  if (details.version.includes('nightly')) {
    const nightlyDeprecated = mirrorVar('nightly_mirror', opts, '');
    if (nightlyDeprecated) {
      base = nightlyDeprecated;
      console.warn(`nightly_mirror is deprecated, please use nightlyMirror`);
    } else {
      base = mirrorVar('nightlyMirror', opts, NIGHTLY_BASE_URL);
    }
  }

  // 获取customDir的值，取值逻辑和上述的mirror一样
  // 然后把值中的'{{ version }}'字符串替换成用户参数中的version值
  // 即mirrorOptions.version
  const path = mirrorVar('customDir', opts, details.version).replace(
    '{{ version }}',
    details.version.replace(/^v/, ''),
  );
  // 获取文件名称加后缀
  // customFilename的取值逻辑和mirror一样
  const file = mirrorVar('customFilename', opts, getArtifactFileName(details));

  // Allow customized download URL resolution.
  // 如果用户设置了resolveAssetURL字段，
  // 则直接使用该字段的值作为远程下载的url地址
  if (opts.resolveAssetURL) {
    const url = await opts.resolveAssetURL(details);
    return url;
  }

  // 否则拼接处理后的mirror+customDir+customFilename的值作为remote下载url
  return `${base}${path}/${file}`;
}
```

### 下载逻辑

下载逻辑优先调用用户的自定义下载函数，否则使用默认的下载函数：

```js
// 指定用于下载的函数，优先取用户自定义的
const downloader = artifactDetails.downloader || (await getDownloaderForSystem());
// 调用download进行electron的remote资源的流下载和本地流写入
await downloader.download(url, tempDownloadPath, artifactDetails.downloadOptions);
```

默认的下载函数实现如下：

```js
export async function getDownloaderForSystem(): Promise<Downloader<DownloadOptions>> {
  const { GotDownloader } = await import('./GotDownloader');
  return new GotDownloader();
}


/**
 * 默认的electron资源下载逻辑
 * 核心逻辑如下：
 *  - 创建目标文件
 *  - 创建fs.createWriteStream可写流
 *  - 利用got进行remote资源流下载
 *  - got边下载，可写流边写入
 *  - 如果30s没有完成下载，则根据参数展示进度条
 */
export class GotDownloader implements Downloader<GotDownloaderOptions> {
  async download(
    url: string,
    targetFilePath: string,
    options?: GotDownloaderOptions,
  ): Promise<void> {
    if (!options) {
      options = {};
    }
    const { quiet, getProgressCallback, ...gotOptions } = options;
    let downloadCompleted = false;
    let bar: ProgressBar | undefined;
    let progressPercent: number;
    let timeout: NodeJS.Timeout | undefined = undefined;
    // 文件夹不存在则创建
    await fs.mkdirp(path.dirname(targetFilePath));
    // 创建可写流
    const writeStream = fs.createWriteStream(targetFilePath);

    // 如果设置展示进度条，则在下载30s后初始化下载进度的进度条
    if (!quiet || !process.env.ELECTRON_GET_NO_PROGRESS) {
      const start = new Date();
      timeout = setTimeout(() => {
        if (!downloadCompleted) {
          bar = new ProgressBar(
            `Downloading ${path.basename(url)}: [:bar] :percent ETA: :eta seconds `,
            {
              curr: progressPercent,
              total: 100,
            },
          );
          // https://github.com/visionmedia/node-progress/issues/159
          // eslint-disable-next-line @typescript-eslint/no-explicit-any
          (bar as any).start = start;
        }
      }, PROGRESS_BAR_DELAY_IN_SECONDS * 1000);
    }
    await new Promise((resolve, reject) => {
      // 利用got库进行资源流下载
      const downloadStream = got.stream(url, gotOptions);
      // 监听下载进度，更新下载的进度条
      downloadStream.on('downloadProgress', async progress => {
        progressPercent = progress.percent;
        if (bar) {
          bar.update(progress.percent);
        }
        // 如果设置了下载进度的钩子函数，则进行钩子函数调用
        if (getProgressCallback) {
          await getProgressCallback(progress);
        }
      });

      // 下载出错时关闭流
      downloadStream.on('error', error => {
        if (error.name === 'HTTPError' && error.statusCode === 404) {
          error.message += ` for ${error.url}`;
        }
        if (writeStream.destroy) {
          writeStream.destroy(error);
        }

        reject(error);
      });
      writeStream.on('error', error => reject(error));
      writeStream.on('close', () => resolve());

      // 下载的流进行流写入
      downloadStream.pipe(writeStream);
    });

    // 下载成功
    downloadCompleted = true;

    // 如果在30s内完成了下载则清除定时器
    if (timeout) {
      clearTimeout(timeout);
    }
  }
}
```

- 创建临时文件夹
- 利用`fs.createWriteStream`创建可写流
- 非`quiet`模式下，如果超过指定时间（默认30s）还未下载完成，则展示下载进度
- 利用`got.stream`方法使用流下载remote资源，一边下载一边写入流
- 下载成功清除定时器

## 总结

该库的主要实现逻辑在上述架构图中已经很清晰了，总结来说就是根据用户参数获取remote资源路径，利用got+fs边下载边写入到本地临时路径，最后将下载的资源放入缓存中。

其中如下几段代码可以单独拿出来复用：

### 边下载边写入流

```js
const path = require('path');
const got = require('got');
const fs = require('fs-extra');

async function download(remoteUrl, targetPath) {
  let progressPercent;
  const gotOptions = {};

  // 文件夹不存在则创建
  await fs.ensureDir(path.dirname(targetPath));
  // 创建可写流
  const writeStream = fs.createWriteStream(targetPath);

  // 利用got库进行资源流下载
  const downloadStream = got.stream(remoteUrl, gotOptions);
  // 这里可以监听下载进度
  downloadStream.on('downloadProgress', async progress => {
    progressPercent = progress.percent;
    // 如果设置了下载进度的钩子函数，则进行钩子函数调用
    // if (options.getProgressCallback) {
    //   await options.getProgressCallback(progress);
    // }
  });

  // 下载出错时关闭流
  downloadStream.on('error', error => {
    if (error.name === 'HTTPError' && error.statusCode === 404) {
      error.message += ` for ${error.url}`;
    }
    if (writeStream.destroy) {
      writeStream.destroy(error);
    }
  });

  writeStream.on('error', error => {});
  writeStream.on('close', () => {
    // 下载成功，比如此处可以resolve等逻辑
  });

  // 下载的流进行流写入
  downloadStream.pipe(writeStream);
}
```

### 使用临时目录并删除的代码段

```js
// 使用并删除文件夹目录
async function useAndRemoveDirectory<T>(
  directory: string,
  fn: (directory: string) => Promise<T>,
): Promise<T> {
  let result: T;
  try {
    result = await fn(directory);
  } finally {
    await fs.remove(directory);
  }
  return result;
}
```