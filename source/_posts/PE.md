---
title: PE
date: 2019-09-15 10:34:33
tags:
- CS
- Windows
categories:
- CS
---

# 一、Intro

PE(**P**rotable **E**xecutable) 是主要使用在 Windows 上目标代码格式，还有 UEFI 也是 PE 格式的。在 ELF64 中谈到 PE 和 ELF 都是 COFF 发展而来。也称为 PE/COFF 格式。32-bit 为 PE，64-bit 为 PE32+，基本就是 PE 扩展为 64-bit 地址格式。

和 ELF 基本类似，也是基于段的格式。同样段名只是提示作用和控制 link。自定义段名 

```c
#pragma data_seg("FOO")    // set
int global = 1;            // FOO section
#pragma data_seg(".data")  // reset
```

MSVC 的编译器和链接器分别叫 `cl` 和 `link`

进行实验的工具有 `dumpbin，xxd`



# 二、Format

## 1. Overview

先使用 `dumpbin /Summary test.exe` 查看一个可执行文件

```txt
.CRT .bss .data .debug_str .idata 
.pdata .rdata .text .tls .xdata ...
```

可以看到很多熟悉的 section。

和 ELF 类似，由许多的 headers(file headers and section table/headers) 、sections 组成。

大致看一下 32-bit PE 的结构(图片来自wiki百科)：

![PE](https://upload.wikimedia.org/wikipedia/commons/1/1b/Portable_Executable_32_bit_Structure_in_SVG_fixed.svg)

**Dos Header**

开头为 `0x5A4D`(小端)。在 `0x3C` 有4字节(小端)的 PE signature 文件偏移。比如 `0x3C` 处的 4 字节为 0x80 00 00 00，那么在文件偏移 0x80 处有 PE signature。



**MS-Dos Stub(Image only)**

接下来就是 Dos Stub，如果在 Dos 下运行，打印 "This program cannot be run in DOS mode"。然后从 PE  signature 就开始 COFF file format。



**PE format**

```c
typedef struct _IMAGE_NT_HEADERS
{
     ULONG Signature;
     IMAGE_FILE_HEADER FileHeader;
     IMAGE_OPTIONAL_HEADER OptionalHeader;
} IMAGE_NT_HEADERS, *PIMAGE_NT_HEADERS;
```



## 2. File headers

**PE signature(Image only)**

`"PE\0\0"`

**COFF file header(object and image)**

```c
typedef struct _IMAGE_FILE_HEADER {
    WORD  Machine;
    WORD  NumberOfSections;
    DWORD TimeDateStamp;
    DWORD PointerToSymbolTable;
    DWORD NumberOfSymbols;
    WORD  SizeOfOptionalHeader;
    WORD  Characteristics;
} IMAGE_FILE_HEADER, *PIMAGE_FILE_HEADER;
```

在 object 文件的开头或在 image file 的 PE signature 后。`WORD` 为 2bytes，`DWORD` 为 4 bytes。



**Optional header(image only)**

为 loader 提供信息，所以对 image 文件是必须的，对 object 文件时可选的。包括以下部分：

- Standard fields: 被 COFF 的所有的实现所定义。在 64-bit 没有
- Windows-specific fields: Windows 的 linker 和 loader 所需要的额外信息，比如 subsystem
- Data directories: 在 image 文件中找到并且由操作系统使用的特殊表的 address/size pair，比如 import table 和 export table。



## 3. Section Table

```c
typedef struct _IMAGE_SECTION_HEADER {
    BYTE  Name[IMAGE_SIZEOF_SHORT_NAME];  // section name
    union {
        DWORD PhysicalAddress;
        DWORD VirtualSize;        // 段被加载到内存的大小
    } Misc;
    DWORD VirtualAddress;         // 段被加载到内存后的虚拟地址
    DWORD SizeOfRawData;          // 该段在文件中的大小，.bss 段为 0
    DWORD PointerToRawData;       // 段在文件中的位置
    DWORD PointerToRelocations;   // 段在重定向表的位置
    DWORD PointerToLinenumbers;   // 段的行号表在文件中的位置
    WORD  NumberOfRelocations;
    WORD  NumberOfLinenumbers;
    DWORD Characteristics;        // 各种属性，见下文档连接
} IMAGE_SECTION_HEADER, *PIMAGE_SECTION_HEADER;
```

[Characteristics doc](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#section-flags)



## 4. Sections

Section Table(Header) 之后就是一个一个 Sections 的实际内容了，很多段和 ELF 文件类似。先简单介绍一些简单的 sections。

`.idata|.edata` 导入表和导出表。

`.tls` TLS(Thread-Local Storage) 数据

`.pdata` Exception information

`.debug$S|.debug$P|.debug$T`  包含 symbol、precompiled header file、type 相关的调试信息

接下来主要看一下 `.drectve` 

drectve 是 directive 的缩写，主要是 compiler 给 linker 留下的指令(directive)，告诉 linker 应该怎么去 link 这个 object file。

```txt
# example: dumpbin /ALL test.obj 
# ...
SECTION HEADER #1
.drectve name
       0 physical address
       0 virtual address
     181 size of raw data
    4754 file pointer to raw data (00004754 to 000048D4)
       0 file pointer to relocation table
       0 file pointer to line numbers
       0 number of relocations
       0 number of line numbers
  100A00 flags
         Info          # 该段是信息或注释 section
         Remove        # 链接成 image file 时移除该段
         1 byte align  
```

具体的内容有版本、宏、指定 CRT/Lib 还有其他传递给 linker 的参数。



## 5. Other

Sections 结束之后是是一个非常大的 Symbol Table，符号表的位置在 File headers  `PointerToSymbolTable` 中给出。再接下来是 String Table，看到 String Table 在 dumpbin 的输出只有大小，可以通过 COFF header 中的 `PointerToSymbolTable ` 加上 `NumberOfSymbols * sizeof(Symbol)` 就可以得到  String Table 的位置，再加上 String Table 的大小就能定位到整个 String Table。



# 三、参考资料

> https://docs.microsoft.com/en/windows/win32/debug/pe-format
>
> https://en.wikipedia.org/wiki/Portable_Executable
>
> https://wiki.osdev.org/PE
>
> [程序员的自我修养-链接、装载与库](https://book.douban.com/subject/3652388/)