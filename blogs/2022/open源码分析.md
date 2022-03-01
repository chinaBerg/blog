# open源码分析

> version 8.4.0
> 愣锤 2022/03/01

[open](https://github.com/sindresorhus/open#readme)是一个跨平台的用于打开URL、文件、可执行文件的库。

```js
// 使用默认浏览器打开百度
open('https://www.baidu.com');

// 使用火狐浏览器打开百度
open('https://www.baidu.com', {
  app: {
    name: 'firefox',
  },
});
```

### 源码分析

操作系统中提供了唤起app的命令，比如MacOS、IOS中可以在终端使用open命令唤起app，windows系统中使用利用PowerShell命令借助powershell唤起app:

```bash
# mac的terminal终端
open https://www.baidu.com # 使用默认浏览器打开百度
open -a "google chrome" https://www.baidu.com # 指定谷歌浏览器打开百度

# windows系统终端
PowerShell Start https://www.baidu.com # 使用默认浏览器打开百度
```

因此，该库的核心实现就是判断不同的操作系统，利用node的子进程执行不同的系统命令：

```js
/**
 * 对外暴露的方法，打开程序
 */
const open = (target, options) => {
	if (typeof target !== 'string') {
		throw new TypeError('Expected a `target`');
	}

	return baseOpen({
		...options,
		target
	});
};

module.exports = open;
```

下面看下baseOpen的实现：

```js
const { platform, arch } = process;

const baseOpen = async options => {
  // 合并初始化参数,省略部分参数初始化的代码
  options = {
	wait: false,
	background: false,
	newInstance: false,
	allowNonzeroExitCode: false,
	...options
  };
	
  let command;
  const cliArguments = [];
  const childProcessOptions = {};
	
  /**
   * MacOS、IOS系统
   */
  if (platform === 'darwin') {
    // ....
  }
	
  /**
   * windows系统、linux下的windows系统
   */
  else if (platform === 'win32' || (isWsl && !isDocker())) {
    // ....
  }
	
  // linux或其他系统
  else {}

  if (options.target) {
	cliArguments.push(options.target);
  }

  if (platform === 'darwin' && appArguments.length > 0) {
	cliArguments.push('--args', ...appArguments);
  }
	
  // 利用spawn开启一个子进程，执行command命令，传入对应的cliArguments参数
  const subprocess = childProcess.spawn(command, cliArguments, childProcessOptions);

  if (options.wait) {
    // ...
  }

  // 父进程不再等待子进程的退出，而是父进程独立于子进程进行退出
  subprocess.unref();

  return subprocess;
}
```

注意，spawn的子进程开启代码：

```js
const subprocess = childProcess.spawn(command, cliArguments, {
  // 忽略子进程中的文件描述符
  stdio: 'ignore',
  // 让子进程在父进程退出后继续运行
  detached: true,
});

// 父进程不再等待子进程退出
subprocess.unref();
```

这里要注意的是，参数`detached: true,`让子进程在父进程退出后继续运行，`subprocess.unref();`让父进程不再等待子进程退出再退出。

- windows中的参数实现：

```js
/**
 * windows系统、linux下的windows系统
 */
else if (platform === 'win32' || (isWsl && !isDocker())) {
  // 获取linux下的windows系统的驱动器入口地址
  const mountPoint = await getWslDrivesMountPoint();

  /**
   * 根据不同的windows系统获取powershell程序的路径
   *  - isWsl判断是否是在linux系统的windows子系统中运行的进程
   *  - process.env.SYSTEMROOT用于获取系统路径， EG: C:\Windows
   */
    command = isWsl ?
		`${mountPoint}c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe` :
		`${process.env.SYSTEMROOT}\\System32\\WindowsPowerShell\\v1.0\\powershell`;

  /**
   * 可以在cmd终端输入PowserShell -Help查看相关参数的帮助文档：
   * -NoProfile 不使用用户配置文件
   * -NonInteractive 不向用户显示交互式提示
   * –ExecutionPolicy Bypass 设置会话的默认执行策略为Bypass
   * -EncodedCommand 命令接受Base64的编码字符串。用于支持参数中使用一些特殊字符
   */
	cliArguments.push(
		'-NoProfile',
		'-NonInteractive',
		'–ExecutionPolicy',
		'Bypass',
		'-EncodedCommand'
	);

	if (!isWsl) {
		childProcessOptions.windowsVerbatimArguments = true;
	}

    // 需要编码的参数
	const encodedArguments = ['Start'];

	if (options.wait) {
		encodedArguments.push('-Wait');
	}

    // 如果指定了启动的app，则使用指定的app，否则使用默认的启动程序
    // 当Start命令后面直接拼接options.target参数时，此时因为没有指定app，所以系统会使用默认的启动程序
	if (app) {
		// Double quote with double quotes to ensure the inner quotes are passed through.
		// Inner quotes are delimited for PowerShell interpretation with backticks.
		encodedArguments.push(`"\`"${app}\`""`, '-ArgumentList');
		if (options.target) {
			appArguments.unshift(options.target);
		}
	} else if (options.target) {
		encodedArguments.push(`"${options.target}"`);
	}

	if (appArguments.length > 0) {
		appArguments = appArguments.map(arg => `"\`"${arg}\`""`);
		encodedArguments.push(appArguments.join(','));
	}

	// Using Base64-encoded command, accepted by PowerShell, to allow special characters.
    // 对需要编码的参数使用Buffer转换成base64的字符串
	options.target = Buffer.from(encodedArguments.join(' '), 'utf16le').toString('base64');
}
```

注意，windows系统中通过通过`powershell`程序打开app时传入的参数`-EncodedCommand`可以让参数执行base64的数据，这样便可以支持复杂的字符。`base64`字符数据的转换利用了`Buffer`对象。

- MacOS系统下的实现

```js
/**
 * MacOS、IOS系统
 */
if (platform === 'darwin') {
	command = 'open';

	if (options.wait) {
		cliArguments.push('--wait-apps');
	}

	if (options.background) {
		cliArguments.push('--background');
	}

	if (options.newInstance) {
		cliArguments.push('--new');
	}

	if (app) {
		cliArguments.push('-a', app);
	}
}
```

同样也是指定要运行的命令和拼接参数，没什么特殊注意的点。

总结一个open库原理的基本实现如下：


```js
const { spawn } = require('child_process');

// console.log(process.env.SYSTEMROOT)
// console.log(os.homedir())
// spawn('PowerShell', ['Start', 'https://www.baidu.com']);


const { platform } = process;

function open(target) {
  let command;
  const commandArgs = [];

  if (platform === 'darwin') {
    command = 'open';
    commandArgs.push(target);
  } else if (platform === 'win32') {
    const startArgs = [];
    command = 'PowerShell';
    commandArgs.push(
      '-NoProfile',
			'-NonInteractive',
			'–ExecutionPolicy',
			'Bypass',
			'-EncodedCommand',
    );
    startArgs.push('Start');
    startArgs.push(target);
    commandArgs.push(
      Buffer.from(startArgs.join(' '), 'utf16le').toString('base64'),
    );
  }

  spawn(command, commandArgs).unref();
}

// 测试，唤起效果
open('https://www.baidu.com');
```


### 总结

open库实现打开应用的原理，都是利用node子进程直接或间接执行系统的启动命令：
- `windows`系统或者`linux`下的`windows`系统，利用`powershell`程序传入`Start`命令和启动参数进行app唤起。（直接利用`PowerShell`命令也可以）
- `MacOS`和`IOS`系统下利用`open`系统命令
