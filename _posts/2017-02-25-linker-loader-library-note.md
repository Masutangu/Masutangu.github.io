---
layout: post
date: 2017-02-25T10:55:19+08:00
title: 链接和装载 
category: 读书笔记
---

本文是读《程序员的自我修养: 链接、装载与库》所整理的读书笔记。

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

注意，```objdump -h``` 只会显示关键的段，可以用 ```readelf -S``` 查看完整的段表信息：

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

动态链接需要注意**全局符号介入**的问题，当一个符号需要被加入全局符号表时，如果相同的符号名已经存在，则后加入的符号将被忽略。由于可能存在全局符号介入的问题，模块内函数的调用不能用相对地址调用，编译器会将其当做模块外部符号来处理，使用 .got.plt 进行重定位。因此为了提高模块内函数调用的效率，建议使用 static 关键字将被调用的函数设置为编译单元私有函数。此时编译器会采用相对地址的编译方式，因为能确保该函数不会被其他模块所覆盖。


# 内存

下图为 Linux 进程内存空间的典型布局：

<img src="/assets/images/linker-loader-library-note/llustration-14.png" width="800" />

其中 dynamic libraries 用于映射装载的动态链接库。

## 栈 

栈在程序运行中起了举足轻重的地位。栈保存了一个函数调用所需要的维护信息，通常称为堆栈帧（Stack Frame）。通常有以下内容：

* 函数的返回地址和参数
* 临时变量，包括函数的非静态局部变量以及编译器自动生成的其他临时变量
* 保存的上下文，包括函数调用前后需要保存的寄存器

i386 中，esp 寄存器始终指向栈顶，ebp 寄存器指向了堆栈帧的一个固定位置，因此 ebp 又称为帧指针（Frame Pointer），如下图：

<img src="/assets/images/linker-loader-library-note/llustration-15.png" width="800" />

i386 的函数调用过程如下：

* 把参数压入栈中
* 把当前指令的下一条指令的地址压入栈中
* 跳转到函数体执行

i386 的函数体开头大致如下：

* push ebp: 把旧的 ebp 压入栈中 
* mov ebp, esp: 将 esp 的值赋给 ebp，此时 ebp 和 esp 都指向栈顶，栈顶保存的就是 old ebp
* sub esp, XXX: 可选，在栈上分配 XXX 字节的临时空间
* push XXX: 可选，如有必要，保存名为 XXX 的寄存器 

函数返回的流程如下：

* pop XXX: 可选，如有必要，恢复保存过的寄存器的值
* mov esp, ebp: 恢复 ESP，回收局部变量的空间
* pop ebp: 从栈顶恢复旧的 ebp 的值
* ret: 从栈顶取得返回地址，并跳转到该位置

eax 寄存器通常是函数返回值的通道。函数将返回值存储在 eax 中，返回后调用方读取 eax 得到函数的返回值。由于 eax 只有 4 个字节，如果返回的是 5~8 字节对象的情况，惯例是采用 eax 和 edx 联合返回的方式。超过 8 字节的，则通过临时变量的方式来实现：

* 调用者在栈上开辟了一片空间，并且将这块空间的一部分作为传递返回值的临时对象 temp
* 将 temp 对象的地址作为隐藏参数传递给被调用的函数
* 被调用的函数将返回值拷贝给 temp 对象，并将 temp 对象的地址用 eax 传出
* 返回后，调用者只需要拷贝 eax 指向的 temp 对象的内容就能得到返回值

这也是为什么不能直接对函数返回值进行取址的原因，因为函数返回后临时变量 temp 就被释放掉了。

## 堆
Linux 提供了两种堆空间分配的方式：brk() 和 mmap()。

brk() 的作用是设置进程数据段的结束地址，它可以扩大或缩小数据段。mmap() 的作用是向操作系统申请一段虚拟地址空间，可以是映射到某个文件也可以是匿名空间。

glibc 的 malloc 函数，对小于 128 KB 的请求来说，会在现有的堆空间里按照堆分配算法分配一块空间并返回。对于大于 128 KB 的请求来说，它会使用 mmap() 函数分配一块匿名空间。
