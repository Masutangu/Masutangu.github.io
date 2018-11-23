---
layout: post
date: 2018-11-23T13:00:28+08:00
title: Core dump 原理探究学习笔记（八）
tags: 
  - 读书笔记
  - 汇编
---

本系列文章是读[《coredump问题原理探究》](https://blog.csdn.net/xuzhina/article/category/1322964)的读书笔记。
 
## Set

```c++
#include <set>

int main() {
  std::set<int> iSet;
  iSet.insert(0x523);
  iSet.insert(0x352);
  iSet.insert(0x808);

  return 0;
}
```

```
(gdb) set print asm-demangle
(gdb) disassemble main
Dump of assembler code for function main:
   0x00000000004008b0 <+0>:     push   %rbp
   0x00000000004008b1 <+1>:     mov    %rsp,%rbp
   0x00000000004008b4 <+4>:     push   %rbx
   0x00000000004008b5 <+5>:     sub    $0x48,%rsp
   0x00000000004008b9 <+9>:     lea    -0x50(%rbp),%rax
   0x00000000004008bd <+13>:    mov    %rax,%rdi
   0x00000000004008c0 <+16>:    callq  0x400976 <std::set<int, std::less<int>, std::allocator<int> >::set()>
   0x00000000004008c5 <+21>:    movl   $0x523,-0x1c(%rbp)
   0x00000000004008cc <+28>:    lea    -0x1c(%rbp),%rdx
   0x00000000004008d0 <+32>:    lea    -0x50(%rbp),%rax
   0x00000000004008d4 <+36>:    mov    %rdx,%rsi
   0x00000000004008d7 <+39>:    mov    %rax,%rdi
   0x00000000004008da <+42>:    callq  0x400a04 <std::set<int, std::less<int>, std::allocator<int> >::insert(int const&)>
   0x00000000004008df <+47>:    movl   $0x352,-0x18(%rbp)
   0x00000000004008e6 <+54>:    lea    -0x18(%rbp),%rdx
   0x00000000004008ea <+58>:    lea    -0x50(%rbp),%rax
   0x00000000004008ee <+62>:    mov    %rdx,%rsi
   0x00000000004008f1 <+65>:    mov    %rax,%rdi
   0x00000000004008f4 <+68>:    callq  0x400a04 <std::set<int, std::less<int>, std::allocator<int> >::insert(int const&)>
   0x00000000004008f9 <+73>:    movl   $0x808,-0x14(%rbp)
   0x0000000000400900 <+80>:    lea    -0x14(%rbp),%rdx
   0x0000000000400904 <+84>:    lea    -0x50(%rbp),%rax
   0x0000000000400908 <+88>:    mov    %rdx,%rsi
   0x000000000040090b <+91>:    mov    %rax,%rdi
   0x000000000040090e <+94>:    callq  0x400a04 <std::set<int, std::less<int>, std::allocator<int> >::insert(int const&)>
   0x0000000000400913 <+99>:    mov    $0x0,%ebx
   0x0000000000400918 <+104>:   lea    -0x50(%rbp),%rax
   0x000000000040091c <+108>:   mov    %rax,%rdi
   0x000000000040091f <+111>:   callq  0x40095c <std::set<int, std::less<int>, std::allocator<int> >::~set()>
   0x0000000000400924 <+116>:   mov    %ebx,%eax
   0x0000000000400926 <+118>:   jmp    0x400942 <main+146>
   0x0000000000400928 <+120>:   mov    %rax,%rbx
   0x000000000040092b <+123>:   lea    -0x50(%rbp),%rax
   0x000000000040092f <+127>:   mov    %rax,%rdi
   0x0000000000400932 <+130>:   callq  0x40095c <std::set<int, std::less<int>, std::allocator<int> >::~set()>
   0x0000000000400937 <+135>:   mov    %rbx,%rax
   0x000000000040093a <+138>:   mov    %rax,%rdi
   0x000000000040093d <+141>:   callq  0x4007b0 <_Unwind_Resume@plt>
   0x0000000000400942 <+146>:   add    $0x48,%rsp
   0x0000000000400946 <+150>:   pop    %rbx
   0x0000000000400947 <+151>:   pop    %rbp
   0x0000000000400948 <+152>:   retq   
End of assembler dump.
```

构造函数打断点，观察 set 对象：

```
(gdb) tbreak *0x00000000004008c0
Temporary breakpoint 1 at 0x4008c0
(gdb) r
(gdb) x /12wx $rbp-0x50
0x7fffffffe420: 0x00000001      0x00007fff      0xf7203890      0x00007fff
0x7fffffffe430: 0x00000001      0x00000000      0x0040146d      0x00000000
0x7fffffffe440: 0x00000000      0x00000000      0x00000000      0x00000000
(gdb) ni
0x00000000004008c5 in main ()
(gdb) x /12wx $rbp-0x50
0x7fffffffe420: 0x00000001      0x00007fff      0x00000000      0x00007fff
0x7fffffffe430: 0x00000000      0x00000000      0xffffe428      0x00007fff
0x7fffffffe440: 0xffffe428      0x00007fff      0x00000000      0x00000000
```

和 map 一致。在第一处 insert 打断点：


```
(gdb) tbreak *0x00000000004008df
Temporary breakpoint 2 at 0x4008df
(gdb) c
Continuing.

Temporary breakpoint 2, 0x00000000004008df in main ()
(gdb) x /12wx $rbp-0x50
0x7fffffffe420: 0x00000001      0x00007fff      0x00000000      0x00007fff
0x7fffffffe430: 0x00604010      0x00000000      0x00604010      0x00000000
0x7fffffffe440: 0x00604010      0x00000000      0x00000001      0x00000000
(gdb) x /10wx 0x00604010
0x604010:       0x00000001      0x00000000      0xffffe428      0x00007fff
0x604020:       0x00000000      0x00000000      0x00000000      0x00000000
0x604030:       0x00000523      0x00000000
```

第二处 insert 打断点：


```
(gdb) tbreak *0x00000000004008f9
Temporary breakpoint 3 at 0x4008f9
(gdb) c
Continuing.

Temporary breakpoint 3, 0x00000000004008f9 in main ()
(gdb) x /12wx $rbp-0x50
0x7fffffffe420: 0x00000001      0x00007fff      0x00000000      0x00007fff
0x7fffffffe430: 0x00604010      0x00000000      0x00604040      0x00000000
0x7fffffffe440: 0x00604010      0x00000000      0x00000002      0x00000000
(gdb) x /10wx 0x00604010
0x604010:       0x00000001      0x00000000      0xffffe428      0x00007fff
0x604020:       0x00604040      0x00000000      0x00000000      0x00000000
0x604030:       0x00000523      0x00000000
(gdb) x /10wx 0x00604040
0x604040:       0x00000000      0x00000000      0x00604010      0x00000000
0x604050:       0x00000000      0x00000000      0x00000000      0x00000000
0x604060:       0x00000352      0x00000000
```

第三处 insert 打断点：

```
(gdb) tbreak *0x0000000000400913
Temporary breakpoint 4 at 0x400913
(gdb) c
Continuing.

Temporary breakpoint 4, 0x0000000000400913 in main ()
(gdb) x /12wx $rbp-0x50
0x7fffffffe420: 0x00000001      0x00007fff      0x00000000      0x00007fff
0x7fffffffe430: 0x00604010      0x00000000      0x00604040      0x00000000
0x7fffffffe440: 0x00604070      0x00000000      0x00000003      0x00000000
(gdb) x /10wx 0x00604040
0x604040:       0x00000000      0x00000000      0x00604010      0x00000000
0x604050:       0x00000000      0x00000000      0x00000000      0x00000000
0x604060:       0x00000352      0x00000000
(gdb) x /10wx 0x00604010
0x604010:       0x00000001      0x00000000      0xffffe428      0x00007fff
0x604020:       0x00604040      0x00000000      0x00604070      0x00000000
0x604030:       0x00000523      0x00000000
(gdb) x /10wx 0x00604070
0x604070:       0x00000000      0x00000000      0x00604010      0x00000000
0x604080:       0x00000000      0x00000000      0x00000000      0x00000000
0x604090:       0x00000808      0x00000000
```
可见 set 和 map 基本是一致的。

## String

```c++
#include <string>
#include <stdio.h>

int main() {
  std::string str;
  const char* ptr = "hello world!";

  for (int i = 0; i < 0x10; i++) {
    str.append(ptr);
  }

  return 0;
}
```

```
(gdb) set print asm-demangle
(gdb) disassemble main
Dump of assembler code for function main:
   0x00000000004007e0 <+0>:     push   %rbp
   0x00000000004007e1 <+1>:     mov    %rsp,%rbp
   0x00000000004007e4 <+4>:     push   %rbx
   0x00000000004007e5 <+5>:     sub    $0x28,%rsp
   0x00000000004007e9 <+9>:     lea    -0x30(%rbp),%rax
   0x00000000004007ed <+13>:    mov    %rax,%rdi
   0x00000000004007f0 <+16>:    callq  0x400680 <_ZNSsC1Ev@plt>
   0x00000000004007f5 <+21>:    movq   $0x4008f0,-0x20(%rbp)
   0x00000000004007fd <+29>:    movl   $0x0,-0x14(%rbp)
   0x0000000000400804 <+36>:    jmp    0x40081d <main+61>
   0x0000000000400806 <+38>:    mov    -0x20(%rbp),%rdx
   0x000000000040080a <+42>:    lea    -0x30(%rbp),%rax
   0x000000000040080e <+46>:    mov    %rdx,%rsi
   0x0000000000400811 <+49>:    mov    %rax,%rdi
   0x0000000000400814 <+52>:    callq  0x4006c0 <_ZNSs6appendEPKc@plt>
   0x0000000000400819 <+57>:    addl   $0x1,-0x14(%rbp)
   0x000000000040081d <+61>:    cmpl   $0xf,-0x14(%rbp)
   0x0000000000400821 <+65>:    jle    0x400806 <main+38>
   0x0000000000400823 <+67>:    mov    $0x0,%ebx
   0x0000000000400828 <+72>:    lea    -0x30(%rbp),%rax
   0x000000000040082c <+76>:    mov    %rax,%rdi
   0x000000000040082f <+79>:    callq  0x4006b0 <_ZNSsD1Ev@plt>
   0x0000000000400834 <+84>:    mov    %ebx,%eax
   0x0000000000400836 <+86>:    jmp    0x400852 <main+114>
   0x0000000000400838 <+88>:    mov    %rax,%rbx
   0x000000000040083b <+91>:    lea    -0x30(%rbp),%rax
   0x000000000040083f <+95>:    mov    %rax,%rdi
   0x0000000000400842 <+98>:    callq  0x4006b0 <_ZNSsD1Ev@plt>
   0x0000000000400847 <+103>:   mov    %rbx,%rax
   0x000000000040084a <+106>:   mov    %rax,%rdi
   0x000000000040084d <+109>:   callq  0x4006e0 <_Unwind_Resume@plt>
   0x0000000000400852 <+114>:   add    $0x28,%rsp
   0x0000000000400856 <+118>:   pop    %rbx
   0x0000000000400857 <+119>:   pop    %rbp
   0x0000000000400858 <+120>:   retq   
End of assembler dump.

(gdb) x /s 0x4008f0
0x4008f0:       "hello world!"
```

由：

```
0x00000000004007e9 <+9>:     lea    -0x30(%rbp),%rax
0x00000000004007ed <+13>:    mov    %rax,%rdi
0x00000000004007f0 <+16>:    callq  0x400680 <_ZNSsC1Ev@plt>
```

知道 string 的 this 指针保存在 -0x30(%rbp)。

调用

```
(gdb) tbreak *0x00000000004007f5
Temporary breakpoint 1 at 0x4007f5
(gdb) r
Temporary breakpoint 1, 0x00000000004007f5 in main ()
Missing separate debuginfos, use: debuginfo-install glibc-2.17-157.tl2.2.x86_64 libgcc-4.8.5-4.el7.x86_64 libstdc++-4.8.5-4.el7.x86_64
(gdb) x /8wx $rbp-0x30
0x7fffffffe440: 0xf7ddc3f8      0x00007fff      0x00000000      0x00000000
0x7fffffffe450: 0x00400860      0x00000000      0x004006f0      0x00000000
(gdb) x /s 0x00007ffff7ddc3f8
0x7ffff7ddc3f8 <std::string::_Rep::_S_empty_rep_storage+24>:    ""
(gdb) x /12wx 0x00007ffff7ddc3f8-24
0x7ffff7ddc3e0 <std::string::_Rep::_S_empty_rep_storage>:       0x00000000      0x00000000      0x00000000      0x00000000
0x7ffff7ddc3f0 <std::string::_Rep::_S_empty_rep_storage+16>:    0x00000000      0x00000000      0x00000000      0x00000000
0x7ffff7ddc400 <std::basic_string<wchar_t, std::char_traits<wchar_t>, std::allocator<wchar_t> >::_Rep::_S_empty_rep_storage>:   0x00000000      0x00000000      0x00000000      0x00000000
```

注意 ```<std::string::_Rep::_S_empty_rep_storage+24>``` 的 ```+24```。在 append 后打断点：

```
(gdb) break *0x0000000000400819
Breakpoint 2 at 0x400819
(gdb) c
Continuing.

Breakpoint 2, 0x0000000000400819 in main ()
(gdb) x /8wx $rbp-0x30
0x7fffffffe440: 0x00602028      0x00000000      0x00000000      0x00000000
0x7fffffffe450: 0x004008f0      0x00000000      0x004006f0      0x00000000
(gdb) x /s 0x00602028
0x602028:       "hello world!"
(gdb) x /12wx 0x00602028-24
0x602010:       0x0000000c      0x00000000      0x0000000c      0x00000000
0x602020:       0x00000000      0x00000000      0x6c6c6568      0x6f77206f
0x602030:       0x21646c72      0x00000000      0x00020fd1      0x00000000
(gdb) c
Continuing.

Breakpoint 2, 0x0000000000400819 in main ()
(gdb) x /8wx $rbp-0x30
0x7fffffffe440: 0x00602058      0x00000000      0x00000000      0x00000000
0x7fffffffe450: 0x004008f0      0x00000000      0x004006f0      0x00000001
(gdb) x /s 0x00602058
0x602058:       "hello world!hello world!"
(gdb) x /12wx 0x00602058-24
0x602040:       0x00000018      0x00000000      0x00000018      0x00000000
0x602050:       0x00000000      0x00000000      0x6c6c6568      0x6f77206f
0x602060:       0x21646c72      0x6c6c6568      0x6f77206f      0x21646c72
(gdb) c
Continuing.

Breakpoint 2, 0x0000000000400819 in main ()
(gdb) x /8wx $rbp-0x30
0x7fffffffe440: 0x00602098      0x00000000      0x00000000      0x00000000
0x7fffffffe450: 0x004008f0      0x00000000      0x004006f0      0x00000002
(gdb) x /s 0x00602098
0x602098:       "hello world!hello world!hello world!"
(gdb) x /12wx 0x00602098-24
0x602080:       0x00000024      0x00000000      0x00000030      0x00000000
0x602090:       0x00000000      0x00000000      0x6c6c6568      0x6f77206f
0x6020a0:       0x21646c72      0x6c6c6568      0x6f77206f      0x21646c72
```

可见 string 对象的第一个成员是 size，第二个成员是 capacity（this+8），第四个成员是字符串（this+24）。