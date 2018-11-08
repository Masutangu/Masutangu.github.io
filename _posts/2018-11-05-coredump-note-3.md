---
layout: post
date: 2018-11-05T22:28:22+08:00
title: Core dump 原理探究学习笔记（三）
tags: 
  - 读书笔记
  - 汇编
---

本系列文章是读[《coredump问题原理探究》](https://blog.csdn.net/xuzhina/article/category/1322964)的读书笔记。

## 基本数据类型

### char

```c++
#include <stdio.h>
int main() {
  char c1 = 'a';
  char c2 = 'b';
  char c3 = 'c';
  printf("addresses of c1 to c3 are (%x, %x, %x)\n", &c1, &c2, &c3);

  char* p = &c1;
  *p++;
  p = &c2;
  (*p) += 2;
  p = &c3;
  (*p) += 3;
  printf("address and value of p is ( %x, %c )\n", &p, *p);

  return 0;
}
```

```
(gdb) disassemble main
Dump of assembler code for function main:
0x00000000004005f0 <+0>: push %rbp
0x00000000004005f1 <+1>: mov %rsp,%rbp
0x00000000004005f4 <+4>: sub $0x10,%rsp
0x00000000004005f8 <+8>: movb $0x61,-0x1(%rbp)
0x00000000004005fc <+12>: movb $0x62,-0x2(%rbp)
0x0000000000400600 <+16>: movb $0x63,-0x3(%rbp)
0x0000000000400604 <+20>: lea -0x3(%rbp),%rcx
0x0000000000400608 <+24>: lea -0x2(%rbp),%rdx
0x000000000040060c <+28>: lea -0x1(%rbp),%rax
```

char 占一个字节，因此 movb 是 char 类型的特征指令。lea 指令用于获取内存单元的地址，为指针的特征指令。

### short

```c++
#include <stdio.h>
int main() {
  short c1 = 'a';
  short c2 = 'b';
  short c3 = 'c';
  printf("addresses of c1 to c3 are (%x, %x, %x)\n", &c1, &c2, &c3);

  short* p = &c1;
  *p++;
  p = &c2;
  (*p) += 2;
  p = &c3;
  (*p) += 3;
  printf("address and value of p is ( %x, %d )\n", &p, *p);

  return 0;
}
```

```
(gdb) disassemble main
Dump of assembler code for function main:
0x00000000004005f0 <+0>: push %rbp
0x00000000004005f1 <+1>: mov %rsp,%rbp
0x00000000004005f4 <+4>: sub $0x10,%rsp
0x00000000004005f8 <+8>: movw $0x61,-0x2(%rbp)
0x00000000004005fe <+14>: movw $0x62,-0x4(%rbp)
0x0000000000400604 <+20>: movw $0x63,-0x6(%rbp)
```

short是两个字节的，因此 movw 就是其特征指令。

下面表格是特征指令的总结：

| 类型 | 特征指令 |
| char | movb |
| short | movw |
| int | movl |
| long | movl (32-bit) <br> movq (64-bit) |
| 指针 | lea |

### char 数组

```c++
#include <stdio.h>
int main() {
  char buf[16];
  char c = 'a';

  printf("head of array:%x, tail of array:%x", buf, &buf[15]);

  for (int i = 0; i < 16; i++, c++) {
    buf[i] = c;
  }

  buf[15] = '\0';
  printf( "%s\n", buf );

  return 0;
}
```

```
(gdb) disassemble main
Dump of assembler code for function main:
0x0000000000400640 <+0>: push %rbp
0x0000000000400641 <+1>: mov %rsp,%rbp
0x0000000000400644 <+4>: sub $0x20,%rsp
0x0000000000400648 <+8>: movb $0x61,-0x1(%rbp)
0x000000000040064c <+12>: lea -0x20(%rbp),%rax
0x0000000000400650 <+16>: lea 0xf(%rax),%rdx
0x0000000000400654 <+20>: lea -0x20(%rbp),%rax
0x0000000000400658 <+24>: mov %rax,%rsi
0x000000000040065b <+27>: mov $0x400740,%edi
0x0000000000400660 <+32>: mov $0x0,%eax
0x0000000000400665 <+37>: callq 0x400510 <printf@plt>
0x000000000040066a <+42>: movl $0x0,-0x8(%rbp)
0x0000000000400671 <+49>: jmp 0x40068e <main+78>
0x0000000000400673 <+51>: mov -0x8(%rbp),%eax
0x0000000000400676 <+54>: cltq
0x0000000000400678 <+56>: movzbl -0x1(%rbp),%edx
0x000000000040067c <+60>: mov %dl,-0x20(%rbp,%rax,1)
0x0000000000400680 <+64>: addl $0x1,-0x8(%rbp)
0x0000000000400684 <+68>: movzbl -0x1(%rbp),%eax
0x0000000000400688 <+72>: add $0x1,%eax
0x000000000040068b <+75>: mov %al,-0x1(%rbp)
0x000000000040068e <+78>: cmpl $0xf,-0x8(%rbp)
0x0000000000400692 <+82>: jle 0x400673 <main+51>
0x0000000000400694 <+84>: movb $0x0,-0x11(%rbp)
0x0000000000400698 <+88>: lea -0x20(%rbp),%rax
0x000000000040069c <+92>: mov %rax,%rdi
0x000000000040069f <+95>: callq 0x400530 <puts@plt>
0x00000000004006a4 <+100>: mov $0x0,%eax
0x00000000004006a9 <+105>: leaveq
0x00000000004006aa <+106>: retq
End of assembler dump.
```

从
```
0x000000000040064c <+12>: lea -0x20(%rbp),%rax
0x0000000000400650 <+16>: lea 0xf(%rax),%rdx
```
可以看出数组 buf 的基址为 -0x20(%rbp)，buf[15] 的地址则由 0xf(%rax) 得出，即基址加上 0xf，刚好为 基址 + 15 * sizeof(char)。

### short 数组

```c++
#include <stdio.h>
int main() {
  short buf[16];
  short s = 'a';

  printf( "head of array:%x, tail of array:%x", buf, &buf[15] );

  for ( int i = 0; i < 16; i++, s++ ) {
    buf[i] = s;
  }

  return buf[15];
}
```

```
(gdb) disassemble main
Dump of assembler code for function main:
0x00000000004005f0 <+0>: push %rbp
0x00000000004005f1 <+1>: mov %rsp,%rbp
0x00000000004005f4 <+4>: sub $0x30,%rsp
0x00000000004005f8 <+8>: movw $0x61,-0x2(%rbp)
0x00000000004005fe <+14>: lea -0x30(%rbp),%rax
0x0000000000400602 <+18>: lea 0x1e(%rax),%rdx
0x0000000000400606 <+22>: lea -0x30(%rbp),%rax
0x000000000040060a <+26>: mov %rax,%rsi
0x000000000040060d <+29>: mov $0x4006e0,%edi
0x0000000000400612 <+34>: mov $0x0,%eax
0x0000000000400617 <+39>: callq 0x4004d0 <printf@plt>
0x000000000040061c <+44>: movl $0x0,-0x8(%rbp)
0x0000000000400623 <+51>: jmp 0x400642 <main+82>
0x0000000000400625 <+53>: mov -0x8(%rbp),%eax
0x0000000000400628 <+56>: cltq
0x000000000040062a <+58>: movzwl -0x2(%rbp),%edx
0x000000000040062e <+62>: mov %dx,-0x30(%rbp,%rax,2)
0x0000000000400633 <+67>: addl $0x1,-0x8(%rbp)
0x0000000000400637 <+71>: movzwl -0x2(%rbp),%eax
0x000000000040063b <+75>: add $0x1,%eax
0x000000000040063e <+78>: mov %ax,-0x2(%rbp)
0x0000000000400642 <+82>: cmpl $0xf,-0x8(%rbp)
0x0000000000400646 <+86>: jle 0x400625 <main+53>
0x0000000000400648 <+88>: movzwl -0x12(%rbp),%eax
0x000000000040064c <+92>: cwtl
0x000000000040064d <+93>: leaveq
0x000000000040064e <+94>: retq
End of assembler dump.
```

由
```
0x00000000004005fe <+14>: lea -0x30(%rbp),%rax
0x0000000000400602 <+18>: lea 0x1e(%rax),%rdx
```

得知 buf 数组基址是为 -0x30(%rbp)，buf[15] 的地址为基址加上 0x1e （short 类型的大小是2个字节）。

由
```
0x000000000040061c <+44>: movl $0x0,-0x8(%rbp)
0x0000000000400625 <+53>: mov -0x8(%rbp),%eax
0x000000000040062e <+62>: mov %dx,-0x30(%rbp,%rax,2)
```
得出变量 i 存放在 -0x8(%rbp)，每次对数组元素的引用 buf[i] 为基址 -0x30(%rbp) + sizeof(short) * i 即 -0x30(%rbp) + 2 * i。

## coredump 分析

执行下面这段代码：

```c++
#include <stdlib.h>
#include <string.h>
int main() {
  int* ptrArray[4] = { NULL, };

  for (int i = 0; i < 4; i += 2) {
    ptrArray[i] = new int[8];
  }

  for (char c = 0; c < 4; c++) {
    memset(ptrArray[c], c, 8 * sizeof(int));
  }

  return 0;
}
```

产生的 coredump 文件：

```
(gdb) bt
#0 0x00007fb3cf8e972c in __memset_sse2 () from /lib64/libc.so.6
#1 0x00000000004006e4 in main ()
```

执行 disassemble 看看堆栈：

```
(gdb) bt
#0 0x00007f53cf4c972c in __memset_sse2 () from /lib64/libc.so.6
#1 0x00000000004006e4 in main ()
(gdb) frame 1
#1 0x00000000004006e4 in main ()
(gdb) disassemble
Dump of assembler code for function main:
0x0000000000400670 <+0>: push %rbp
0x0000000000400671 <+1>: mov %rsp,%rbp
0x0000000000400674 <+4>: sub $0x30,%rsp
0x0000000000400678 <+8>: movq $0x0,-0x30(%rbp)
0x0000000000400680 <+16>: movq $0x0,-0x28(%rbp)
0x0000000000400688 <+24>: movq $0x0,-0x20(%rbp)
0x0000000000400690 <+32>: movq $0x0,-0x18(%rbp)
0x0000000000400698 <+40>: movl $0x0,-0x4(%rbp)
0x000000000040069f <+47>: jmp 0x4006ba <main+74>
0x00000000004006a1 <+49>: mov $0x20,%edi
0x00000000004006a6 <+54>: callq 0x400560 <_Znam@plt>
0x00000000004006ab <+59>: mov -0x4(%rbp),%edx
0x00000000004006ae <+62>: movslq %edx,%rdx
0x00000000004006b1 <+65>: mov %rax,-0x30(%rbp,%rdx,8)
0x00000000004006b6 <+70>: addl $0x2,-0x4(%rbp)
0x00000000004006ba <+74>: cmpl $0x3,-0x4(%rbp)
0x00000000004006be <+78>: jle 0x4006a1 <main+49>
0x00000000004006c0 <+80>: movb $0x0,-0x5(%rbp)
0x00000000004006c4 <+84>: jmp 0x4006ee <main+126>
0x00000000004006c6 <+86>: mov $0x20,%edx
0x00000000004006cb <+91>: movsbl -0x5(%rbp),%ecx
0x00000000004006cf <+95>: movsbl -0x5(%rbp),%eax
0x00000000004006d3 <+99>: cltq
0x00000000004006d5 <+101>: mov -0x30(%rbp,%rax,8),%rax
0x00000000004006da <+106>: mov %ecx,%esi
0x00000000004006dc <+108>: mov %rax,%rdi
0x00000000004006df <+111>: callq 0x400540 <memset@plt>
=> 0x00000000004006e4 <+116>: movzbl -0x5(%rbp),%eax
0x00000000004006e8 <+120>: add $0x1,%eax
0x00000000004006eb <+123>: mov %al,-0x5(%rbp)
0x00000000004006ee <+126>: cmpb $0x3,-0x5(%rbp)
0x00000000004006f2 <+130>: jle 0x4006c6 <main+86>
0x00000000004006f4 <+132>: mov $0x0,%eax
0x00000000004006f9 <+137>: leaveq
0x00000000004006fa <+138>: retq
End of assembler dump.
```

可以看出是在调用 ```memset``` 之后 core 的。通过汇编推测出循环变量 c 存放于 $rbp-0x5，查看崩溃时 c 的值：

```
(gdb) x /c $rbp-0x5
0x7ffec6da164b: 1 '\001'
```

崩溃时正好指向 arr 的第二个元素 ptrArray[1]。由
```
0x00000000004006b1 <+65>: mov %rax,-0x30(%rbp,%rdx,8)
```

推断出数组的基址为 -0x30(%rbp), 由数组元素作为 ```memset``` 的参数，可见数组的元素是指针， 由步长为 8 推断出是 64-bit 的机器。
查看 ptrArray[1] 的值：

```
(gdb) x /2x $rbp-0x30+8
0x7ffd641fbcb8: 0x00000000 0x00000000
```

确定是空指针传入 ```memset``` 导致 core。

## coredump 分析 II

```
Program terminated with signal 11, Segmentation fault.
#0  0x0000000000000000 in ?? ()
Missing separate debuginfos, use: debuginfo-install glibc-2.17-157.tl2.2.x86_64 libgcc-4.8.5-4.el7.x86_64 libstdc++-4.8.5-4.el7.x86_64
(gdb) bt
#0  0x0000000000000000 in ?? ()
#1  0x0000000000400619 in result(xuzhina_dump_c05_s3_ex*, int) ()
#2  0x000000000040086d in main ()
(gdb) disassemble result
Dump of assembler code for function _Z6resultP22xuzhina_dump_c05_s3_exi:
   0x00000000004005b0 <+0>:     push   %rbp
   0x00000000004005b1 <+1>:     mov    %rsp,%rbp
   0x00000000004005b4 <+4>:     sub    $0x20,%rsp
   0x00000000004005b8 <+8>:     mov    %rdi,-0x18(%rbp)
   0x00000000004005bc <+12>:    mov    %esi,-0x1c(%rbp)
   0x00000000004005bf <+15>:    movl   $0x0,-0x4(%rbp)
   0x00000000004005c6 <+22>:    movl   $0x0,-0x8(%rbp)
   0x00000000004005cd <+29>:    jmp    0x400620 <_Z6resultP22xuzhina_dump_c05_s3_exi+112>
   0x00000000004005cf <+31>:    mov    -0x8(%rbp),%eax
   0x00000000004005d2 <+34>:    cltq   
   0x00000000004005d4 <+36>:    shl    $0x4,%rax
   0x00000000004005d8 <+40>:    mov    %rax,%rdx
   0x00000000004005db <+43>:    mov    -0x18(%rbp),%rax
   0x00000000004005df <+47>:    add    %rdx,%rax
   0x00000000004005e2 <+50>:    mov    0x8(%rax),%rax
   0x00000000004005e6 <+54>:    mov    -0x8(%rbp),%edx
   0x00000000004005e9 <+57>:    movslq %edx,%rdx
   0x00000000004005ec <+60>:    mov    %rdx,%rcx
   0x00000000004005ef <+63>:    shl    $0x4,%rcx
   0x00000000004005f3 <+67>:    mov    -0x18(%rbp),%rdx
   0x00000000004005f7 <+71>:    add    %rcx,%rdx
   0x00000000004005fa <+74>:    mov    0x4(%rdx),%ecx
   0x00000000004005fd <+77>:    mov    -0x8(%rbp),%edx
   0x0000000000400600 <+80>:    movslq %edx,%rdx
   0x0000000000400603 <+83>:    mov    %rdx,%rsi
   0x0000000000400606 <+86>:    shl    $0x4,%rsi
   0x000000000040060a <+90>:    mov    -0x18(%rbp),%rdx
   0x000000000040060e <+94>:    add    %rsi,%rdx
   0x0000000000400611 <+97>:    mov    (%rdx),%edx
   0x0000000000400613 <+99>:    mov    %ecx,%esi
   0x0000000000400615 <+101>:   mov    %edx,%edi
   0x0000000000400617 <+103>:   callq  *%rax
   0x0000000000400619 <+105>:   add    %eax,-0x4(%rbp)
   0x000000000040061c <+108>:   addl   $0x1,-0x8(%rbp)
   0x0000000000400620 <+112>:   mov    -0x8(%rbp),%eax
   0x0000000000400623 <+115>:   cmp    -0x1c(%rbp),%eax
   0x0000000000400626 <+118>:   jl     0x4005cf <_Z6resultP22xuzhina_dump_c05_s3_exi+31>
   0x0000000000400628 <+120>:   mov    -0x4(%rbp),%eax
   0x000000000040062b <+123>:   leaveq 
   0x000000000040062c <+124>:   retq   
End of assembler dump.
(gdb) i r rip
rip            0x0      0x0
```

可以看到调用堆栈顶层的地址为空，rip 也为 0。这种情况只可能是调用了地址为 0 的函数指针。

由
```
    0x00000000004005bc <+12>:    mov    %esi,-0x1c(%rbp)
    0x00000000004005c6 <+22>:    movl   $0x0,-0x8(%rbp)
    ...
    0x000000000040061c <+108>:   addl   $0x1,-0x8(%rbp)
    0x0000000000400620 <+112>:   mov    -0x8(%rbp),%eax
    0x0000000000400623 <+115>:   cmp    -0x1c(%rbp),%eax
    0x0000000000400626 <+118>:   jl     0x4005cf <_Z6resultP22xuzhina_dump_c05_s3_exi+31>
```
推测是循环遍历，循环变量 i 存在 -0x8(%rbp)，每次递增 1，然后与变量 -0x1c(%rbp) 比较。


由
```
    0x00000000004005b8 <+8>:     mov    %rdi,-0x18(%rbp)
    ...

    0x00000000004005cf <+31>:    mov    -0x8(%rbp),%eax
    0x00000000004005d2 <+34>:    cltq   
    0x00000000004005d4 <+36>:    shl    $0x4,%rax
    0x00000000004005d8 <+40>:    mov    %rax,%rdx
    0x00000000004005db <+43>:    mov    -0x18(%rbp),%rax
    0x00000000004005df <+47>:    add    %rdx,%rax
    0x00000000004005e2 <+50>:    mov    0x8(%rax),%rax

    0x00000000004005e6 <+54>:    mov    -0x8(%rbp),%edx
    0x00000000004005e9 <+57>:    movslq %edx,%rdx
    0x00000000004005ec <+60>:    mov    %rdx,%rcx
    0x00000000004005ef <+63>:    shl    $0x4,%rcx
    0x00000000004005f3 <+67>:    mov    -0x18(%rbp),%rdx
    0x00000000004005f7 <+71>:    add    %rcx,%rdx
    0x00000000004005fa <+74>:    mov    0x4(%rdx),%ecx

    0x00000000004005fd <+77>:    mov    -0x8(%rbp),%edx
    0x0000000000400600 <+80>:    movslq %edx,%rdx
    0x0000000000400603 <+83>:    mov    %rdx,%rsi
    0x0000000000400606 <+86>:    shl    $0x4,%rsi
    0x000000000040060a <+90>:    mov    -0x18(%rbp),%rdx
    0x000000000040060e <+94>:    add    %rsi,%rdx
    0x0000000000400611 <+97>:    mov    (%rdx),%edx
```

这三段基本一致的汇编代码我们推测是在遍历数组，数组的基址为 -0x18(%rbp)。由每段最后一句汇编推测数组元素为结构体，每段取结构体中的不同元素。

从```0x00000000004005d4 <+36>:    shl    $0x4,%rax```得知每次遍历数组的偏移值为 16（左移 4 位 即乘以 16），说明数组元素的大小为 16 字节。

看看崩溃时 i 的值：

```
(gdb) x /x $rbp-0x8
0x7ffecfe8e358: 0x00000003
```

crash 时 i 等于 3，即第 4 个数组元素。

由
```
    0x00000000004005d4 <+36>:    shl    $0x4,%rax
    0x00000000004005d8 <+40>:    mov    %rax,%rdx
    0x00000000004005db <+43>:    mov    -0x18(%rbp),%rax
    0x00000000004005df <+47>:    add    %rdx,%rax
    0x00000000004005e2 <+50>:    mov    0x8(%rax),%rax
    ...
    0x0000000000400617 <+103>:   callq  *%rax
```

得知函数指针存放于结构体的 0x8 偏移的字段，我们打印下下数组第三个元素的内存值，即基址 -0x18(%rbp) + 3 * 16 = -0x18(%rbp) + 0x30：

```
(gdb) x /2x $rbp-0x18
0x7ffecfe8e348: 0xcfe8e370      0x00007ffe
(gdb) x /3x 0x00007ffecfe8e370+0x30
0x7ffecfe8e3a0: 0x00000003      0x00000003      0x00000000
```

可以看出数组第一和第二个字段值为3，第三个元素为 0，正好是空指针。
对比下下面的源码：

```c++
typedef int (*operation)(int a, int b);
struct xuzhina_dump_c05_s3_ex {
    int a;
    int b;
    operation oper;
};

int result(struct xuzhina_dump_c05_s3_ex test[], int num) {
    int res = 0;
    for (int i = 0; i < num; i++) {
        res += test[i].oper(test[i].a, test[i].b);
    }
    
    return res;
}

int add(int a, int b) {
    return a + b;
}

int sub(int a, int b) {
    return a - b;
}

int mul(int a, int b) {
    return a * b;
}

void init(struct xuzhina_dump_c05_s3_ex test[], int num) {
    for (int i = 0; i < num; i++) {
        switch( i % 4 ) {
        case 0:
            test[i].a = i / 4;
            test[i].b = 0;
            test[i].oper = add;
            break;
        case 1:
            test[i].a = i / 4;
            test[i].b = i % 4;
            test[i].oper = mul;
            break;
        case 2:
            test[i].a = i % 4;
            test[i].b = i / 4;
            test[i].oper = sub;
            break;
        default:
            test[i].a = i;
            test[i].b = i % 4;
            test[i].oper = 0;
            break;
        }
    }
}

int main() {
    struct xuzhina_dump_c05_s3_ex test[15];
    init(test, 15);

    return result(test, 15);
}
```

数组第三个元素的字段分别为 3、3、0，和我们的推测相符。

## 类

### 类成员函数

```c++
#include <stdio.h>
class xuzhina_dump_c06_s1 {
  public:
    void hello() {
      printf("hello\n");
    }

    void print() {
      printf("this:%p\n", this);
    }
};

int main() {
  xuzhina_dump_c06_s1 test;
  printf("address of test:%p\n", &test);
  test.print();
  test.hello();
  return 0;
}
```

```
(gdb) disassemble main
Dump of assembler code for function main:
   0x0000000000400640 <+0>:     push   %rbp
   0x0000000000400641 <+1>:     mov    %rsp,%rbp
   0x0000000000400644 <+4>:     sub    $0x10,%rsp
   0x0000000000400648 <+8>:     lea    -0x1(%rbp),%rax
   0x000000000040064c <+12>:    mov    %rax,%rsi
   0x000000000040064f <+15>:    mov    $0x40075f,%edi
   0x0000000000400654 <+20>:    mov    $0x0,%eax
   0x0000000000400659 <+25>:    callq  0x400510 <printf@plt>
   0x000000000040065e <+30>:    lea    -0x1(%rbp),%rax
   0x0000000000400662 <+34>:    mov    %rax,%rdi
   0x0000000000400665 <+37>:    callq  0x400696 <_ZN19xuzhina_dump_c06_s15printEv>
   0x000000000040066a <+42>:    lea    -0x1(%rbp),%rax
   0x000000000040066e <+46>:    mov    %rax,%rdi
   0x0000000000400671 <+49>:    callq  0x40067e <_ZN19xuzhina_dump_c06_s15helloEv>
   0x0000000000400676 <+54>:    mov    $0x0,%eax
   0x000000000040067b <+59>:    leaveq 
   0x000000000040067c <+60>:    retq   
End of assembler dump.
```

调用 ```print()``` 和 ```hello()``` 的汇编指令如下：

```
0x000000000040065e <+30>:    lea    -0x1(%rbp),%rax
0x0000000000400662 <+34>:    mov    %rax,%rdi
0x0000000000400665 <+37>:    callq  0x400696 <_ZN19xuzhina_dump_c06_s15printEv>
0x000000000040066a <+42>:    lea    -0x1(%rbp),%rax
0x000000000040066e <+46>:    mov    %rax,%rdi
0x0000000000400671 <+49>:    callq  0x40067e <_ZN19xuzhina_dump_c06_s15helloEv>
```

可以看到，调用类成员函数时会隐式传入 this 指针作为第一个参数。

### 类成员变量

```c++
#include <stdio.h>
class xuzhina_dump_c06_s2 {
private:
  short m_c;
  char m_d;
  int m_e;
public:
  xuzhina_dump_c06_s2(int a, int b) {
    m_c = (short)(a + b);
    m_d = 'd';
    m_e = a - b;
  }

  void print() {
    printf("member %d, %c, %d\n", m_c, m_d, m_e);
  }
};

int main() {
  xuzhina_dump_c06_s2 test(2, 3);
  test.print();
  return 0;
}
```

```
(gdb) disassemble main
Dump of assembler code for function main:
   0x00000000004005f0 <+0>:     push   %rbp
   0x00000000004005f1 <+1>:     mov    %rsp,%rbp
   0x00000000004005f4 <+4>:     sub    $0x10,%rsp
   0x00000000004005f8 <+8>:     lea    -0x10(%rbp),%rax
   0x00000000004005fc <+12>:    mov    $0x3,%edx
   0x0000000000400601 <+17>:    mov    $0x2,%esi
   0x0000000000400606 <+22>:    mov    %rax,%rdi
   0x0000000000400609 <+25>:    callq  0x400622 <_ZN19xuzhina_dump_c06_s2C2Eii>
   0x000000000040060e <+30>:    lea    -0x10(%rbp),%rax
   0x0000000000400612 <+34>:    mov    %rax,%rdi
   0x0000000000400615 <+37>:    callq  0x40065c <_ZN19xuzhina_dump_c06_s25printEv>
   0x000000000040061a <+42>:    mov    $0x0,%eax
   0x000000000040061f <+47>:    leaveq 
   0x0000000000400620 <+48>:    retq   
End of assembler dump.
```

打个断点：

```
(gdb) tbreak *0x000000000040060e
Temporary breakpoint 1 at 0x40060e
(gdb) r
```

查看对象是如何存放的：

```
(gdb) x /8x $rbp-0x10
0x7fffffffe460: 0xff640005      0xffffffff      0x00000000      0x00000000
0x7fffffffe470: 0x00000000      0x00000000      0xf721cb35      0x00007fff
(gdb) x /c 0x7fffffffe460+0x2
0x7fffffffe462: 100 'd'
```

对象由低到高分别存储 5（0x0005）, 'd'（0xff64）, -1（0xffffffff）。可见类的成员布局和结构体一样。


## coredump 分析 III

```
(gdb) bt
#0  0x0000000000400711 in xuzhina_dump_c06_s2_ex::print() ()
#1  0x00000000004007fd in main ()
(gdb) disassemble 
Dump of assembler code for function _ZN22xuzhina_dump_c06_s2_ex5printEv:
   0x00000000004006d8 <+0>:     push   %rbp
   0x00000000004006d9 <+1>:     mov    %rsp,%rbp
   0x00000000004006dc <+4>:     sub    $0x20,%rsp
   0x00000000004006e0 <+8>:     mov    %rdi,-0x18(%rbp)
   0x00000000004006e4 <+12>:    movl   $0x0,-0x4(%rbp)
   0x00000000004006eb <+19>:    jmpq   0x400781 <_ZN22xuzhina_dump_c06_s2_ex5printEv+169>
   0x00000000004006f0 <+24>:    mov    -0x18(%rbp),%rax
   0x00000000004006f4 <+28>:    movzwl (%rax),%eax
   0x00000000004006f7 <+31>:    cwtl   
   0x00000000004006f8 <+32>:    test   %eax,%eax
   0x00000000004006fa <+34>:    je     0x400703 <_ZN22xuzhina_dump_c06_s2_ex5printEv+43>
   0x00000000004006fc <+36>:    cmp    $0x1,%eax
   0x00000000004006ff <+39>:    je     0x40072a <_ZN22xuzhina_dump_c06_s2_ex5printEv+82>
   0x0000000000400701 <+41>:    jmp    0x400757 <_ZN22xuzhina_dump_c06_s2_ex5printEv+127>
   0x0000000000400703 <+43>:    mov    -0x18(%rbp),%rax
   0x0000000000400707 <+47>:    mov    0x18(%rax),%rdx
   0x000000000040070b <+51>:    mov    -0x4(%rbp),%eax
   0x000000000040070e <+54>:    add    %rdx,%rax
=> 0x0000000000400711 <+57>:    movzbl (%rax),%eax
   0x0000000000400714 <+60>:    movsbl %al,%eax
   0x0000000000400717 <+63>:    mov    %eax,%esi
   0x0000000000400719 <+65>:    mov    $0x4008a0,%edi
   0x000000000040071e <+70>:    mov    $0x0,%eax
   0x0000000000400723 <+75>:    callq  0x400540 <printf@plt>
   0x0000000000400728 <+80>:    jmp    0x40077d <_ZN22xuzhina_dump_c06_s2_ex5printEv+165>
   0x000000000040072a <+82>:    mov    -0x18(%rbp),%rax
   0x000000000040072e <+86>:    mov    0x18(%rax),%rax
   0x0000000000400732 <+90>:    mov    -0x4(%rbp),%edx
   0x0000000000400735 <+93>:    shl    $0x2,%rdx
   0x0000000000400739 <+97>:    add    %rdx,%rax
   0x000000000040073c <+100>:   movss  (%rax),%xmm0
   0x0000000000400740 <+104>:   unpcklps %xmm0,%xmm0
   0x0000000000400743 <+107>:   cvtps2pd %xmm0,%xmm0
   0x0000000000400746 <+110>:   mov    $0x4008a4,%edi
   0x000000000040074b <+115>:   mov    $0x1,%eax
   0x0000000000400750 <+120>:   callq  0x400540 <printf@plt>
   0x0000000000400755 <+125>:   jmp    0x40077d <_ZN22xuzhina_dump_c06_s2_ex5printEv+165>
   0x0000000000400757 <+127>:   mov    -0x18(%rbp),%rax
   0x000000000040075b <+131>:   mov    0x18(%rax),%rax
   0x000000000040075f <+135>:   mov    -0x4(%rbp),%edx
   0x0000000000400762 <+138>:   shl    $0x2,%rdx
   0x0000000000400766 <+142>:   add    %rdx,%rax
   0x0000000000400769 <+145>:   mov    (%rax),%eax
   0x000000000040076b <+147>:   mov    %eax,%esi
   0x000000000040076d <+149>:   mov    $0x4008a8,%edi
   0x0000000000400772 <+154>:   mov    $0x0,%eax
   0x0000000000400777 <+159>:   callq  0x400540 <printf@plt>
   0x000000000040077c <+164>:   nop
   0x000000000040077d <+165>:   addl   $0x1,-0x4(%rbp)
   0x0000000000400781 <+169>:   mov    -0x18(%rbp),%rax
   0x0000000000400785 <+173>:   mov    0x20(%rax),%eax
   0x0000000000400788 <+176>:   cmp    -0x4(%rbp),%eax
   0x000000000040078b <+179>:   ja     0x4006f0 <_ZN22xuzhina_dump_c06_s2_ex5printEv+24>
   0x0000000000400791 <+185>:   leaveq 
   0x0000000000400792 <+186>:   retq   
End of assembler dump.
```

由 ```xuzhina_dump_c06_s2_ex::print()``` 我们知道 ```print()``` 是类成员函数。因此第一个参数是 this 指针，从 ```0x00000000004006e0 <+8>: mov    %rdi,-0x18(%rbp)``` 得知 this 指针存放于 -0x18(%rbp)。

由 core 附近的语句：

```
   0x00000000004006e4 <+12>:    movl   $0x0,-0x4(%rbp)
   ...
   0x0000000000400701 <+41>:    jmp    0x400757 <_ZN22xuzhina_dump_c06_s2_ex5printEv+127>
   0x0000000000400703 <+43>:    mov    -0x18(%rbp),%rax
   0x0000000000400707 <+47>:    mov    0x18(%rax),%rdx
   0x000000000040070b <+51>:    mov    -0x4(%rbp),%eax
   0x000000000040070e <+54>:    add    %rdx,%rax
=> 0x0000000000400711 <+57>:    movzbl (%rax),%eax
```

可以推测出 this 指针 +0x18 偏移的成员变量是数组，数组索引是保存在 -0x4(%rbp) 的变量。由 ```movzbl``` 指令知道数组元素类型是 char。

通过
```
   0x00000000004006f0 <+24>:    mov    -0x18(%rbp),%rax
   0x00000000004006f4 <+28>:    movzwl (%rax),%eax
   0x00000000004006f7 <+31>:    cwtl   
   0x00000000004006f8 <+32>:    test   %eax,%eax  
   0x00000000004006fa <+34>:    je     0x400703 <_ZN22xuzhina_dump_c06_s2_ex5printEv+43>  // 等于 0 则跳转到 <+43>
   0x00000000004006fc <+36>:    cmp    $0x1,%eax  
   0x00000000004006ff <+39>:    je     0x40072a <_ZN22xuzhina_dump_c06_s2_ex5printEv+82>  // 等于 1 则跳转到 <+82>
   0x0000000000400701 <+41>:    jmp    0x400757 <_ZN22xuzhina_dump_c06_s2_ex5printEv+127>
```

我们知道当 this 对象的第一个成员变量（因为其地址就是 this 指针的地址所以是第一个成员变量）为 0 时才会执行到崩溃的代码，由 ```movzwl``` 指令知道该成员为 short 类型。

接下来看看 -0x4(%rbp) 变量的逻辑：

```
0x00000000004006e4 <+12>:    movl   $0x0,-0x4(%rbp)
...
0x000000000040077d <+165>:   addl   $0x1,-0x4(%rbp)
0x0000000000400781 <+169>:   mov    -0x18(%rbp),%rax
0x0000000000400785 <+173>:   mov    0x20(%rax),%eax
0x0000000000400788 <+176>:   cmp    -0x4(%rbp),%eax
0x000000000040078b <+179>:   ja     0x4006f0 <_ZN22xuzhina_dump_c06_s2_ex5printEv+24>
```

可以看到 -0x4(%rbp) 为计数变量，每次于 this 指针 +0x20 偏移的成员变量做比较，代码框架大致如下：

```c++
for (int i = 0; i < this->count; i++) {
    if (this->flag == 0) {
        // core 了
    } else if (this->flag == 1) {
        ...
    }
}
```

由
```   
0x0000000000400703 <+43>:    mov    -0x18(%rbp),%rax
0x0000000000400707 <+47>:    mov    0x18(%rax),%rdx
```

core 之前访问了偏移为 0x18 的成员变量。前面分析得知该成员变量是 char 数组，gdb 查看下该变量：

```
(gdb) x /2x $rbp-0x18  // this 指针
0x7ffd988dfe98: 0x988dfed0      0x00007ffd
(gdb) x /2x 0x00007ffd988dfed0  // *this 
0x7ffd988dfed0: 0x45480000      0x45454545
(gdb) x /s 0x00007ffd988dfed0+0x2  // 第一个成员为 short 类型 因此第二个成员为 *this + 0x2
0x7ffd988dfed2: "H", 'E' <repeats 45 times>
(gdb) x /2x 0x00007ffd988dfed0+0x18  // *this + 0x18 成员变量
0x7ffd988dfee8: 0x45    0x45
```

可以看到 +0x18 成员变量指向的地址不是有效的地址，被第二个成员一大串字符串 "H", 'E' <repeats 45 times> 覆盖了。

接下来看看 ```main``` 函数，分析下是在哪里被覆盖的：

```
(gdb) set print asm-demangle 
(gdb) disassemble main
Dump of assembler code for function main:
   0x0000000000400793 <+0>:     push   %rbp
   0x0000000000400794 <+1>:     mov    %rsp,%rbp
   0x0000000000400797 <+4>:     sub    $0x40,%rsp
   0x000000000040079b <+8>:     mov    %edi,-0x34(%rbp)
   0x000000000040079e <+11>:    mov    %rsi,-0x40(%rbp)
   0x00000000004007a2 <+15>:    cmpl   $0x2,-0x34(%rbp)
   0x00000000004007a6 <+19>:    jg     0x4007af <main+28>
   0x00000000004007a8 <+21>:    mov    $0xffffffff,%eax
   0x00000000004007ad <+26>:    jmp    0x400802 <main+111>
   0x00000000004007af <+28>:    mov    -0x40(%rbp),%rax
   0x00000000004007b3 <+32>:    add    $0x10,%rax
   0x00000000004007b7 <+36>:    mov    (%rax),%rax
   0x00000000004007ba <+39>:    mov    %rax,%rdi
   0x00000000004007bd <+42>:    callq  0x400570 <strlen@plt>
   0x00000000004007c2 <+47>:    mov    %eax,%ecx
   0x00000000004007c4 <+49>:    mov    -0x40(%rbp),%rax
   0x00000000004007c8 <+53>:    add    $0x10,%rax
   0x00000000004007cc <+57>:    mov    (%rax),%rdx
   0x00000000004007cf <+60>:    mov    -0x40(%rbp),%rax 
   0x00000000004007d3 <+64>:    add    $0x8,%rax
   0x00000000004007d7 <+68>:    mov    (%rax),%rsi
   0x00000000004007da <+71>:    lea    -0x30(%rbp),%rax
   0x00000000004007de <+75>:    mov    %ecx,%r8d
   0x00000000004007e1 <+78>:    mov    %rdx,%rcx
   0x00000000004007e4 <+81>:    mov    $0x0,%edx
   0x00000000004007e9 <+86>:    mov    %rax,%rdi
   0x00000000004007ec <+89>:    callq  0x400680 <xuzhina_dump_c06_s2_ex::xuzhina_dump_c06_s2_ex(char*, short, void*, unsigned int)>
   0x00000000004007f1 <+94>:    lea    -0x30(%rbp),%rax
   0x00000000004007f5 <+98>:    mov    %rax,%rdi
   0x00000000004007f8 <+101>:   callq  0x4006d8 <xuzhina_dump_c06_s2_ex::print()>
   0x00000000004007fd <+106>:   mov    $0x0,%eax
   0x0000000000400802 <+111>:   leaveq 
   0x0000000000400803 <+112>:   retq   
End of assembler dump.
```
从 main 的汇编代码发现只调用了 ```strlen```、```xuzhina_dump_c06_s2_ex::xuzhina_dump_c06_s2_ex(char*, short, void*, unsigned int)```、```xuzhina_dump_c06_s2_ex::print()``` 这三个函数。先看看构造函数：

*注：x64 参数自左向右依次传入寄存器rdi, rsi, rdx, rcx, r8*

```
(gdb) disassemble 0x400680
Dump of assembler code for function _ZN22xuzhina_dump_c06_s2_exC2EPcsPvj:
   0x0000000000400680 <+0>:     push   %rbp
   0x0000000000400681 <+1>:     mov    %rsp,%rbp
   0x0000000000400684 <+4>:     sub    $0x20,%rsp
   0x0000000000400688 <+8>:     mov    %rdi,-0x8(%rbp)  // this 指针
   0x000000000040068c <+12>:    mov    %rsi,-0x10(%rbp)  // char* 
   0x0000000000400690 <+16>:    mov    %edx,%eax  // short
   0x0000000000400692 <+18>:    mov    %rcx,-0x20(%rbp)  // void*
   0x0000000000400696 <+22>:    mov    %r8d,-0x18(%rbp) // unsigned int
   0x000000000040069a <+26>:    mov    %ax,-0x14(%rbp)  // short
   0x000000000040069e <+30>:    mov    -0x8(%rbp),%rax
   0x00000000004006a2 <+34>:    mov    -0x20(%rbp),%rdx
   0x00000000004006a6 <+38>:    mov    %rdx,0x18(%rax)
   0x00000000004006aa <+42>:    mov    -0x8(%rbp),%rax
   0x00000000004006ae <+46>:    mov    -0x18(%rbp),%edx
   0x00000000004006b1 <+49>:    mov    %edx,0x20(%rax)
   0x00000000004006b4 <+52>:    mov    -0x8(%rbp),%rax
   0x00000000004006b8 <+56>:    lea    0x2(%rax),%rdx
   0x00000000004006bc <+60>:    mov    -0x10(%rbp),%rax
   0x00000000004006c0 <+64>:    mov    %rax,%rsi
   0x00000000004006c3 <+67>:    mov    %rdx,%rdi
   0x00000000004006c6 <+70>:    callq  0x400580 <strcpy@plt>
   0x00000000004006cb <+75>:    mov    -0x8(%rbp),%rax
   0x00000000004006cf <+79>:    movzwl -0x14(%rbp),%edx
   0x00000000004006d3 <+83>:    mov    %dx,(%rax)
   0x00000000004006d6 <+86>:    leaveq 
   0x00000000004006d7 <+87>:    retq   
End of assembler dump.
```

由
```
   0x0000000000400688 <+8>:     mov    %rdi,-0x8(%rbp)
   ...
   0x000000000040069e <+30>:    mov    -0x8(%rbp),%rax
   0x00000000004006a2 <+34>:    mov    -0x20(%rbp),%rdx
   0x00000000004006a6 <+38>:    mov    %rdx,0x18(%rax)  // void* 指针 赋值给 0x18偏移 即类对象的第三个字段
   ...

   0x00000000004006aa <+42>:    mov    -0x8(%rbp),%rax
   0x00000000004006ae <+46>:    mov    -0x18(%rbp),%edx // unsigned int
   0x00000000004006b1 <+49>:    mov    %edx,0x20(%rax)  // unsigned int 赋值给 0x20偏移 即类对象的第四个字段

   0x00000000004006b4 <+52>:    mov    -0x8(%rbp),%rax
   0x00000000004006b8 <+56>:    lea    0x2(%rax),%rdx //  0x2偏移字段 
   0x00000000004006bc <+60>:    mov    -0x10(%rbp),%rax  // char*
   0x00000000004006c0 <+64>:    mov    %rax,%rsi // char* 作为 strcpy 的第二个参数
   0x00000000004006c3 <+67>:    mov    %rdx,%rdi  //  0x2偏移字段 即类对象的第二个字段 作为 strcpy 的第一个参数
   0x00000000004006c6 <+70>:    callq  0x400580 <strcpy@plt>  
   0x00000000004006cb <+75>:    mov    -0x8(%rbp),%rax  
   0x00000000004006cf <+79>:    movzwl -0x14(%rbp),%edx
   0x00000000004006d3 <+83>:    mov    %dx,(%rax) // 把 short 赋值给 0x0偏移 即类对象的第一个字段 

```

得知类对象有四个成员变量，类结构体如下：

```c++
class xuzhina_dump_c06_s2_ex {
    short flag;  // 0x0
    char arr[??];  // 0x2
    void* ptr;  // 0x18
    unsigned int count;  // 0x20
}
```

发现高危函数 ```strcpy```，猜测是 ```strcpy``` 第二个字段的时候把第三个字段给覆盖了。```strcpy``` 第二个参数是 ```main``` 调用 ```xuzhina_dump_c06_s2_ex::xuzhina_dump_c06_s2_ex``` 函数的 char* 参数。切换到 frame 1，查看  ```main``` 中传给 ```xuzhina_dump_c06_s2_ex::xuzhina_dump_c06_s2_ex``` 函数的 char* 参数：

```
(gdb) f 1
#1  0x00000000004007fd in main ()
```

由 ```main``` 汇编代码中的：
```
0x000000000040079e <+11>:    mov    %rsi,-0x40(%rbp)
...
0x00000000004007cf <+60>:    mov    -0x40(%rbp),%rax 
0x00000000004007d3 <+64>:    add    $0x8,%rax
0x00000000004007d7 <+68>:    mov    (%rax),%rsi  // 第二个参数值存于 rsi, 也就是 char*
...
0x00000000004007ec <+89>:    callq  0x400680 <xuzhina_dump_c06_s2_ex::xuzhina_dump_c06_s2_ex(char*, short, void*, unsigned int)>
```

```
(gdb) x /2x $rbp-0x40
0x7ffd988dfec0: 0x988dffe8      0x00007ffd
(gdb) x /2x 0x00007ffd988dffe8 + 0x8
0x7ffd988dfff0: 0x988e0793      0x00007ffd
(gdb) x /s 0x00007ffd988e0793
0x7ffd988e0793: "H", 'E' <repeats 45 times>
```

可见，长度超过了 0x18 导致第三个成员变量被覆盖。


源码：

```c++
#include <string.h>
#include <stdio.h>
class xuzhina_dump_c06_s2_ex {
    private:
        short m_type;
        char m_name[16];
        void* m_ptr;
        unsigned int m_len;
    public:
        xuzhina_dump_c06_s2_ex(char* name, short type, void* data, unsigned int len);
        void print();
};

xuzhina_dump_c06_s2_ex::xuzhina_dump_c06_s2_ex(char* name, short type, void* data, unsigned int len) {
    m_ptr = data;
    m_len = len;
    strcpy( m_name, name );
    m_type = type;

}

void xuzhina_dump_c06_s2_ex::print() {
    for (unsigned int i = 0; i < m_len; i++) {
        switch(m_type) {
            case 0:
                printf( "%c ", *((char*)m_ptr + i ) );
                break;
            case 1:
                printf( "%f ", *((float*)m_ptr + i ) );
                break;
            default:
                printf( "%d ", *((int*)m_ptr + i ) );
                break;
        }
    }
}

int main(int argc, char* argv[]) {
    if (argc < 3) {
        return -1;
    }

    xuzhina_dump_c06_s2_ex test(argv[1], 0, argv[2], strlen(argv[2]));
    test.print();

    return 0;
}
```

执行 ```./test HEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE 1``` 触发 coredump。