---
title: C++ 预定义宏与各种环境
date: 2021-05-09 20:49:35
tags: 
- C++
- macro
categories:
- C++
---

整理 C++ 预定义宏，来确定操作系统、语言标准（版本）、编译器、配置等信息

下面详细介绍各个平台或编译器预定义宏

# 一、MSVC

```cpp
_WIN32;   // if target is x86, x64, ARM32, ARM64
_WIN64;   // if target is x64, ARM64
_DEBUG;   // if DEBUG, /MDd | /MTd
_MT;      // if multi-thread, /MD | /MDd | /MT | /MTd
_DLL;     // if use dll, /MD | /MDd
__cplusplus and _MSVC_LANG; // see below
_MSC_VER and _MSC_FULL_VER; // see below

#ifdef _MSC_VER
  // MSVC
#endif
```

关于 MD 和 MT 编译选项：

|    选项    |                             说明                             |
| :--------: | :----------------------------------------------------------: |
| `MD`(默认) |      使用多线程 `DLL` 版本，即动态链接，能减少软件大小       |
|    `MT`    | 使用多线程静态库版本，即静态链接，减少依赖缺失或版本不对的问题，但是可能造成在不同堆上进行分配和释放问题 |

`__cplusplus` 默认为 `199711L`，如果需要使之生效需要开启 `/Zc:__cplusplus` 选项。具体见[这里](https://devblogs.microsoft.com/cppblog/msvc-now-correctly-reports-__cplusplus/)。

如果不开启 `/Zc:__cplusplus` 选项，可以使用 `_MSVC_LANG` 来代替 `__cplusplus`，其他的 `__cplusplus` 都是可以正常使用的，介绍其他不再提及 `__cplusplus`

Visual Studio 版本和 `_MSC_VER` 的对应关系，有时候需要判断 VS 版本的操作：

|         Visual Studio 版本         | `_MSC_VER` |
| :--------------------------------: | :--------: |
|         Visual Studio 6.0          |    1200    |
|   Visual Studio .NET 2002 (7.0)    |    1300    |
|   Visual Studio .NET 2003 (7.1)    |    1310    |
|      Visual Studio 2005 (8.0)      |    1400    |
|      Visual Studio 2008 (9.0)      |    1500    |
|     Visual Studio 2010 (10.0)      |    1600    |
|     Visual Studio 2012 (11.0)      |    1700    |
|     Visual Studio 2013 (12.0)      |    1800    |
|     Visual Studio 2015 (14.0)      |    1900    |
|   Visual Studio 2017 RTW (15.0)    |    1910    |
|    Visual Studio 2017 版本 15.3    |    1911    |
|    Visual Studio 2017 版本 15.5    |    1912    |
|    Visual Studio 2017 版本 15.6    |    1913    |
|     Visual Studio 2017 15.7 版     |    1914    |
|    Visual Studio 2017 版本 15.8    |    1915    |
|    Visual Studio 2017 版本 15.9    |    1916    |
|   Visual Studio 2019 RTW (16.0)    |    1920    |
|    Visual Studio 2019 版本 16.1    |    1921    |
|    Visual Studio 2019 版本 16.2    |    1922    |
|    Visual Studio 2019 版本 16.3    |    1923    |
|    Visual Studio 2019 版本 16.4    |    1924    |
|    Visual Studio 2019 版本 16.5    |    1925    |
|    Visual Studio 2019 版本 16.6    |    1926    |
|    Visual Studio 2019 版本 16.7    |    1927    |
| Visual Studio 2019 版本 16.9、16.9 |    1928    |

`_MSC_FULL_VER`，`cl` 编译器的版本号，使用 `cl /?` 可以查看版本号



# 二、Cygwin

Cygwin 运行在 Windows 平台，提供了类 Unix 环境。Cygwin 提供了一套抽象层 `dll`，将 POSIX 接口调用转换为 Windows API，实现相关功能。

Cygwin 提供了良好的 POSIX 支持，将非常多 Linux/BSD 应用移植到 Windows。

在 Cygwin 环境上的程序可以使用 POSIX API，运行需要 `cygwin1.dll` 模拟层。

所以 Cygwin 兼有 POSIX API 和 Win32 API，这就使得问题变得复杂。使用 Cygwin 的 g++ 是基于 POSIX 的，没有 `_WIN32` 等宏的定义，有 `__unix__` 宏，还可以使用 Cygwin 中 MinGW 编译器，基于原生 Windows 程序，使用 Win32 API，则包含 `_WIN32` 宏，但没有 `__unix__` 宏

```cpp
// Common
__CYGWIN__;     // if Cygwin
__CYGWIN32__;   // if Cygwin
__x86_64__;     // if AMD 64
__X86__;        // if x86
__LP64__;       // if LP64 data model

#ifdef __CYGWIN__
# ifdef _WIN32
  // Cygwin target Win32 API
# else
  // Cygwin target POSIX API
# endif
#endif
```

PS: g++/clang++ 可以使用 `g++|clang++ -E -dM - </dev/null` 来查看预定义宏





# 三、MinGW

MinGW 支持原生的 Windows 程序

```cpp
__MINGW32__;   // if MinGW 32 or MinGW-w64
__MINGW64__;   // if Mingw-w64
__GNUC__;
_WIN32;  // if MinGW 32 or MinGW-w64
_WIN64;  // if MinGW-w64

#ifdef __MINGW32__
  // MinGW
#endif
```



# 四、Linux

```cpp
__linux;
__linux__;
__gnu_linux__;
linux;
__unix;
__unix__;
unix;

#ifdef __linux__
  // Linux
#endif
```



# 五、Apple

```cpp
__APPLE__;
__MACH__;
__unix;
__unix__;
unix;

#ifdef __APPLE__
// or #if defined(__APPLE__) && defined(__MACH__) // macOS and iOS (Darwin)  
#  include <TargetConditionals.h>
#  if TARGET_IPHONE_SIMULATOR == 1
     // iOS
#  elif TARGET_OS_IPHONE == 1
     // iOS
#  elif TARGET_OS_MAC == 1
     // macOS
#  endif
#endif
```







# 六、BSD

```cpp
#if defined(__unix__) || defined(__UNIX__) || (defined(__APPLE__) && defined(__MACH__))
#  include <sys/param.h>
#  if defined(BSD)
     // BSD
#  end
#end
```



# 七、其他 Unix

|      Unix      |      特有宏       |
| :------------: | :---------------: |
|      AIX       |      `_AIX`       |
|     HP-UX      |     `__hpux`      |
| Oracle Solaris | `__sun && __SVR4` |
|    Android     |   `__ANDROID__`   |



# 八、Clang

```cpp
__llvm__;
__clang__;
// clang version...

__GNUC__;
// gnuc version...

#ifdef __clang__
  // Clang
#endif
```



# 九、GCC

```cpp
__GNUC__;
// gnuc version...

#ifdef __GNUC__
  // gcc
#endif
```

需要注意的是 Clang 也定义了 `__GNUC__` 宏，如果需要区分 Clang 和 GCC 需要先判断 Clang，再判断 GCC





# 总结

判断 C++ 版本

```cpp
// 199711L -> CXX98
#if ((defined(_MSVC_LANG) && _MSVC_LANG >= 201103L) || __cplusplus >= 201103L)
  // CXX11
#elif ((defined(_MSVC_LANG) && _MSVC_LANG >= 201402L) || __cplusplus >= 201402L)
  // CXX14
#elif ((defined(_MSVC_LANG) && _MSVC_LANG >= 201703L) || __cplusplus >= 201703L)
  // CXX17
#endif
```



判断操作系统或环境

```cpp
#if defined(_WIN32) || defined(__CYGWIN__)
  // Windows [Win32, Win64, Cygwin, MinGW]
#elif defined(__linux__)
  // Linux
#elif defined(__APPLE__) && defined(__MACH__)
  // Apple [macOS, iOS]
#  include <TargetConditionals.h>
#  if TARGET_IPHONE_SIMULATOR == 1
     // iOS
#  elif TARGET_OS_IPHONE == 1
     // iOS
#  elif TARGET_OS_MAC == 1
     // macOS
#  endif
#elif defined(__unix__) || defined(__UNIX__) || (defined(__APPLE__) && defined(__MACH__))
#  include <sys/param.h>
#  if defined(BSD)
     // BSD
#  end
#elif defined(_AIX)
  // AIX
#elif defined(__hpux)
  // HP-UX
#elif defined(__sun) && defined(__SVR4)
  // Solaris
#elif defined(__ANDROID__)
  // ANDROID
#elif defined(unix) || defined(__unix__) || defined(__unix)
  // Other Unix like OS
#endif
```



判断所使用编译器

```cpp
#if defined(_MSC_VER)
  // MSVC
#elif defined(__clang__)
  // Clang
#elif defined(__GNUC__)
  // GCC
#endif
```



# 参考资料

https://docs.microsoft.com/zh-cn/cpp/preprocessor/predefined-macros?view=msvc-160

https://stackoverflow.com/questions/142508/how-do-i-check-os-with-a-preprocessor-directive

https://jdebp.uk/FGA/predefined-macros-platform.html

https://web.archive.org/web/20191012035921/http://nadeausoftware.com/articles/2012/01/c_c_tip_how_use_compiler_predefined_macros_detect_operating_system#BSD

https://sourceforge.net/p/predef/wiki/OperatingSystems/