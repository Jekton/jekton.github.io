---
title: JVM 之路（1）- cmake 学习
date: 2018-08-19 20:39:36
categories: JVM
tags: way to JVM
---

在看一本书，《自己动手写JVM》，语言用的是 Go。想提高一下 C++ 的熟练度，所以自己用 C++ 跟着写一遍。既然用 C++，就得给自己找一个编译工具，最后选定的是 cmake。这篇文章记录的是个人 cmake 学习的一些总结，书籍使用《Professional CMake - A Practical Guide》。

<!--more-->

## 编译
```shell
mkdir build
cd build
cmake ..
cmake --build .
```

## 最简单的项目
```cmake
cmake_minimum_required(VERSION 3.2)
project(MyApp)
add_executable(myExe main.cpp)
```

参数可以跨越多行，命令名不区分大小写
```
add_executable(myExe
    main.cpp
    foo.cpp
    bar.cpp
)
```

## 添加库
```
add_library(targetName [STATIC | SHARED | MODULE]
              [EXCLUDE_FROM_ALL]
              source1 [source2 ...]
)
```
也可以直到编译时才指定使用静态还是动态库：
```shell
cmake -DBUILD_SHARED_LIBS=YES /path/to/source
```

### 链接

```
target_link_libraries(targetName
       <PRIVATE|PUBLIC|INTERFACE> item1 [item2 ...]
      [<PRIVATE|PUBLIC|INTERFACE> item3 [item4 ...]]
      ...
)
```

这里的库既可以是 cmake 里面定义的，也可以是一个路径、或者库名（如，pthread）


<br>
----
书还没看完，待续。