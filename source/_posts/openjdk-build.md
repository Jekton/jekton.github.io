---
title: 个人记录帖 - 编译 OpenJDK 10
date: 2018-07-22 09:32:41
categories: JVM
tags: openjdk
---

本篇文章纯粹是记录个人编译 openjdk10 的过程，不会很详细地说明各个步奏。这里就算是给自己立 flag 吧。

<!--more-->

### 源码下载
```shell
$ git clone git@github.com:unofficial-openjdk/openjdk.git
# 写作时 openjdk10 最新的 tag
$ git checkout -b jdk-10+46 jdk-10+46
```

### 编译

```shell
$ bash ./configure --enable-debug --with-target-bits=64 --with-jobs=8 --disable-warnings-as-errors --with-jvm-variants=server
$ make
```

### 运行

为了方便以后使用，在 `.zshrc` 加入：
```shell
function openjdk_env {
    export JAVA_HOME="$HOME/dev/java/openjdk10/build/macosx-x86_64-normal-server-fastdebug/jdk"
    export PATH="$JAVA_HOME/bin"
}
```
然后还是回到 terminal：
```shell
$ source ~/.zshrc
$ openjdk_env 
$ java -version
openjdk version "10-internal" 2018-03-20
OpenJDK Runtime Environment (fastdebug build 10-internal+0-adhoc.jekton.openjdk10)
OpenJDK 64-Bit Server VM (fastdebug build 10-internal+0-adhoc.jekton.openjdk10, mixed mode)
```


### 使用 idea 阅读 JDK

执行源码下的 `bin/idea.sh`，它会在源码根目录生成 `.idea`：

```shell
# 假定当前在源码根目录
$ cd bin
$ bash idea.sh
```
这个过程可能会提示你安装 `ant`。

生成后，直接用 idea 打开就可以了，JDK 源码在 `src/java.base` 目录下。


### 使用 VS Code 阅读 hotspot 源码

由于 idea 的 CLion 是付费的，所以这部分选用 VS Code 来阅读。没什么好说的，就不写了。






