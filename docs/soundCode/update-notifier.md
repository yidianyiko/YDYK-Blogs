---
date: 2021-11-10
title: 【源码第五期】分析update-notifier
tags:
  - 源码
describe: 浅析 update-notifier如何检测npm更新
---

### 1.阅读前准备

曾经`npm install`之后会发现会自动弹出npm依赖包升级的页面![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb24bef414364c2fbd3943aa7fb25e87~tplv-k3u1fbpfcp-zoom-1.image)



现在就来研究下到底是怎么做到的

### 2.源码地址

<https://github.com/yeoman/update-notifier>

### 3.开始阅读

`UpdateNotifier`构造函数中，分为三部分函数：check()检查函数；fetchInfo获取信息函数；notify通知函数

整个代码中引用了很多第三方库

先从`UpdateNotifier`构造函数入手

#### 3.1 构造函数

```
constructor(options = {}) {
  	// 主要是对传入的options对象中的参数进行校验
		this.options = options;
		options.pkg = options.pkg || {};
		options.distTag = options.distTag || 'latest';

		// Reduce pkg to the essential keys. with fallback to deprecated options
		// TODO: Remove deprecated options at some point far into the future
		options.pkg = {
			name: options.pkg.name || options.packageName,
			version: options.pkg.version || options.packageVersion
		};
		// 如果缺少必要 则抛出异常
		if (!options.pkg.name || !options.pkg.version) {
			throw new Error('pkg.name and pkg.version required');
		}

		this.packageName = options.pkg.name;
		this.packageVersion = options.pkg.version;
  	// 检查传入的时间戳，如果不是时间则采用默认时间戳
		this.updateCheckInterval = typeof options.updateCheckInterval === 'number' ? options.updateCheckInterval : ONE_DAY;
		this.disabled = 'NO_UPDATE_NOTIFIER' in process.env ||
			process.env.NODE_ENV === 'test' ||
			process.argv.includes('--no-update-notifier') ||
			isCi();
		this.shouldNotifyInNpmScript = options.shouldNotifyInNpmScript;

		if (!this.disabled) {
			try {
				const ConfigStore = configstore();
				this.config = new ConfigStore(`update-notifier-${this.packageName}`, {
					optOut: false,
					// Init with the current time so the first check is only
					// after the set interval, so not to bother users right away
					lastUpdateCheck: Date.now()
				});
			} catch {
				// Expecting error code EACCES or EPERM
				const message =
					chalk().yellow(format(' %s update check failed ', options.pkg.name)) +
					format('\n Try running with %s or get access ', chalk().cyan('sudo')) +
					'\n to the local update config store via \n' +
					chalk().cyan(format(' sudo chown -R $USER:$(id -gn $USER) %s ', xdgBasedir().config));

				process.on('exit', () => {
					console.error(boxen()(message, {align: 'center'}));
				});
			}
		}
	}
```

#### 3.2 check检查函数

```
check() {
  	// 如果出现以下几种情况时，则直接退出
		if (
			!this.config ||
			this.config.get('optOut') ||
			this.disabled
		) {
			return;
		}
		// 获取包更新信息（第一次获取的时候为undefined）
		this.update = this.config.get('update');
		
		if (this.update) {
			// 如果存在，则赋值最新版本
			this.update.current = this.packageVersion;

			// 并清理缓存
			this.config.delete('update');
		}

		// 如果最后一次获取更新的时间小于用户设置的检查时间 则直接退出
		if (Date.now() - this.config.get('lastUpdateCheck') < this.updateCheckInterval) {
			return;
		}

		// 调用子进程执行check文件
  	// 这里的unref 方法用于断绝与父进程的关系，父进程退出不会造成子进程的退出
		spawn(process.execPath, [path.join(__dirname, 'check.js'), JSON.stringify(this.options)], {
			detached: true,
			stdio: 'ignore'
		}).unref();
	}
```

在进行check函数的执行后，会进入check.js文件的执行，进一步看check.js中又发生了什么

```
let updateNotifier = require('.');

const options = JSON.parse(process.argv[2]);

updateNotifier = new updateNotifier.UpdateNotifier(options);

(async () => {
	setTimeout(process.exit, 1000 * 30);
	// 在这里继续调用updateNotifier中的获取数据的方法
	const update = await updateNotifier.fetchInfo();

	// 并更新最后更新检查时间的时间字段
	updateNotifier.config.set('lastUpdateCheck', Date.now());
	// 如果此时时间不是最新，则会更新为最新
	if (update.type && update.type !== 'latest') {
		updateNotifier.config.set('update', update);
	}

	// Call process exit explicitly to terminate the child process,
	// otherwise the child process will run forever, according to the Node.js docs
	process.exit();
})().catch(error => {
	console.error(error);
	process.exit(1);
});
```

check.js这里主要为开启子进程，获取最新版本信息的步骤，执行完成后将退出子进程

#### 3.3 fetchInfo()获取信息函数

```
async fetchInfo() {
  const {distTag} = this.options;
  // 这里主要通过懒加载的方式执行获取包信息的步骤
  const latest = await latestVersion()(this.packageName, {version: distTag});

  return {
    latest,
    current: this.packageVersion,
    // 这里会做信息的diff
    type: semverDiff()(this.packageVersion, latest) || distTag,
    name: this.packageName
  };
}
```

#### 3.4 notify()通知函数

```
	notify(options) {
		const suppressForNpm = !this.shouldNotifyInNpmScript && isNpm().isNpmOrYarn;
		if (!process.stdout.isTTY || suppressForNpm || !this.update || !semver().gt(this.update.latest, this.update.current)) {
			return this;
		}

		options = {
      // 是否为全局安装
			isGlobal: isInstalledGlobally(),
      // 是否为yarn全局安装
			isYarnGlobal: isYarnGlobal()(),
			...options
		};

		let installCommand;
    // 根据yarn和npm的判断 展示不同的命令指示符给用户
		if (options.isYarnGlobal) {
			installCommand = `yarn global add ${this.packageName}`;
		} else if (options.isGlobal) {
			installCommand = `npm i -g ${this.packageName}`;
		} else if (hasYarn()()) {
			installCommand = `yarn add ${this.packageName}`;
		} else {
			installCommand = `npm i ${this.packageName}`;
		}

		const defaultTemplate = 'Update available ' +
			chalk().dim('{currentVersion}') +
			chalk().reset(' → ') +
			chalk().green('{latestVersion}') +
			' \nRun ' + chalk().cyan('{updateCommand}') + ' to update';
		
		const template = options.message || defaultTemplate;

		options.boxenOptions = options.boxenOptions || {
			padding: 1,
			margin: 1,
			align: 'center',
			borderColor: 'yellow',
			borderStyle: 'round'
		};

		const message = boxen()(
			pupa()(template, {
				packageName: this.packageName,
				currentVersion: this.update.current,
				latestVersion: this.update.latest,
				updateCommand: installCommand
			}),
			options.boxenOptions
		);

		if (options.defer === false) {
			console.error(message);
		} else {
			process.on('exit', () => {
				console.error(message);
			});

			process.on('SIGINT', () => {
				console.error('');
				process.exit();
			});
		}

		return this;
	}
```

###

### 4.总结

整个过程比较简单，主体就为updateNotifier构造函数，其中执行的步骤为：

-   new了一个updateNotifier对象
-   执行check()函数

<!---->

-   -   判断是否需要执行check.js

<!---->

-   执行fetchInfo()函数获取信息
-   set('lastUpdateCheck')设置最后获取更新的时间

<!---->

-   set('update')设置一个需要更新的列表
-   notify()函数，展示给用户目前可更新的依赖包，并根据包类型展示命令符