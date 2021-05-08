---
title: ELF64
date: 2019-09-08 22:47:19
tags:
- CS
- Linux
categories:
- CS
---



# 一、Intro

**ELF64**(**E**xecutable and **L**inkable **F**ormat 64) 是一种目前很多 Linux 和很多类 Unix 系统下使用的目标代码(executables, object code, shared libraries and core dumps) 64-bit 格式。是 System V ABI 的一部分。

相关工具有 `nm, objdump, readelf`

一点历史问题：Unix 最早使用可执行文件是 `a.out` 格式。当出现动态库时，原先的 `a.out` 就出现了问题， (SVR3) Unix System V Release 3 使用 COFF 来解决这个问题，后来 Windows NT 基于 COFF 格式，制定了 PE 格式。到 SVR4 的时候，在 COFF 的基础上引入了 ELF 格式。这也是为什么 PE 和 ELF 如此相似的原因。



---



# 二、Format

## 1. Overview

**ELF64 由下列部分组成：**

- ELF header，描述文件特征，比如类型、架构、版本、入口点等等。必须出现在文件开始处。
- Section header，对 relocatable 是必须的，对 loadable files 是可选的。
- Program header，对 loadable files 是必须的，对 relocatable 是可选的。描述了加载程序或动态链接库在准备阶段所需要的 loadable segment 和其他数据结构。
- Contents of the sections or segments，包括 loadable data, relocations,  string and symbol tables

关于术语 section 和 segment，在 ELF 中，section 包含了链接时所需要的信息，被链接器使用。segment 包含了运行时所需要的信息，被 kernel 用 `mmap` syscall 把 segment 映射到虚拟地址空间(位置和权限信息等)。

一个讨论见[这里](https://stackoverflow.com/questions/14361248/whats-the-difference-of-section-and-segment-in-elf-file-format)



**ELF64 数据类型**

|      Name      | Size(byte) |   Purpose   |
| :------------: | :--------: | :---------: |
|  `ELF64_Addr`  |     8      |   address   |
|  `Elf64_Off`   |     8      | file offset |
|  `Elf64_Half`  |     2      |     u16     |
|  `Elf64_Word`  |     4      |     u32     |
| `Elf64_Sword`  |     4      |     s32     |
| `Elf64_Xword`  |     8      |     u64     |
| `Elf64_Sxword` |     8      |     s64     |



## 2. ELF header

header 必须处于 ELF 文件的开头，主要是定位文件的其他部分，就类似于书的扉页和目录。能够查到 section header 和 program header 的位置。然后可以继续查 section header 来查找 section。

可以使用 `readelf -h elf-file` 查看 ELF header 



下面是 header 结构体

```c
typedef struct elf64_hdr {
  unsigned char	e_ident[EI_NIDENT]; // ELF magic number 
  Elf64_Half    e_type;             // Shared object file or Relocatable file
  Elf64_Half    e_machine;          // X86-64 ...
  Elf64_Word    e_version;
  Elf64_Addr    e_entry;		    
  Elf64_Off     e_phoff;            // Program header table file offset
  Elf64_Off     e_shoff;            // Section header table file offset
  Elf64_Word    e_flags;
  Elf64_Half    e_ehsize;           // elf header size
  Elf64_Half    e_phentsize;        // program headers size
  Elf64_Half    e_phnum;            // program headers number
  Elf64_Half    e_shentsize;        // section headers size
  Elf64_Half    e_shnum;            // section headers number
  Elf64_Half    e_shstrndx;         // section header string table index
} Elf64_Ehdr;
```



## 3. Section header

可以使用 `readelf -S elf-files` 或者 `objdump -h files` 查看 section headers，两者输出结果略微不同

ELF 文件的所有信息存储到 sections 中，除了 3 种 header，section headers 是 sections 的一个目录。sections 可以被 section header 的 index 标识。

```c
typedef struct elf64_shdr {
  Elf64_Word  sh_name;        // .text, .data, .bss ...
  Elf64_Word  sh_type;        // PROGBITS, NOBITS
  Elf64_Xword sh_flags;
  Elf64_Addr  sh_addr;
  Elf64_Off   sh_offset;      // file offset
  Elf64_Xword sh_size;        // size of section in bytes
  Elf64_Word  sh_link;        // index of another sectio
  Elf64_Word  sh_info;        // additional section information
  Elf64_Xword sh_addralign;  
  Elf64_Xword sh_entsize;     // entry size if section holds table
} Elf64_Shdr;
```



接下来对各个 section 进行简介

`.bss|.tbss`: Block Started by Symbol, 历史原因。未初始化的 global variables 和 static local variables 和 0。其中 `t` 为 thread data。0 的意思是 `static int x = 0` 也会放到 `.bss` 中节省存储空间。

`.data|.tdata`: 初始化的 global variables 和 static local variables。

`.interp`: Program interpreter path name

`.rodata`: read-only data

`.text`: code section，可以使用 `objdump -S file` 进行反汇编

`.comment`: 版本信息

`.dynamic`: 动态链接信息，`.dynstr|sym`: String | Symbol table for .dynamic section 

`.got`: global offset table, `.plt`: procedure linkage table，在动态链接被用到

`.hash`: 动态符号哈希表，

`.note`: 额外的编译器信息

`.relname|.relaname`: Relocations for section name

`.shstrtab`: Section name string table, `.strtab`: string table, `.symtab`: symbol table

`.init, .fini`: 程序初始化的中止的代码段



自定义段可以使用 `__attribute((section("SectionName"))) decl` 

 具体的 section 用到查[文档](https://uclibc.org/docs/elf-64-gen.pdf)



## 4. Program header

可以使用 `readelf -l elf-file` 查看 program header

sections 被分组为 segments 进行装载。program header 是 segment 的一个目录。

```C
typedef struct elf64_phdr {
  Elf64_Word   p_type;   // NULL, INTERP, LOAD, DYNAMIC, PHDR...
  Elf64_Word   p_flags;  // R W E MASKOS MASKPROC
  Elf64_Off    p_offset; // segment file offset
  Elf64_Addr   p_vaddr;  // segment virtual address
  Elf64_Addr   p_paddr;  // segment physical address
  Elf64_Xword  p_filesz; // segment size in file
  Elf64_Xword  p_memsz;  // segment size in memory
  Elf64_Xword  p_align;  // segment alignment file & memory
} Elf64_Phdr;
```



接下来的文章会对 static link 和 dynamic link 的过程进行分析。



# 三、参考资料

> https://uclibc.org/docs/elf-64-gen.pdf
>
> https://0xax.gitbooks.io/linux-insides/Theory/linux-theory-2.html
>
> https://wiki.osdev.org
>
> https://people.redhat.com/mpolacek/src/devconf2012.pdf
>
> [程序员的自我修养-链接、装载与库](https://book.douban.com/subject/3652388/)

