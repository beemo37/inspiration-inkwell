<!-- omit from toc -->
# Node

- [安装](#安装)
- [fnm](#fnm)
  - [通过fnm安装Node](#通过fnm安装node)
  - [指令](#指令)
- [nvm](#nvm)
  - [通过nvm安装Node](#通过nvm安装node)
- [npm](#npm)
  - [错误解决](#错误解决)


TODO 介绍

[官网](https://nodejs.org/en)
## 安装
<!-- omit from toc-->

- [通过fnm安装Node](#通过fnm安装node)
- [通过nvm安装Node](#通过nvm安装node)


## fnm

[fnm](https://github.com/Schniz/fnm)全称是`Faster Node Manager`，顾名思义，目标是成为更快的`Node`管理器。

特点：

- 跨平台（macOS、Windows、Linux）
- 单文件，安装简单，启动迅速、
- 以速度为设计核心
- 支持.node-version和.nvmrc文件

### 通过fnm安装Node


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