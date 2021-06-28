---
title: "Golang：交叉编译"
date: 2021-05-06T19:38:41+08:00
categories:
- 速查
- 项目部署
tags:
- golang
- 编译
keywords:
- golang交叉编译
- CGO
comments: false
summary: 与解释性语言不同，golang代码需要经过编译后才能运行。当开发平台和运行平台不同时，要么在运行平台重新编译代码，要么使用交叉编译。这里简单记录一下第二种方法。
#thumbnailImage: //example.com/image.jpg
cover: https://w.wallhaven.cc/full/y8/wallhaven-y85ojk.png
---

<!--more-->

### 什么是交叉编译

交叉编译是指在一个平台上生成另一个平台上的可执行程序。

比如要在Windows系统上生成可以在Linux上运行的程序时，就需要用到交叉编译。

### Golang交叉编译

#### 编译参数

不同的编译参数决定了最后生成的程序能在什么平台上运行，默认为当前编译平台。

`GOOS` 目标平台的系统，常用值：windows、darwin（macOS）、linux

`GOARCH` 目标平台CPU架构， 常用的值 ：amd64、386、arm64、arm

`GOARM ` 只有 `GOARCH` 是 `arm64` 才有效, 表示 `arm` 的版本, 目前只能是 5, 6, 7 其中之一

`CGO_ENABLED` 值为1或0，分别表示开启或关闭CGO，默认开启（值为1）。CGO用于Go调用C代码

`CC` 当支持交叉汇编时(即 `CGO_ENABLED=1`), 编译目标文件使用的 `c` 编译器

`CXX` 当支持交叉汇编时(即 `CGO_ENABLED=1`), 编译目标文件使用的 `c++` 编译器

`AR` 当支持交叉汇编时(即 `CGO_ENABLED=1`), 编译目标文件使用的创建库文件命令

#### Go支持的平台

`GOOS` 和 `GOARCH`的不同组合就代表不同平台，可以使用`go tool dist list`命令查看支持的平台

go 1.16.3 版本支持的平台如下：

- linux
  - 386，amd64，arm，arm64，mips，mips64，mips64le，mipsle，riscv64，ppc64，ppc64le，s390x

- windows
  - 386，amd64，arm

- darwin，ios
  - amd64，arm64

- android
  - 386，amd64，arm，arm64

- freebsd，netbsd
  - 386，amd64，arm，arm64

- openbsd
  - 386，amd64，arm，arm64，mips64

- dragonfly，illumos，solaris
  - amd64

- plan9
  - 386，amd64，arm

- js
  - wasm

- aix
  - ppc64

其中386表示x86 32位，amd64表示x86 64位。

### 具体操作

下面列举几种常见的交叉编译场景的具体命令操作

**Mac 下编译 Linux 和 Windows 64位可执行程序**

```shell
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
```

**Linux 下编译 Mac 和 Windows 64位可执行程序**

```shell
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
```

**Windows 下编译 Mac 和 Linux 64位可执行程序**

```powershell
SET CGO_ENABLED=0
SET GOOS=darwin
SET GOARCH=amd64
go build main.go

SET CGO_ENABLED=0
SET GOOS=linux
SET GOARCH=amd64
go build main.go
```

可以通过将参数写入环境变量，这样就不用每次都要设置参数

### 难点：CGO

上文操作示例都关闭了CGO，但当go项目包含C代码的调用就需要开启CGO，然而这会带来问题。

例如在一个嵌入式开发场景中，需要使用`github.com/mattn/go-sqlite3`驱动sqlite，并且目标平台是arm架构，此时go本身的工具链不足以完成编译。

解决方法就是下载对应的编译工具链

[arm编译工具链官网地址](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads)

以Windows为例，下载`gcc-arm-none-eabi-XXXX-major-win32.zip`

下载完成后解压并将其下`bin`文件夹加入环境变量，最后将编译参数`CC`的值设为`arm-none-eabi-gcc`

这里简单提一下交叉编译工具链的命名规则：

arch [-vendor] [-os] [-(gnu)eabi]

- arch - 体系架构，如ARM，MIPS
- vendor - 工具链提供商
- os - 目标操作系统
- eabi - 嵌入式应用二进制接口（Embedded Application Binary Interface）

