---
title: "通过剥离符号表提升 C++ 程序的编译速度"
date: 2023-02-01T14:11:17+08:00
draft: true
tags: ["C++", "Compilation", "Linker"]
categories: ["C++"]
---

# 通过剥离符号表提升 C++ 程序的编译速度

## 基础

### C++ 程序的构建过程

构建 C++ 程序并不是一个一蹴而就的过程，无论是 GCC, Clang 还是 MSVC 都只不过是把 C++ 程序的构建过程打包自动化了而已；C++ 程序的构建分为四个步骤：

1. 预处理器将被包含的头文件内容拷贝到源文件中，并将由 `#define` 定义的符号常量（symbolic constants）替换掉；
2. 编译器通过词法分析（lexical analysis）、语法分析（syntactic analysis）、语义分析（semantic analysis）、中间代码生成和优化、目标代码生成和优化等步骤，将预处理后的源文件（expanded source code file）翻译为对应平台的汇编代码（assembler code）；
3. 汇编器将汇编代码逐行转换为机器码/字节码（bytecode），并生成目标文件（object file）；
4. 链接器通过符号解析（symbol resolution）和重定位（relocation），将可重定位目标文件中未被解析的符号替换为其对应的地址，生成可执行目标文件或动态链接库。

其中，在词法分析阶段，编译器会将所有的标识符/符号（identifier / symbol）添加到符号表中；在接下来的语法分析和语义分析阶段，编译器会更新符号表，将符号的类型和作用域与其关联；在生成中间代码时，编译器会通过符号的类型来确定使用哪些指令来进行寄存器分配；最后在代码生成阶段，将符号的内存地址添加到符号表中。

即使代码中出现了只有声明而没有定义的符号，编译器也是可以正常生成汇编文件的，这些未定义的符号会在链接的时候被替换为对应的地址。

### 一个 helloworld 例子

```cpp
#include <iostream>
using namespace std;

int main() {
    cout << "hello world!" << endl;
    return 0;
}
```

在构建时加上 `-E` 选项可以在预处理阶段后中断；预处理后生成的文件一般会非常大，因为它会直接将包含的头文件展开：

```text
$ gcc -E helloworld.cpp > helloworld.i
$ ll
total 416K
4.0K -rw-rw-r-- 1 joelzychen joelzychen  107 Apr 22 20:28 helloworld.cpp
412K -rw-rw-r-- 1 joelzychen joelzychen 411K Apr 22 20:30 helloworld.i
```

在构建时加上 `-S` 选项可以在编译阶段后中断，生成的汇编文件会以 `.s` 作为扩展名：

```text
$ gcc -S helloworld.cpp
$ head helloworld.s
        .file   "helloworld.cpp"
        .local  _ZStL8__ioinit
        .comm   _ZStL8__ioinit,1,1
        .section        .rodata
.LC0:
        .string "hello world!"
        .text
        .globl  main
        .type   main, @function
main:
```

在构建时加上 `-c` 选项可以在汇编阶段后中断，生成的目标文件会以 `.o` 作为扩展名：

```text
$ gcc -c helloworld.cpp
$ ll
total 424K
412K -rw-rw-r-- 1 joelzychen joelzychen 411K Apr 22 20:31 helloworld.temp
4.0K -rw-rw-r-- 1 joelzychen joelzychen  107 Apr 22 20:31 helloworld.cpp
4.0K -rw-rw-r-- 1 joelzychen joelzychen 1.9K Apr 22 20:33 helloworld.s
4.0K -rw-rw-r-- 1 joelzychen joelzychen 2.7K Apr 22 20:37 helloworld.o
```

### ELF

`readelf` 命令可以用来展示 `elf` 文件。

### 符号表

在编译和汇编过程结束之后可以得到 relocatable object file。

它会生成一个符号表，符号表会将函数名或变量名等标识符/符号（identifier / symbol）与其相关信息，包括地址、类型、作用域、占用空间等关联在一起；链接器在链接时会根据符号表来对标识符进行寻址。

> TODO: 这篇来自 Notion，已经作为 private draft 迁入 blog。后续发布前需要补完整 “剥离符号表如何提升编译速度” 的实验和结论。
