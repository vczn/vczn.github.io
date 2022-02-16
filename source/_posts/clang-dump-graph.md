---
title: Clang 输出各种图形
date: 2020-03-24 22:19:46
tags: 
 - LLVM
 - Clang
 - tool
categories:
- Clang
---

Clang 生成 AST、LLVM IR、Call Graph、CFG 和 Exploded Graph

<!-- more -->


注：以下内容主要针对 Clang 4 之后的版本，而且不保证之后打破向后兼容性，以 Clang 的官方文档为主，本文主要为个人备忘使用  

以下内容使用 clang 8.0.0  



## 生成 AST

```sh
clang -emit-ast test.c         # generate test.ast(binary format)
clang -cc1 -ast-dump test.c    # print AST to stdout(without color, no header)
clang -Xclang -ast-dump test.c # print AST to stdout(with color)
clang -cc1 -ast-view test.c    # print AST dot file
```



## 生成 IR

```sh
clang -emit-llvm -S test.c     # generate test.ll
```



## 生成图数据

生成图数据现在集成到 Clang 静态检测器中，所以需要使用 `clang -cc1 -analyze` 参数。

可以使用 `clang -cc1 -analyzer-checker-help | grep debug` 命令查看目前支持生成的一些图数据。`Dump` 开头的是以文本形式进行标准输出，`View` 开头的是以图形(dot)格式进行输出并打开相应软件来展示。

生成图数据是以函数调用链为单位的，如果需要对某个函数进行分析需要加上 `-analyze-function=xxx` 参数。

使用 `clang -cc1` 命令，Clang 是不会去查找系统头文件的，需要手动 `-I` 或 `-isystem` 来指定头文件目录或使用 AST 来进行分析。

头文件位置可以使用 `clang -### test.c -other-compile-args` 来找到所需头文件位置。

注：在 Windows 系统上，检测器参数需要加引号，比如 `"debug.ViewCFG"`



**函数调用图(Call Graph)**:

```sh
clang -cc1 -analyze -analyzer-checker=debug.ViewCallGraph test.c
# Windows
clang -cc1 -analyze -analyzer-checker="debug.ViewCallGraph" test.c
```



**控制流图(CFG)**:

```sh
clang -cc1 -analyze -analyzer-chekcer=debug.DumpCFG test.c
clang -cc1 -analyze -analyzer-chekcer=debug.DumpCFG test.ast
clang -cc1 -analyze -analyzer-chekcer=debug.ViewCFG test.c
clang -cc1 -analyze -analyzer-chekcer=debug.ViewCFG -analyze-function=foo test.c
```



**爆炸图(Exploded Graph)**:

```sh
clang -cc1 -analyze -analyzer-chekcer=debug.DumpExplodedGraph test.c
clang -cc1 -analyze -analyzer-chekcer=debug.DumpExplodedGraph test.c
clang -cc1 -analyze -analyzer-chekcer=debug.DumpExplodedGraph -analyze-function=foo test.c
```



## references

https://clang-analyzer.llvm.org/checker_dev_manual.html

https://clang.llvm.org/docs/ClangCommandLineReference.html

https://clang.llvm.org/docs/DriverInternals.html

https://clang.llvm.org/docs/FAQ.html