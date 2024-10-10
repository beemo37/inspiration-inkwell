<!-- omit from toc -->
# Node

- [安装](#安装)
- [fnm](#fnm)
  - [通过fnm安装Node](#通过fnm安装node)
    - [Windows](#windows)
    - [macOS](#macos)
  - [指令](#指令)
- [nvm](#nvm)
  - [通过nvm安装Node](#通过nvm安装node)
- [npm](#npm)
  - [错误解决](#错误解决)


TODO 介绍

[官网](https://nodejs.org/en)
## 安装
<!-- omit from toc-->

- [安装](#安装)
- [fnm](#fnm)
  - [通过fnm安装Node](#通过fnm安装node)
    - [Windows](#windows)
    - [macOS](#macos)
  - [指令](#指令)
- [nvm](#nvm)
  - [通过nvm安装Node](#通过nvm安装node)
- [npm](#npm)
  - [错误解决](#错误解决)


## fnm

[fnm](https://github.com/Schniz/fnm)全称是`Faster Node Manager`，顾名思义，目标是成为更快的`Node`管理器。

特点：

- 跨平台（macOS、Windows、Linux）
- 单文件，安装简单，启动迅速、
- 以速度为设计核心
- 支持.node-version和.nvmrc文件

### 通过fnm安装Node

#### Windows

1. 下载fnm

```shell
winget install Schniz.fnm
```

2. 重新打开命令行工具，配置fnm环境变量

```shell
# shell
FOR /f "tokens=*" %i IN ('fnm env --use-on-cd') DO CALL %i
# PowerShell
fnm env --use-on-cd | Out-String | Invoke-Expression
```

3. 下载Node

```shell
fnm use --install-if-missing <NODE_VERSION>
```

4. 测试是否成功

```shell
node -v
npm -v
```

#### macOS

1. 下载fnm

通过`curl -fsSL https://fnm.vercel.app/install | bash`的方式经常超时，访问不了该网址，所以我们使用`brew`进行安装。

```shell
brew install fnm
```
对于高版本macOS来说，使用brew时会如下错误：

```shell
Error: unknown or unsupported macOS version: :dunno
```

此时需要更新一下brew

```shell
brew update 
```

之后再执行`brew install fnm`即可。

最后检验一下是否安装成功

```shell
fnm --version
```
2. 下载nodejs
   
```shell
fnm install <VERSION>
或
# 下载最新lts版本
fnm install --lts
```

但是此时可能又碰到nodejs.org无法访问的问题，此时需要配置镜像地址，已使用`zsh`为例，执行以下指令修改配置文件：

```shell
vim ~/.zshrc
```

之后添加如下配置

```txt
export FNM_NODE_DIST_MIRROR=https://npmmirror.com/mirrors/node/
```

最后刷新配置文件即可

```shell
source ~/.zshrc
```

之后执行安装指令
```shell
fnm install --lts
```

3. 添加fnm环境变量
   
此时虽然node下载完毕，但是fnm相关指令无法使用，并且如果你通过其他方式安装了node，此时使用的并不是我们下载的版本。配置方式如下：

修改zsh配置文件

```shell
vim ~/.zshrc
```

之后添加如下配置

```txt
eval "$(fnm env --use-on-cd --shell zsh)"
```

最后刷新配置文件即可

```shell
source ~/.zshrc
```

4. 检查是否安装成功

```shell
node -v
npm -v
```

5. 如果安装的nodejs版本为20+，那么还需要禁用OpenSSL 3，否则在启动服务的时候会报错。方式如下

修改zsh配置文件

```shell
vim ~/.zshrc
```

之后添加如下配置

```txt
export NODE_OPTIONS=--openssl-legacy-provider
```

最后刷新配置文件即可

```shell
source ~/.zshrc
```

### 指令

- 查看已经安装的node列表

  ```shell
  fnm ls
  ```

- 安装node

  ```shell
  fnm install <NODE_VERSION>
  ```
- 使用指定版本的node

  ```shell
  use <NODE_VERSION>
  ```

- 查看当前使用的node版本

  ```shell
  fnm current
  ```

- 设置默认的node版本

  ```shell
  fnm default <NODE_VERSION>
  ```

## nvm

### 通过nvm安装Node

1. 下载nvm
   
```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

2. 下载node

```shell
nvm install <NODE_VERSION>
```

3. 测试是否成功

```shell
node -v
npm -v
```
## npm

### 错误解决

1. 在`Windows`下执行`npm install`时出现以下错误：
> MSBUILD : error MSB4132: The tools version "2.0" is unrecognized. Available tools versions are "4.0".

此时需要使用**管理员权限**打开`PownerShell`，安装`windows-build-tools`：

```shell
npm install --global --production windows-build-tools
```

注意，可能会自动下载Python2，可以先提前安装。最后会卡在以下位置：

```shell
Status from the installers:
---------- Visual Studio Build Tools ----------
Still waiting for installer log file...
------------------- Python --------------------
Python 2.7.18 is already installed, not installing again.
```

此时可以关掉，之后重新执行`npm install`就可以了。

2. 执行`npm install`时出现以下错误：

```shell
...

npm error gyp verb install --ensure was passed, so won't reinstall if already installed
npm error gyp verb install version not already installed, continuing with install 20.17.0
npm error gyp verb ensuring nodedir is created /Users/beemo/.node-gyp/20.17.0
npm error gyp verb created nodedir /Users/beemo/.node-gyp/20.17.0
npm error gyp http GET https://nodejs.org/download/release/v20.17.0/node-v20.17.0-headers.tar.gz
npm error gyp WARN install got an error, rolling back install
npm error gyp verb command remove [ '20.17.0' ]
npm error gyp verb remove using node-gyp dir: /Users/beemo/.node-gyp
npm error gyp verb remove removing target version: 20.17.0
npm error gyp verb remove removing development files for version: 20.17.0
npm error gyp ERR! configure error 
npm error gyp ERR! stack AggregateError [ETIMEDOUT]: 
npm error gyp ERR! stack     at internalConnectMultiple (node:net:1118:18)
npm error gyp ERR! stack     at afterConnectMultiple (node:net:1685:7)

...
```

根据日志可以看出原因是访问`https://nodejs.org/download/release/v20.17.0/node-v20.17.0-headers.tar.gz`时结果为`ETIMEDOUT`，即超时。

此时我们可以手动下载头文件至指定位置，类似于手动下载jar至maven指定的目录，这样就不会再去远程下载了。

如，可以访问相关仓库源，我这里使用的是华为云 `https://repo.huaweicloud.com/nodejs/v20.16.0/`，进行下载。（应该下载20.17.0，但是只有20.16.0，就先使用这个）

下载完毕后无需解压，将此压缩包复制到`~/.node-gyp/<VERSION>文件夹下即可，如:
```shell
cp /Users/beemo/Downloads/node-v20.16.0-headers.tar.gz ~/.node-gyp/20.17.0/node-v20.17.0-headers.tar.gz
```

最后重新执行`npm run`，此时就不会发生错误了。

3. nodejs 20+版本执行 `npm run serve`时报错：

```shell
Error: error:0308010C:digital envelope routines::unsupported
    at new Hash (node:internal/crypto/hash:79:19)
    at Object.createHash (node:crypto:139:10)
    at module.exports (/Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/webpack/lib/util/createHash.js:135:53)
    at NormalModule._initBuildHash (/Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/webpack/lib/NormalModule.js:417:16)
    at handleParseError (/Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/webpack/lib/NormalModule.js:471:10)
    at /Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/webpack/lib/NormalModule.js:503:5
    at /Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/webpack/lib/NormalModule.js:358:12
    at /Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/loader-runner/lib/LoaderRunner.js:373:3
    at iterateNormalLoaders (/Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/loader-runner/lib/LoaderRunner.js:214:10)
    at Array.<anonymous> (/Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/loader-runner/lib/LoaderRunner.js:205:4)
    at Storage.finished (/Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/enhanced-resolve/lib/CachedInputFileSystem.js:55:16)
    at /Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/enhanced-resolve/lib/CachedInputFileSystem.js:91:9
    at /Users/beemo/work/workspace/heading/zl-crm/nightfury-web-ui/node_modules/graceful-fs/graceful-fs.js:123:16
    at FSReqCallback.readFileAfterClose [as oncomplete] (node:internal/fs/read/context:68:3) {
  opensslErrorStack: [
    'error:03000086:digital envelope routines::initialization error',
    'error:0308010C:digital envelope routines::unsupported'
  ],
  library: 'digital envelope routines',
  reason: 'unsupported',
  code: 'ERR_OSSL_EVP_UNSUPPORTED'

```

原因为Node.js 20.x 版本对 OpenSSL 3 的不兼容，此时可以将其禁用，方法如下：

修改zsh配置文件

```shell
vim ~/.zshrc
```

之后添加如下配置

```txt
export NODE_OPTIONS=--openssl-legacy-provider
```

最后刷新配置文件即可

```shell
source ~/.zshrc
```