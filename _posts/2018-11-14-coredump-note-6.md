---
layout: post
date: 2018-11-14T12:52:06+08:00
title: Core dump 原理探究学习笔记（六）
tags: 
  - 读书笔记
  - 汇编
---

本系列文章是读[《coredump问题原理探究》](https://blog.csdn.net/xuzhina/article/category/1322964)的读书笔记。
 
```c++

#include <vector>

int main() {
  std::vector<int> vec;
  vec.push_back(0xffeeffab);
  vec.push_back(0xabcdef01);
  vec.push_back(0x12345678);
  return 0;
}
```

```
(gdb) r
Starting program: /data/home/tmp/test 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Breakpoint 1, 0x0000000000400954 in main ()
Missing separate debuginfos, use: debuginfo-install glibc-2.17-157.tl2.2.x86_64 libgcc-4.8.5-4.el7.x86_64 libstdc++-4.8.5-4.el7.x86_64
(gdb) set print asm-demangle 
(gdb) disassemble
Dump of assembler code for function main:
   0x0000000000400950 <+0>:     push   %rbp
   0x0000000000400951 <+1>:     mov    %rsp,%rbp
=> 0x0000000000400954 <+4>:     push   %rbx
   0x0000000000400955 <+5>:     sub    $0x38,%rsp
   0x0000000000400959 <+9>:     lea    -0x40(%rbp),%rax
   0x000000000040095d <+13>:    mov    %rax,%rdi
   0x0000000000400960 <+16>:    callq  0x4009fc <std::vector<int, std::allocator<int> >::vector()>
   0x0000000000400965 <+21>:    movl   $0xffeeffab,-0x1c(%rbp)
   0x000000000040096c <+28>:    lea    -0x1c(%rbp),%rdx
   0x0000000000400970 <+32>:    lea    -0x40(%rbp),%rax
   0x0000000000400974 <+36>:    mov    %rdx,%rsi
   0x0000000000400977 <+39>:    mov    %rax,%rdi
   0x000000000040097a <+42>:    callq  0x400a7c <std::vector<int, std::allocator<int> >::push_back(int const&)>
   0x000000000040097f <+47>:    movl   $0xabcdef01,-0x18(%rbp)
   0x0000000000400986 <+54>:    lea    -0x18(%rbp),%rdx
   0x000000000040098a <+58>:    lea    -0x40(%rbp),%rax
   0x000000000040098e <+62>:    mov    %rdx,%rsi
   0x0000000000400991 <+65>:    mov    %rax,%rdi
   0x0000000000400994 <+68>:    callq  0x400a7c <std::vector<int, std::allocator<int> >::push_back(int const&)>
   0x0000000000400999 <+73>:    movl   $0x12345678,-0x14(%rbp)
   0x00000000004009a0 <+80>:    lea    -0x14(%rbp),%rdx
   0x00000000004009a4 <+84>:    lea    -0x40(%rbp),%rax
   0x00000000004009a8 <+88>:    mov    %rdx,%rsi
   0x00000000004009ab <+91>:    mov    %rax,%rdi
   0x00000000004009ae <+94>:    callq  0x400a7c <std::vector<int, std::allocator<int> >::push_back(int const&)>
   0x00000000004009b3 <+99>:    mov    $0x0,%ebx
   0x00000000004009b8 <+104>:   lea    -0x40(%rbp),%rax
   0x00000000004009bc <+108>:   mov    %rax,%rdi
   0x00000000004009bf <+111>:   callq  0x400a16 <std::vector<int, std::allocator<int> >::~vector()>
   0x00000000004009c4 <+116>:   mov    %ebx,%eax
   0x00000000004009c6 <+118>:   jmp    0x4009e2 <main+146>
   0x00000000004009c8 <+120>:   mov    %rax,%rbx
   0x00000000004009cb <+123>:   lea    -0x40(%rbp),%rax
   0x00000000004009cf <+127>:   mov    %rax,%rdi
   0x00000000004009d2 <+130>:   callq  0x400a16 <std::vector<int, std::allocator<int> >::~vector()>
   0x00000000004009d7 <+135>:   mov    %rbx,%rax
   0x00000000004009da <+138>:   mov    %rax,%rdi
   0x00000000004009dd <+141>:   callq  0x400850 <_Unwind_Resume@plt>
   0x00000000004009e2 <+146>:   add    $0x38,%rsp
   0x00000000004009e6 <+150>:   pop    %rbx
   0x00000000004009e7 <+151>:   pop    %rbp
   0x00000000004009e8 <+152>:   retq   
End of assembler dump.
```

从
```
   0x0000000000400959 <+9>:     lea    -0x40(%rbp),%rax
   0x000000000040095d <+13>:    mov    %rax,%rdi
   0x0000000000400960 <+16>:    callq  0x4009fc <std::vector<int, std::allocator<int> >::vector()>
```

得知 vector 的 this 指针存放于 -0x40(%rbp)。在各个成员函数调用处打断点观察：

```
(gdb) tbreak *0x0000000000400960
Note: breakpoint 2 also set at pc 0x400960.
Temporary breakpoint 3 at 0x400960
(gdb) c
Continuing.
(gdb) x /8wx $rbp-0x40
0x7fffffffe430: 0x00000001      0x00000000      0x004016ed      0x00000000
0x7fffffffe440: 0x00000000      0x00000000      0x00000000      0x00000000
(gdb) ni
0x0000000000400965 in main ()
(gdb) x /8wx $rbp-0x40
0x7fffffffe430: 0x00000000      0x00000000      0x00000000      0x00000000
0x7fffffffe440: 0x00000000      0x00000000      0x00000000      0x00000000
```

构造函数完成后，vector 指向的内存都初始化为 0。

```
(gdb) tbreak *0x000000000040097a
Temporary breakpoint 4 at 0x40097a
(gdb) c
Continuing.

Temporary breakpoint 4, 0x000000000040097a in main ()
(gdb) x /8x $rbp-0x40
0x7fffffffe430: 0x00000000      0x00000000      0x00000000      0x00000000
0x7fffffffe440: 0x00000000      0x00000000      0x00000000      0x00000000
(gdb) ni
0x000000000040097f in main ()
(gdb) x /8x $rbp-0x40
0x7fffffffe430: 0x00604010      0x00000000      0x00604014      0x00000000
0x7fffffffe440: 0x00604014      0x00000000      0x00000000      0x00000000
(gdb) x /4wx  0x00604010
0x604010:       0xffeeffab      0x00000000      0x00000000      0x00000000
```

第一次调用 ```push_back``` 之后，vector 的第一个成员指向的内存存放了 pusb back 的值 ```0xffeeffab```，vector 的第二个成员变成 ```0x00604014```，和 ```0x00604010``` 相差 4 字节。

```
(gdb) tbreak *0x0000000000400994
Temporary breakpoint 2 at 0x400994
(gdb) c
Continuing.

Temporary breakpoint 2, 0x0000000000400994 in main ()
(gdb) x /8wx $rbp-0x40
0x7fffffffe430: 0x00604010      0x00000000      0x00604014      0x00000000
0x7fffffffe440: 0x00604014      0x00000000      0x00000000      0x00000000
(gdb) ni
0x0000000000400999 in main ()
(gdb) x /8wx $rbp-0x40
0x7fffffffe430: 0x00604030      0x00000000      0x00604038      0x00000000
0x7fffffffe440: 0x00604038      0x00000000      0x00000000      0x00000000
```

第二个 ```push_back``` 之后，第一个成员指向的值变成了 ```0x00604030```，但这个地址指向的内容还是一样，并且后面还存放着第二个 push back 进去的值 ```0xabcdef01```：

```
(gdb) x /4wx 0x00604030
0x604030:       0xffeeffab      0xabcdef01      0x00000000      0x00000000
```

vector 的第二个成员变成了 ```0x00604038```，和 ```0x00604030``` 相差 8 字节。

```
(gdb) tbreak *0x00000000004009ae
Temporary breakpoint 3 at 0x4009ae
(gdb) c
Continuing.

Temporary breakpoint 3, 0x00000000004009ae in main ()
(gdb) x /8wx $rbp-0x40
0x7fffffffe430: 0x00604030      0x00000000      0x00604038      0x00000000
0x7fffffffe440: 0x00604038      0x00000000      0x00000000      0x00000000
(gdb) ni
0x00000000004009b3 in main ()
(gdb) x /8wx $rbp-0x40
0x7fffffffe430: 0x00604010      0x00000000      0x0060401c      0x00000000
0x7fffffffe440: 0x00604020      0x00000000      0x00000000      0x00000000
(gdb) x /4wx 0x00604010
0x604010:       0xffeeffab      0xabcdef01      0x12345678      0x00000000
```

第三个 ```push_back``` 之后，第一个成员的指向的地址再次发生变化，变成了 ```0x00604010```，指向的内容不变，并且还新增了第三个 push back 的值 ```0x12345678```。第二个成员 ```0x0060401c``` 和第一个成员 ```0x00604010``` 相差 12个字节。

因此可以推断，vector 的第一个成员指向 begin()，第二个成员指向 end()。

## coredump 分析

```
(gdb) bt
#0  0x0000000000400dd8 in __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::merge<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >) ()
#1  0x0000000000400b23 in main ()
```


```
(gdb) disassemble
Dump of assembler code for function _ZSt5mergeIN9__gnu_cxx17__normal_iteratorIPiSt6vectorIiSaIiEEEES6_S6_ET1_T_S8_T0_S9_S7_:
   0x0000000000400d47 <+0>:     push   %rbp
   0x0000000000400d48 <+1>:     mov    %rsp,%rbp
   0x0000000000400d4b <+4>:     push   %rbx
   0x0000000000400d4c <+5>:     sub    $0x58,%rsp
   0x0000000000400d50 <+9>:     mov    %rdi,-0x20(%rbp)
   0x0000000000400d54 <+13>:    mov    %rsi,-0x30(%rbp)
   0x0000000000400d58 <+17>:    mov    %rdx,-0x40(%rbp)
   0x0000000000400d5c <+21>:    mov    %rcx,-0x50(%rbp)
   0x0000000000400d60 <+25>:    mov    %r8,-0x60(%rbp)
   0x0000000000400d64 <+29>:    jmpq   0x400df2 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::merge<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >)+171>
   0x0000000000400d69 <+34>:    lea    -0x40(%rbp),%rax
   0x0000000000400d6d <+38>:    mov    %rax,%rdi
   0x0000000000400d70 <+41>:    callq  0x4012b0 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator*() const>
   0x0000000000400d75 <+46>:    mov    (%rax),%ebx
   0x0000000000400d77 <+48>:    lea    -0x20(%rbp),%rax
   0x0000000000400d7b <+52>:    mov    %rax,%rdi
   0x0000000000400d7e <+55>:    callq  0x4012b0 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator*() const>
   0x0000000000400d83 <+60>:    mov    (%rax),%eax
   0x0000000000400d85 <+62>:    cmp    %eax,%ebx
   0x0000000000400d87 <+64>:    setl   %al
   0x0000000000400d8a <+67>:    test   %al,%al
   0x0000000000400d8c <+69>:    je     0x400dbb <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::merge<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >)+116>
   0x0000000000400d8e <+71>:    lea    -0x60(%rbp),%rax
   0x0000000000400d92 <+75>:    mov    %rax,%rdi
   0x0000000000400d95 <+78>:    callq  0x4012b0 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator*() const>
   0x0000000000400d9a <+83>:    mov    %rax,%rbx
   0x0000000000400d9d <+86>:    lea    -0x40(%rbp),%rax
   0x0000000000400da1 <+90>:    mov    %rax,%rdi
   0x0000000000400da4 <+93>:    callq  0x4012b0 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator*() const>
   0x0000000000400da9 <+98>:    mov    (%rax),%eax
   0x0000000000400dab <+100>:   mov    %eax,(%rbx)
   0x0000000000400dad <+102>:   lea    -0x40(%rbp),%rax
   0x0000000000400db1 <+106>:   mov    %rax,%rdi
   0x0000000000400db4 <+109>:   callq  0x4012c2 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator++()>
   0x0000000000400db9 <+114>:   jmp    0x400de6 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::merge<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >)+159>
   0x0000000000400dbb <+116>:   lea    -0x60(%rbp),%rax
   0x0000000000400dbf <+120>:   mov    %rax,%rdi
   0x0000000000400dc2 <+123>:   callq  0x4012b0 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator*() const>
   0x0000000000400dc7 <+128>:   mov    %rax,%rbx
   0x0000000000400dca <+131>:   lea    -0x20(%rbp),%rax
   0x0000000000400dce <+135>:   mov    %rax,%rdi
   0x0000000000400dd1 <+138>:   callq  0x4012b0 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator*() const>
   0x0000000000400dd6 <+143>:   mov    (%rax),%eax
=> 0x0000000000400dd8 <+145>:   mov    %eax,(%rbx)
   0x0000000000400dda <+147>:   lea    -0x20(%rbp),%rax
   0x0000000000400dde <+151>:   mov    %rax,%rdi
   0x0000000000400de1 <+154>:   callq  0x4012c2 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator++()>
   0x0000000000400de6 <+159>:   lea    -0x60(%rbp),%rax
   0x0000000000400dea <+163>:   mov    %rax,%rdi
   0x0000000000400ded <+166>:   callq  0x4012c2 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator++()>
   0x0000000000400df2 <+171>:   lea    -0x30(%rbp),%rdx
   0x0000000000400df6 <+175>:   lea    -0x20(%rbp),%rax
   0x0000000000400dfa <+179>:   mov    %rdx,%rsi
   0x0000000000400dfd <+182>:   mov    %rax,%rdi
   0x0000000000400e00 <+185>:   callq  0x401274 <bool __gnu_cxx::operator!=<int*, std::vector<int, std::allocator<int> > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > const&, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > const&)>
   0x0000000000400e05 <+190>:   test   %al,%al
   0x0000000000400e07 <+192>:   je     0x400e27 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::merge<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >)+224>
   0x0000000000400e09 <+194>:   lea    -0x50(%rbp),%rdx
   0x0000000000400e0d <+198>:   lea    -0x40(%rbp),%rax
   0x0000000000400e11 <+202>:   mov    %rdx,%rsi
   0x0000000000400e14 <+205>:   mov    %rax,%rdi
   0x0000000000400e17 <+208>:   callq  0x401274 <bool __gnu_cxx::operator!=<int*, std::vector<int, std::allocator<int> > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > const&, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > const&)>
   0x0000000000400e1c <+213>:   test   %al,%al
   0x0000000000400e1e <+215>:   je     0x400e27 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::merge<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >)+224>
   0x0000000000400e20 <+217>:   mov    $0x1,%eax
   0x0000000000400e25 <+222>:   jmp    0x400e2c <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::merge<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >)+229>
   0x0000000000400e27 <+224>:   mov    $0x0,%eax
   0x0000000000400e2c <+229>:   test   %al,%al
   0x0000000000400e2e <+231>:   jne    0x400d69 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::merge<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >)+34>
   0x0000000000400e34 <+237>:   mov    -0x60(%rbp),%rdx
   0x0000000000400e38 <+241>:   mov    -0x30(%rbp),%rcx
   0x0000000000400e3c <+245>:   mov    -0x20(%rbp),%rax
   0x0000000000400e40 <+249>:   mov    %rcx,%rsi
   0x0000000000400e43 <+252>:   mov    %rax,%rdi
   0x0000000000400e46 <+255>:   callq  0x4012e2 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::copy<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >)>
   0x0000000000400e4b <+260>:   mov    %rax,%rdx
   0x0000000000400e4e <+263>:   mov    -0x50(%rbp),%rcx
   0x0000000000400e52 <+267>:   mov    -0x40(%rbp),%rax
   0x0000000000400e56 <+271>:   mov    %rcx,%rsi
   0x0000000000400e59 <+274>:   mov    %rax,%rdi
   0x0000000000400e5c <+277>:   callq  0x4012e2 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::copy<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >)>
   0x0000000000400e61 <+282>:   add    $0x58,%rsp
   0x0000000000400e65 <+286>:   pop    %rbx
   0x0000000000400e66 <+287>:   pop    %rbp
   0x0000000000400e67 <+288>:   retq   
End of assembler dump.
```

由

```
   0x0000000000400dbb <+116>:   lea    -0x60(%rbp),%rax
   0x0000000000400dbf <+120>:   mov    %rax,%rdi
   0x0000000000400dc2 <+123>:   callq  0x4012b0 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator*() const>

   0x0000000000400dc7 <+128>:   mov    %rax,%rbx
   0x0000000000400dca <+131>:   lea    -0x20(%rbp),%rax
   0x0000000000400dce <+135>:   mov    %rax,%rdi
   0x0000000000400dd1 <+138>:   callq  0x4012b0 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator*() const>
   0x0000000000400dd6 <+143>:   mov    (%rax),%eax
=> 0x0000000000400dd8 <+145>:   mov    %eax,(%rbx)
```

可见崩溃时的指令是 ```=> 0x0000000000400dd8 <+145>:   mov    %eax,(%rbx)```
eax 和 rbx 正好是调用了 ```__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> >``` 函数的返回值，而调用这个函数的 this 指针分别是 -0x20(%rbp) 和 -0x60(%rbp)，从

```
   0x0000000000400d50 <+9>:     mov    %rdi,-0x20(%rbp)
   ...
   0x0000000000400d60 <+25>:    mov    %r8,-0x60(%rbp)
```
知道是 ```_ZSt5mergeIN9__gnu_cxx17__normal_iteratorIPiSt6vectorIiSaIiEEEES6_S6_ET1_T_S8_T0_S9_S7_``` 函数的第一个参数和第五个参数。

```
(gdb) f 1
#1  0x0000000000400b23 in main ()
(gdb) disassemble main
Dump of assembler code for function main:
   0x0000000000400a60 <+0>:     push   %rbp
   0x0000000000400a61 <+1>:     mov    %rsp,%rbp
   0x0000000000400a64 <+4>:     push   %r14
   0x0000000000400a66 <+6>:     push   %r13
   0x0000000000400a68 <+8>:     push   %r12
   0x0000000000400a6a <+10>:    push   %rbx
   0x0000000000400a6b <+11>:    sub    $0x60,%rsp
   0x0000000000400a6f <+15>:    lea    -0x40(%rbp),%rax
   0x0000000000400a73 <+19>:    mov    %rax,%rdi
   0x0000000000400a76 <+22>:    callq  0x400bfe <std::vector<int, std::allocator<int> >::vector()>
   0x0000000000400a7b <+27>:    movl   $0x1,-0x28(%rbp)
   0x0000000000400a82 <+34>:    lea    -0x28(%rbp),%rdx
   0x0000000000400a86 <+38>:    lea    -0x40(%rbp),%rax
   0x0000000000400a8a <+42>:    mov    %rdx,%rsi
   0x0000000000400a8d <+45>:    mov    %rax,%rdi
   0x0000000000400a90 <+48>:    callq  0x400c7e <std::vector<int, std::allocator<int> >::push_back(int const&)>
   0x0000000000400a95 <+53>:    lea    -0x60(%rbp),%rax
   0x0000000000400a99 <+57>:    mov    %rax,%rdi
   0x0000000000400a9c <+60>:    callq  0x400bfe <std::vector<int, std::allocator<int> >::vector()>
   0x0000000000400aa1 <+65>:    movl   $0x8,-0x24(%rbp)
   0x0000000000400aa8 <+72>:    lea    -0x24(%rbp),%rdx
   0x0000000000400aac <+76>:    lea    -0x60(%rbp),%rax
   0x0000000000400ab0 <+80>:    mov    %rdx,%rsi
   0x0000000000400ab3 <+83>:    mov    %rax,%rdi
   0x0000000000400ab6 <+86>:    callq  0x400c7e <std::vector<int, std::allocator<int> >::push_back(int const&)>
   0x0000000000400abb <+91>:    lea    -0x80(%rbp),%rax
   0x0000000000400abf <+95>:    mov    %rax,%rdi
   0x0000000000400ac2 <+98>:    callq  0x400bfe <std::vector<int, std::allocator<int> >::vector()>
   0x0000000000400ac7 <+103>:   lea    -0x80(%rbp),%rax
   0x0000000000400acb <+107>:   mov    %rax,%rdi
   0x0000000000400ace <+110>:   callq  0x400cf8 <std::vector<int, std::allocator<int> >::begin()>
   0x0000000000400ad3 <+115>:   mov    %rax,%r14
   0x0000000000400ad6 <+118>:   lea    -0x60(%rbp),%rax
   0x0000000000400ada <+122>:   mov    %rax,%rdi
   0x0000000000400add <+125>:   callq  0x400d1e <std::vector<int, std::allocator<int> >::end()>
   0x0000000000400ae2 <+130>:   mov    %rax,%r13
   0x0000000000400ae5 <+133>:   lea    -0x60(%rbp),%rax
   0x0000000000400ae9 <+137>:   mov    %rax,%rdi
   0x0000000000400aec <+140>:   callq  0x400cf8 <std::vector<int, std::allocator<int> >::begin()>
   0x0000000000400af1 <+145>:   mov    %rax,%r12
   0x0000000000400af4 <+148>:   lea    -0x40(%rbp),%rax
   0x0000000000400af8 <+152>:   mov    %rax,%rdi
   0x0000000000400afb <+155>:   callq  0x400d1e <std::vector<int, std::allocator<int> >::end()>
   0x0000000000400b00 <+160>:   mov    %rax,%rbx
   0x0000000000400b03 <+163>:   lea    -0x40(%rbp),%rax
   0x0000000000400b07 <+167>:   mov    %rax,%rdi
   0x0000000000400b0a <+170>:   callq  0x400cf8 <std::vector<int, std::allocator<int> >::begin()>
   0x0000000000400b0f <+175>:   mov    %r14,%r8
   0x0000000000400b12 <+178>:   mov    %r13,%rcx
   0x0000000000400b15 <+181>:   mov    %r12,%rdx
   0x0000000000400b18 <+184>:   mov    %rbx,%rsi
   0x0000000000400b1b <+187>:   mov    %rax,%rdi
   0x0000000000400b1e <+190>:   callq  0x400d47 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::merge<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > >(__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >)>
=> 0x0000000000400b23 <+195>:   mov    $0x0,%ebx
   0x0000000000400b28 <+200>:   lea    -0x80(%rbp),%rax
   0x0000000000400b2c <+204>:   mov    %rax,%rdi
   0x0000000000400b2f <+207>:   callq  0x400c18 <std::vector<int, std::allocator<int> >::~vector()>
   0x0000000000400b34 <+212>:   lea    -0x60(%rbp),%rax
   0x0000000000400b38 <+216>:   mov    %rax,%rdi
   0x0000000000400b3b <+219>:   callq  0x400c18 <std::vector<int, std::allocator<int> >::~vector()>
   0x0000000000400b40 <+224>:   lea    -0x40(%rbp),%rax
   0x0000000000400b44 <+228>:   mov    %rax,%rdi
   0x0000000000400b47 <+231>:   callq  0x400c18 <std::vector<int, std::allocator<int> >::~vector()>
   0x0000000000400b4c <+236>:   mov    %ebx,%eax
   0x0000000000400b4e <+238>:   jmp    0x400b8c <main+300>
   0x0000000000400b50 <+240>:   mov    %rax,%rbx
   0x0000000000400b53 <+243>:   lea    -0x80(%rbp),%rax
   0x0000000000400b57 <+247>:   mov    %rax,%rdi
   0x0000000000400b5a <+250>:   callq  0x400c18 <std::vector<int, std::allocator<int> >::~vector()>
   0x0000000000400b5f <+255>:   jmp    0x400b64 <main+260>
   0x0000000000400b61 <+257>:   mov    %rax,%rbx
   0x0000000000400b64 <+260>:   lea    -0x60(%rbp),%rax
   0x0000000000400b68 <+264>:   mov    %rax,%rdi
   0x0000000000400b6b <+267>:   callq  0x400c18 <std::vector<int, std::allocator<int> >::~vector()>
   0x0000000000400b70 <+272>:   jmp    0x400b75 <main+277>
   0x0000000000400b72 <+274>:   mov    %rax,%rbx
   0x0000000000400b75 <+277>:   lea    -0x40(%rbp),%rax
   0x0000000000400b79 <+281>:   mov    %rax,%rdi
   0x0000000000400b7c <+284>:   callq  0x400c18 <std::vector<int, std::allocator<int> >::~vector()>
   0x0000000000400b81 <+289>:   mov    %rbx,%rax
   0x0000000000400b84 <+292>:   mov    %rax,%rdi
   0x0000000000400b87 <+295>:   callq  0x400960 <_Unwind_Resume@plt>
   0x0000000000400b8c <+300>:   add    $0x60,%rsp
   0x0000000000400b90 <+304>:   pop    %rbx
   0x0000000000400b91 <+305>:   pop    %r12
   0x0000000000400b93 <+307>:   pop    %r13
   0x0000000000400b95 <+309>:   pop    %r14
   0x0000000000400b97 <+311>:   pop    %rbp
   0x0000000000400b98 <+312>:   retq   
End of assembler dump.
```

由下面这段汇编
```
   0x0000000000400ac7 <+103>:   lea    -0x80(%rbp),%rax
   0x0000000000400acb <+107>:   mov    %rax,%rdi
   0x0000000000400ace <+110>:   callq  0x400cf8 <std::vector<int, std::allocator<int> >::begin()>
   0x0000000000400ad3 <+115>:   mov    %rax,%r14
   ...
   0x0000000000400b03 <+163>:   lea    -0x40(%rbp),%rax
   0x0000000000400b07 <+167>:   mov    %rax,%rdi
   0x0000000000400b0a <+170>:   callq  0x400cf8 <std::vector<int, std::allocator<int> >::begin()>
   ...
   0x0000000000400b0f <+175>:   mov    %r14,%r8
   ...
   0x0000000000400b1b <+187>:   mov    %rax,%rdi
```

可以看到，传给 ```__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > ``` 的第一个参数，正是 ```<std::vector<int, std::allocator<int> >::begin()>``` 的返回值，而调用 ```<std::vector<int, std::allocator<int> >::begin()>``` 的 this 指针，是变量  -0x40(%rbp)。

```
(gdb) x /6wx $rbp-0x40
0x7ffee5fc2f40: 0x018ab010      0x00000000      0x018ab014      0x00000000
0x7ffee5fc2f50: 0x018ab014      0x00000000
```

而第五个参数，也是调用的 ```<std::vector<int, std::allocator<int> >::begin()>``` 的返回值，调用的 this 指针为 -0x80(%rbp)：

```
(gdb) x /6wx $rbp-0x80
0x7ffee5fc2f00: 0x00000000      0x00000000      0x00000000      0x00000000
0x7ffee5fc2f10: 0x00000000      0x00000000
```

可见这个 vector 只是执行了构造函数，并没有申请空间。调用 merge 会导致 core。

源码：

```c++
#include <vector>
#include <algorithm>
#include <iostream>

int main()
{
  std::vector<int> a;
  a.push_back(1);

  std::vector<int> b;
  b.push_back(8);

  std::vector<int> c;
  std::merge(a.begin(), a.end(), b.begin(), b.end(), c.begin());

  return 0;
}                         
```