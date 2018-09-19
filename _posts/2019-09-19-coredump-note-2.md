---
layout: post
date: 2018-09-19T21:15:05+08:00
title: Core dump 原理探究学习笔记（二）
category: 读书笔记
---

本系列文章是读[《coredump问题原理探究》](https://blog.csdn.net/xuzhina/article/category/1322964)的读书笔记。


## 函数逆向

这是将要逆向的样例源码：

```c++
#include <stdio.h>
#include <string.h>
int main(int argc, char* argv[]) {
  for (int i = 0; i < argc; i++) {
    int len = strlen(argv[i]);
    switch (len) {
    case 0:
      printf("%c\n", argv[i][0]);
      break;
    case 1:
      printf("%s\n", argv[i+1]);
      break;
    case 2:
      printf("%d\n", i);
      break;
    default:
      printf("%s\n", i);
      break;
    }
  }  
  return 0;
}
```

```./test helloworld``` 运行程序 core 掉。

```
(gdb) bt
#0  0x00007f1128e62694 in vfprintf () from /lib64/libc.so.6
#1  0x00007f1128e6b879 in printf () from /lib64/libc.so.6
#2  0x000000000040074d in main ()
(gdb) disassemble main
Dump of assembler code for function main:
   0x0000000000400680 <+0>:     push   %rbp
   0x0000000000400681 <+1>:     mov    %rsp,%rbp
   0x0000000000400684 <+4>:     sub    $0x20,%rsp
   0x0000000000400688 <+8>:     mov    %edi,-0x14(%rbp)
   0x000000000040068b <+11>:    mov    %rsi,-0x20(%rbp)
   0x000000000040068f <+15>:    movl   $0x0,-0x4(%rbp)
   0x0000000000400696 <+22>:    jmpq   0x400752 <main+210>
   0x000000000040069b <+27>:    mov    -0x4(%rbp),%eax
   0x000000000040069e <+30>:    cltq   
   0x00000000004006a0 <+32>:    lea    0x0(,%rax,8),%rdx
   0x00000000004006a8 <+40>:    mov    -0x20(%rbp),%rax
   0x00000000004006ac <+44>:    add    %rdx,%rax
   0x00000000004006af <+47>:    mov    (%rax),%rax
   0x00000000004006b2 <+50>:    mov    %rax,%rdi
   0x00000000004006b5 <+53>:    callq  0x400580 <strlen@plt>
   0x00000000004006ba <+58>:    mov    %eax,-0x8(%rbp)
   0x00000000004006bd <+61>:    mov    -0x8(%rbp),%eax
   0x00000000004006c0 <+64>:    cmp    $0x1,%eax
   0x00000000004006c3 <+67>:    je     0x4006fe <main+126>
   0x00000000004006c5 <+69>:    cmp    $0x2,%eax
   0x00000000004006c8 <+72>:    je     0x400723 <main+163>
   0x00000000004006ca <+74>:    test   %eax,%eax
   0x00000000004006cc <+76>:    jne    0x400739 <main+185>
   0x00000000004006ce <+78>:    mov    -0x4(%rbp),%eax
   0x00000000004006d1 <+81>:    cltq   
   0x00000000004006d3 <+83>:    lea    0x0(,%rax,8),%rdx
   0x00000000004006db <+91>:    mov    -0x20(%rbp),%rax
   0x00000000004006df <+95>:    add    %rdx,%rax
   0x00000000004006e2 <+98>:    mov    (%rax),%rax
   0x00000000004006e5 <+101>:   movzbl (%rax),%eax
   0x00000000004006e8 <+104>:   movsbl %al,%eax
   0x00000000004006eb <+107>:   mov    %eax,%esi
   0x00000000004006ed <+109>:   mov    $0x400800,%edi
   0x00000000004006f2 <+114>:   mov    $0x0,%eax
   0x00000000004006f7 <+119>:   callq  0x400540 <printf@plt>
   0x00000000004006fc <+124>:   jmp    0x40074e <main+206>
   0x00000000004006fe <+126>:   mov    -0x4(%rbp),%eax
   0x0000000000400701 <+129>:   cltq   
   0x0000000000400703 <+131>:   add    $0x1,%rax
   0x0000000000400707 <+135>:   lea    0x0(,%rax,8),%rdx
   0x000000000040070f <+143>:   mov    -0x20(%rbp),%rax
   0x0000000000400713 <+147>:   add    %rdx,%rax
   0x0000000000400716 <+150>:   mov    (%rax),%rax
   0x0000000000400719 <+153>:   mov    %rax,%rdi
   0x000000000040071c <+156>:   callq  0x400560 <puts@plt>
   0x0000000000400721 <+161>:   jmp    0x40074e <main+206>
   0x0000000000400723 <+163>:   mov    -0x4(%rbp),%eax
   0x0000000000400726 <+166>:   mov    %eax,%esi
   0x0000000000400728 <+168>:   mov    $0x400804,%edi
   0x000000000040072d <+173>:   mov    $0x0,%eax
   0x0000000000400732 <+178>:   callq  0x400540 <printf@plt>
   0x0000000000400737 <+183>:   jmp    0x40074e <main+206>
   0x0000000000400739 <+185>:   mov    -0x4(%rbp),%eax
   0x000000000040073c <+188>:   mov    %eax,%esi
   0x000000000040073e <+190>:   mov    $0x400808,%edi
   0x0000000000400743 <+195>:   mov    $0x0,%eax
   0x0000000000400748 <+200>:   callq  0x400540 <printf@plt>
   0x000000000040074d <+205>:   nop
   0x000000000040074e <+206>:   addl   $0x1,-0x4(%rbp)
   0x0000000000400752 <+210>:   mov    -0x4(%rbp),%eax
   0x0000000000400755 <+213>:   cmp    -0x14(%rbp),%eax
   0x0000000000400758 <+216>:   jl     0x40069b <main+27>
   0x000000000040075e <+222>:   mov    $0x0,%eax
   0x0000000000400763 <+227>:   leaveq 
   0x0000000000400764 <+228>:   retq   
End of assembler dump.
```

由 
```0x0000000000400758 <+216>:   jl     0x40069b <main+27>```
推断出 \<main+27\> 到 \<main+216\> 构成一个循环。


看看循环前的判断语句：

```
0x000000000040074e <+206>:   addl   $0x1,-0x4(%rbp)
0x0000000000400752 <+210>:   mov    -0x4(%rbp),%eax
0x0000000000400755 <+213>:   cmp    -0x14(%rbp),%eax
0x0000000000400758 <+216>:   jl     0x40069b <main+27>
```

从main 开头的汇编可以看出，edi 寄存器保存了 argc, rsi 寄存器保存了 argv：

```
0x0000000000400688 <+8>:     mov    %edi,-0x14(%rbp)
0x000000000040068b <+11>:    mov    %rsi,-0x20(%rbp)
```

因此循环的判断语句是对比 0x4(%rbp) 这个变量和 argc，命名这个变量为 cnt。上面判断的语句语义为：执行 cnt++，如果 cnt 小于 argc 则跳转到 \<main+27\>。

由：

```
0x000000000040068f <+15>:    movl   $0x0,-0x4(%rbp)
0x0000000000400696 <+22>:    jmpq   0x400752 <main+210>
```
可以看出，程序初始时 cnt 被赋值为 0，然后跳转到循环开始处 \<main+210\> 与 argc 比较。

由：

```
0x000000000040075e <+222>:   mov    $0x0,%eax
0x0000000000400763 <+227>:   leaveq 
```

知道 main 函数始终返回 0（eax 寄存器保存函数返回值）。

因此目前我们推算出的源码架构为：

```c++
int main(int argc, char* argv[]) {
  int cnt = 0;
  for (; cnt < args; cnt++) {
      <main+32> - <main+206>
  }
}
```

继续看看 \<main+32\> 到 \<main+206\> 的汇编。重点关注跳转指令：

```
0x00000000004006b5 <+53>:    callq  0x400580 <strlen@plt>
0x00000000004006ba <+58>:    mov    %eax,-0x8(%rbp)
0x00000000004006bd <+61>:    mov    -0x8(%rbp),%eax
0x00000000004006c0 <+64>:    cmp    $0x1,%eax
0x00000000004006c3 <+67>:    je     0x4006fe <main+126>
0x00000000004006c5 <+69>:    cmp    $0x2,%eax
0x00000000004006c8 <+72>:    je     0x400723 <main+163>
0x00000000004006ca <+74>:    test   %eax,%eax
0x00000000004006cc <+76>:    jne    0x400739 <main+185>
```

上面的代码根据 eax 寄存器的值来跳转对应的代码，而 eax 保存的是调用 strlen 函数的返回值。看看 strlen 的入参：

```
0x000000000040069b <+27>:    mov    -0x4(%rbp),%eax
0x000000000040069e <+30>:    cltq   
0x00000000004006a0 <+32>:    lea    0x0(,%rax,8),%rdx
0x00000000004006a8 <+40>:    mov    -0x20(%rbp),%rax
0x00000000004006ac <+44>:    add    %rdx,%rax
0x00000000004006af <+47>:    mov    (%rax),%rax
0x00000000004006b2 <+50>:    mov    %rax,%rdi
0x00000000004006b5 <+53>:    callq  0x400580 <strlen@plt>
```

由上面知道 -0x4(%rbp) 是变量 cnt，-0x20(%rbp) 保存 rsi 的值也就是 argv，这段汇编表示取 argv[cnt] 的值，并存入寄存器 rdi 当作 strlen 函数的参数。

此时推算出的 main 函数如下：

```c++
int main(int argc, char* argv[]) {
  int cnt = 0;
  for (; cnt < args; cnt++) {
      int i = strlen(argv[cnt]);
      switch (i) {
      case 1:
        <main+126> - <main+156>
        break;
      case 2:
        <main+163> - <main+178>
        break;
      case 0:
        <main+78> - <main+119>
        break;
      default:
        <main+185> - <main+205>
      }
  }
}
```


依次查看每个 case 的汇编。

**case 1：**

```
0x00000000004006fe <+126>:   mov    -0x4(%rbp),%eax
0x0000000000400701 <+129>:   cltq   
0x0000000000400703 <+131>:   add    $0x1,%rax
0x0000000000400707 <+135>:   lea    0x0(,%rax,8),%rdx
0x000000000040070f <+143>:   mov    -0x20(%rbp),%rax
0x0000000000400713 <+147>:   add    %rdx,%rax
0x0000000000400716 <+150>:   mov    (%rax),%rax
0x0000000000400719 <+153>:   mov    %rax,%rdi
0x000000000040071c <+156>:   callq  0x400560 <puts@plt>
```

取 argv[cnt + 1] 的值入栈，调用 puts@plt。即：

```
puts(argv[cnt + 1]);
```

**case 2：**

```
0x0000000000400723 <+163>:   mov    -0x4(%rbp),%eax
0x0000000000400726 <+166>:   mov    %eax,%esi
0x0000000000400728 <+168>:   mov    $0x400804,%edi
0x000000000040072d <+173>:   mov    $0x0,%eax
0x0000000000400732 <+178>:   callq  0x400540 <printf@plt>
```

调用 printf 函数，esi 保存参数 cnt，edi 保存参数 $0x400804，gdb 下 $0x400804 指向的内容：

```
(gdb) x /s 0x400804
0x400804:       "%d\n"
```

即：

```
printf("%d\n", cnt);
```

**case 0：**


```
0x00000000004006ce <+78>:    mov    -0x4(%rbp),%eax
0x00000000004006d1 <+81>:    cltq   
0x00000000004006d3 <+83>:    lea    0x0(,%rax,8),%rdx
0x00000000004006db <+91>:    mov    -0x20(%rbp),%rax
0x00000000004006df <+95>:    add    %rdx,%rax
0x00000000004006e2 <+98>:    mov    (%rax),%rax
0x00000000004006e5 <+101>:   movzbl (%rax),%eax
0x00000000004006e8 <+104>:   movsbl %al,%eax
0x00000000004006eb <+107>:   mov    %eax,%esi
0x00000000004006ed <+109>:   mov    $0x400800,%edi
0x00000000004006f2 <+114>:   mov    $0x0,%eax
0x00000000004006f7 <+119>:   callq  0x400540 <printf@plt>
```

<+78> 到 <+98> 是取 argv[cnt]。\<+101\> 到 \<+104\> 则为 argv[cnt][0]。gdb 0x400800 地址：

```
(gdb) x /s 0x400800
0x400800:       "%c\n"
```

即为：
```
printf("%c\n", argv[cnt][0]);
```

**default：**

```  
0x0000000000400739 <+185>:   mov    -0x4(%rbp),%eax
0x000000000040073c <+188>:   mov    %eax,%esi
0x000000000040073e <+190>:   mov    $0x400808,%edi
0x0000000000400743 <+195>:   mov    $0x0,%eax
0x0000000000400748 <+200>:   callq  0x400540 <printf@plt>
0x000000000040074d <+205>:   nop
```

gdb 0x400808：

```
(gdb) x /s 0x400808
0x400808:       "%s\n"
```

即

```
printf("%s\n", cnt);
```

因此逆向出来的 main 函数如下：

```c++
int main(int argc, char* argv[]) {
  int cnt = 0;
  for (; cnt < args; cnt++) {
      int i = strlen(argv[cnt]);
      switch (i) {
      case 1:
        puts(argv[cnt + 1]);
        break;
      case 2:
        printf("%d\n", cnt);
        break;
      case 0:
        printf("%c\n", argv[cnt][0]);
        break;
      default:
        printf("%s\n", cnt);
        break;
      }
  }
}
```

和上面我们给的源码几乎一致。

由 于core 时栈地址 0x000000000040074d 位于 \<+188\> 到 \<+205\> 的范围，即 default 分支的代码。根据我们逆向的 default 分支的代码：

```
printf("%s\n", cnt);
```

确实会引起崩溃。