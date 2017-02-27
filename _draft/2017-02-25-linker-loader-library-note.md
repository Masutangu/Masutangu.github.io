---
layout: post
date: 2017-02-25T10:55:19+08:00
title: 链接和装载 
category: 读书笔记
---

# 概论

从源文件到可执行文件，可以分解为四个过程：**预处理**，**编译**，**汇编**，**链接**。

预处理主要完成以下工作：

* 展开所有宏定义，删除 #define
* 处理所有条件预编译指令
* 处理 #include 预编译指令。将被包含的文件插入到该预编译指令的位置
* 删除所有注释
* 添加行号和文件名标识，以便编译时编辑器产生调试用的行号信息以及编译产生错误或警告时的行号信息
* 保留所有 #pragma 编译器指令

编译的过程即把预处理完的文件进行一系列的词法分析、语法分析、语义分析以及优化后产生相应的汇编代码文件。

汇编器将汇编代码转变为机器可以执行的指令，即目标文件，再由链接器将多个目标文件输出成最后的可执行文件。链接器负责解决**模块间符号引用**的问题，链接的过程包括了**地址和空间分配**、**符号决议**和**重定位**。

# 目标文件 

目标文件和可执行文件的格式很相似，在 Linux 下统称为 ELF 文件。ELF 文件格式的核心思想是按不同属性分段（section）存储。

编译后的目标文件格式如下：

<img src="/assets/images/linker-loader-library-note/llustration-1.png" width="800" />

程序源代码编译后的机器指令通常放在代码段（.text），已初始化的全局变量和静态局部变量放在数据段（.data），.bss 段用于为未初始化的全局变量和静态局部变量预留空间，因此该段在文件中不占据空间。.rodata 段存放的是只读数据，一般是只读变量和字符串常量。

分段的好处在于：**方便权限控制**、**提高 CPU 缓存命中率**和**节省内存**。

下面看个实际的例子：

```c
/*
simplesection.c
linux: gcc -c simplesection.c -o simplesection.o
*/

int printf(const char* format, ...);

int global_init_var = 84;
int global_uninit_var;

void func1(int i) 
{
    printf("%d\n", i);
}

int main(void) 
{
    static int static_var = 85;
    static int static_var2;

    int a = 1;
    int b;

    func1(static_var + static_var2 + a + b);
    return a;
}
```

使用 ```objdump -h``` 查看段表的信息：

<img src="/assets/images/linker-loader-library-note/llustration-2.png" width="800" />

```objdump -h``` 只会显示关键的段，可以用 ```readelf -S``` 查看完整的段表信息：

<img src="/assets/images/linker-loader-library-note/llustration-5.png" width="800" />

注意这里的 .rel.text 段是重定位表，.symtab 段是符号表。下面会重点说明。

可以使用 ```objdump -s -d``` 查看各个段的内容：

<img src="/assets/images/linker-loader-library-note/llustration-3.png" width="800" />

可以看到 .data 段前后四个字节分别是是 54000000 和 55000000，即十进制 84 和 85。正好对应 global\_init\_var 和 static\_var 这两个变量。.rodata 段存放了 25640a00，即 %d\n 的 ASCII 字节序。

接下来用 ```readelf -h``` 看看 ELF 文件的头部：

<img src="/assets/images/linker-loader-library-note/llustration-4.png" width="800" />

入口地址（entry point address）标识了 ELF 程序的入口虚拟地址，操作系统在加载完该程序后从这个入口地址开始执行进程的指令。可重定位文件一般没有入口地址，即值为 0。

### 符号表 

用 ```readelf -s``` 看看 ELF 文件的符号表：

<img src="/assets/images/linker-loader-library-note/llustration-6.png" width="800" />

Type 列含义如下：

* NOTYPE：未知类型
* OBJECT：数据对象
* FUNC：函数或可执行代码
* SECTION：段
* FILE：文件名

Bind 列含义如下：

* LOCAL：局部符号，对目标文件的外部不可见
* GLOBAL：全局符号
* WEAK：弱引用

如果符号定义在本目标文件中，那 Ndx 列表示符号所在段在段表中的下标，如果不是定义在本目标文件中或是特殊符号，含义如下：

* ABS：表明该符号包含了一个绝对值，例如文件名符号
* COMMON：表明该符号是一个 COMMON 块类型的符号，一般未初始化的全局变量就是这种类型
* UNDEF：表明该符号未定义，在本目标文件被引用但定义在其他目标文件中

Value 列含义如下：
* 如果是符号的定义并且不是 COMMON 块，则表示符号在段中的偏移
* 如果是 COMMON 块，则表示对齐属性
* 可执行文件中，表示符号的虚拟地址，该虚拟地址对动态链接器来说非常有用

# 静态链接

链接器一般采用两步链接的方法来分配空间：

* 扫描所有输入目标文件，收集各个段的长度、属性和未位置，并将输入目标文件符号表中的所有符号定义和符号引用汇总到全局符号表。
* 读取输入文件中段的数据和重定位信息，进行符号解析与重定位。

来看一个链接的例子：

```
/* a.c */

extern int shared;

int main() 
{
    int a = 100;
    swap(&a, &shared);
}

/* b.c */
int shared = 1;

void swap(int *a, int *b) 
{
    *a ^= *b ^= *a ^= *b;
}
```

可以看到 a.c 中引用了 b.c 的两个符号，先用 objdump 看看 a.o 中代码段的反编译结果：

<img src="/assets/images/linker-loader-library-note/llustration-11.png" width="800" />

当 a.c 被编译成目标文件时，编译器并不知道 share 和 swap 的地址，因此编译器暂时把 地址 0 看成 shared 的地址：

```
13:	be 00 00 00 00       	mov    $0x0,%esi
```

相似的，调用 swap 的时候使用了临时假地址：

```
20: e8 00 00 00 00      	callq  25 <main+0x25>
```

使用 ld 将 a.o 和 b.o 链接起来：

<img src="/assets/images/linker-loader-library-note/llustration-7.png" width="800" />

再用 objdump 查看链接前后地址分配的情况：

<img src="/assets/images/linker-loader-library-note/llustration-8.png" width="800" />

<img src="/assets/images/linker-loader-library-note/llustration-9.png" width="800" />

<img src="/assets/images/linker-loader-library-note/llustration-10.png" width="800" />

VMA 表示虚拟内存地址，LMA 表示加载地址，正常情况下这两个值是一样的。

可以发现链接前目标文件所有段的 VMA 都是 0，因为虚拟空间还没分配。链接后可执行文件 ab 的各个段都分配了相应的虚拟地址。

当链接完成，由于各个段地址已经确定，各个符号在段内的相对地址也是固定的，这样各个符号的绝对地址也已经确定了。可以通过 objdump 观察到 ab 文件中符号地址已经修正了：

<img src="/assets/images/linker-loader-library-note/llustration-12.png" width="800" />

实际上，链接器是通过 ELF 文件中的重定位表，来找到需要重定位的符号的。```objdump -r``` 可以查看重定位表的信息：

<img src="/assets/images/linker-loader-library-note/llustration-13.png" width="800" />

OFFSET 表示该重定位符号在段中的偏移。

# 装载

Linux 下创建一个进程，加载可执行文件并执行，大致步骤如下：

* 创建一个独立的虚拟地址空间
* 读取可执行文件头，建立虚拟空间与可执行文件的映射关系。**Linux 内核的 VMA 结构就是用与建立虚拟空间和可执行文件的映射关系**
* 将 CPU 指令寄存器设置为可执行文件的入口地址，启动运行

操作系统并不关心可执行文件中各个段的实际内容，只关心段的权限。为了减少内存浪费（映射时以页长作为单位），对于相同权限的段，合并到一起当做一个段进行映射。因此 ELF 可执行文件引入了 segment 的概念。一个 segment 包含一个或多个属性相似的 section。描述 segment 的结构叫程序头（program header），可以使用 ```readelf -l``` 查看 segment 信息。

VMA 除了用以映射可执行文件的各个 segment 以外，还被用来管理进程的地址空间。进程执行需要的栈和堆，也是以 VMA 形式存在的。可以通过 ```cat /proc/pid/maps``` 查看进程的虚拟空间分布。

# 动态链接 

为了节省空间，简化程序的更新和发布，引入了动态链接的概念。动态链接的基本思想在于**将链接的过程推迟到运行时**。

由于共享对象的加载地址在编译时是不确定的，因此需要在装载时重定位。但由于装载时重定位会修改指令，没有办法做到同一份指令被多个进程共享，因此引入了**地址无关代码**（PIC）的概念。其基本思想是把指令中那些需要修改的部分抽离出来放到数据部分，这样的话指令部分可以保持不变，被多个进程共享，而数据部分每个进程都有一个副本。

当使用 PIC 编译时，模块内的数据引用和函数访问是通过相对寻址的方式。模块间的数据引用和函数访问则通过数据段的全局偏移表（GOT）来实现。

使用 gcc 编译时，指定 ```-shared``` 使用装载时重定位的方法，如果指定了 ```fPIC``` 则产生地址无关的共享对象。

为了提高动态链接的效率，引入了 PLT 来实现延迟绑定。





全局符号介入

vma 与linux 内核数据结构的对应

为什么不能取函数返回值的地址？
看《程序员的自我修养》 p324 

大于 128k mmap 所以小于128k的效率高