---
layout: post
date: 2018-11-08T22:30:08+08:00
title: Core dump 原理探究学习笔记（四）
tags: 
  - 读书笔记
  - 汇编
---

本系列文章是读[《coredump问题原理探究》](https://blog.csdn.net/xuzhina/article/category/1322964)的读书笔记。

## 虚函数
```c++
#include <stdio.h>
class xuzhina_dump_c06_s3 {
private:
  int m_a;
public:
  xuzhina_dump_c06_s3() { m_a = 0; }
  virtual void inc() { m_a++; }
  virtual void dec() { m_a--; }
  virtual void print() {
      printf( "%d\n", m_a );
  }
};

int main()
{
  xuzhina_dump_c06_s3* test = new xuzhina_dump_c06_s3;
  if (test != NULL) {
    test->inc();
    test->inc();
    test->print();
  }
  return 0;
}
```


```
(gdb) set print asm-demangle 
(gdb) disassemble main
Dump of assembler code for function main:
   0x00000000004006e0 <+0>:     push   %rbp
   0x00000000004006e1 <+1>:     mov    %rsp,%rbp
   0x00000000004006e4 <+4>:     push   %rbx
   0x00000000004006e5 <+5>:     sub    $0x18,%rsp
   0x00000000004006e9 <+9>:     mov    $0x10,%edi
   0x00000000004006ee <+14>:    callq  0x4005e0 <_Znwm@plt>
   0x00000000004006f3 <+19>:    mov    %rax,%rbx
   0x00000000004006f6 <+22>:    mov    %rbx,%rdi
   0x00000000004006f9 <+25>:    callq  0x400752 <xuzhina_dump_c06_s3::xuzhina_dump_c06_s3()>
   0x00000000004006fe <+30>:    mov    %rbx,-0x18(%rbp)
   0x0000000000400702 <+34>:    cmpq   $0x0,-0x18(%rbp)
   0x0000000000400707 <+39>:    je     0x400746 <main+102>
   0x0000000000400709 <+41>:    mov    -0x18(%rbp),%rax
   0x000000000040070d <+45>:    mov    (%rax),%rax
   0x0000000000400710 <+48>:    mov    (%rax),%rax
   0x0000000000400713 <+51>:    mov    -0x18(%rbp),%rdx
   0x0000000000400717 <+55>:    mov    %rdx,%rdi
   0x000000000040071a <+58>:    callq  *%rax
   0x000000000040071c <+60>:    mov    -0x18(%rbp),%rax
   0x0000000000400720 <+64>:    mov    (%rax),%rax
   0x0000000000400723 <+67>:    mov    (%rax),%rax
   0x0000000000400726 <+70>:    mov    -0x18(%rbp),%rdx
   0x000000000040072a <+74>:    mov    %rdx,%rdi
   0x000000000040072d <+77>:    callq  *%rax
   0x000000000040072f <+79>:    mov    -0x18(%rbp),%rax
   0x0000000000400733 <+83>:    mov    (%rax),%rax
   0x0000000000400736 <+86>:    add    $0x10,%rax
   0x000000000040073a <+90>:    mov    (%rax),%rax
   0x000000000040073d <+93>:    mov    -0x18(%rbp),%rdx
   0x0000000000400741 <+97>:    mov    %rdx,%rdi
   0x0000000000400744 <+100>:   callq  *%rax
   0x0000000000400746 <+102>:   mov    $0x0,%eax
   0x000000000040074b <+107>:   add    $0x18,%rsp
   0x000000000040074f <+111>:   pop    %rbx
   0x0000000000400750 <+112>:   pop    %rbp
   0x0000000000400751 <+113>:   retq   
End of assembler dump.
```
由
```
   0x00000000004006ee <+14>:    callq  0x4005e0 <_Znwm@plt>
   0x00000000004006f3 <+19>:    mov    %rax,%rbx
```
知道 test 变量保存在 rbx 寄存器。在调用构造函数前后打下断点：

```
(gdb) tbreak *0x00000000004006f6
Temporary breakpoint 1 at 0x4006f6
(gdb) tbreak * 0x00000000004006fe
Temporary breakpoint 2 at 0x4006fe
```

```
(gdb) r
(gdb) x /4x $rbx
0x602010:       0x00000000      0x00000000      0x00000000      0x00000000
(gdb) c
Continuing.

Temporary breakpoint 2, 0x00000000004006fe in main ()
(gdb) x /4x $rbx
0x602010:       0x00400890      0x00000000      0x00000000      0x00000000
```

可以看到，构造函数执行成功后，rbx 寄存器地址指向的内存发生了变化。正常来说该地址应该是 m_a，可以看到构造函数 m_a 被赋值为 0，但这里 0x602010 存放的值却是 0x00400890。

查看下构造函数的汇编：

```
(gdb) disassemble 0x400752
Dump of assembler code for function _ZN19xuzhina_dump_c06_s3C2Ev:
   0x0000000000400752 <+0>:     push   %rbp
   0x0000000000400753 <+1>:     mov    %rsp,%rbp
   0x0000000000400756 <+4>:     mov    %rdi,-0x8(%rbp)
   0x000000000040075a <+8>:     mov    -0x8(%rbp),%rax
   0x000000000040075e <+12>:    movq   $0x400890,(%rax)
   0x0000000000400765 <+19>:    mov    -0x8(%rbp),%rax
   0x0000000000400769 <+23>:    movl   $0x0,0x8(%rax)
   0x0000000000400770 <+30>:    pop    %rbp
   0x0000000000400771 <+31>:    retq   
End of assembler dump.
```

可以看到 0x400890 是构造函数中赋值进去的，而由 ```0x0000000000400769 <+23>:    movl   $0x0,0x8(%rax)``` 发现 m_a 成员变量位于类对象的 +0x8 偏移位置。查看 0x400890：

```
(gdb) x /8x 0x00400890
0x400890 <vtable for xuzhina_dump_c06_s3+16>:   0x00400772      0x00000000      0x0040078e      0x00000000
0x4008a0 <vtable for xuzhina_dump_c06_s3+32>:   0x004007aa      0x00000000      0x00000000      0x00000000
(gdb) info symbol 0x00400772
xuzhina_dump_c06_s3::inc() in section .text of /data/home/tmp/test
(gdb) info symbol 0x0040078e
xuzhina_dump_c06_s3::dec() in section .text of /data/home/tmp/test
(gdb) info symbol 0x004007aa
xuzhina_dump_c06_s3::print() in section .text of /data/home/tmp/test
```

因此 0x400890 地址指向的是虚函数表指针，保存各个虚函数的地址。从 ```main``` 的汇编也可以验证： 

```
0x0000000000400709 <+41>:    mov    -0x18(%rbp),%rax  // this 指针，同时也指向虚函数表指针
0x000000000040070d <+45>:    mov    (%rax),%rax
0x0000000000400710 <+48>:    mov    (%rax),%rax
0x0000000000400713 <+51>:    mov    -0x18(%rbp),%rdx
0x0000000000400717 <+55>:    mov    %rdx,%rdi
0x000000000040071a <+58>:    callq  *%rax  // 调用 inc()

0x000000000040071c <+60>:    mov    -0x18(%rbp),%rax  // this 指针，同时也指向虚函数表指针
0x0000000000400720 <+64>:    mov    (%rax),%rax
0x0000000000400723 <+67>:    mov    (%rax),%rax
0x0000000000400726 <+70>:    mov    -0x18(%rbp),%rdx
0x000000000040072a <+74>:    mov    %rdx,%rdi
0x000000000040072d <+77>:    callq  *%rax // 调用 inc()

0x000000000040072f <+79>:    mov    -0x18(%rbp),%rax  // this 指针，同时也指向虚函数表指针
0x0000000000400733 <+83>:    mov    (%rax),%rax
0x0000000000400736 <+86>:    add    $0x10,%rax  // 虚函数表指针 +0x10，指向 print()
0x000000000040073a <+90>:    mov    (%rax),%rax
0x000000000040073d <+93>:    mov    -0x18(%rbp),%rdx
0x0000000000400741 <+97>:    mov    %rdx,%rdi
0x0000000000400744 <+100>:   callq  *%rax  // 调用 print()
```

## 继承

```c++
#include <stdio.h>
class xuzhina_dump_c06_s4_base {
  private:
    int m_a;
  public:
    xuzhina_dump_c06_s4_base() { m_a = 1; }
    
    virtual void inc() {
        m_a++;
    }

    virtual void print() {
      printf("m_a:%d\n", m_a);
    }
};

class xuzhina_dump_c06_s4_derived: public xuzhina_dump_c06_s4_base {
  private:
    int m_b;
    int m_c;
  public:
    xuzhina_dump_c06_s4_derived() {
      m_b = 0;
      m_c = 2*m_b;
    }

    virtual void mul() {
      m_c *= m_b;
    }

    virtual void print() {
      printf("m_b:%d, m_c:%d\n", m_b, m_c);
    }

    virtual void dec() {
      m_c -= m_b;
    }

    virtual void inc() {
      m_b++;
      m_c += m_b;
    }
};

int main()
{
  xuzhina_dump_c06_s4_base* p = new xuzhina_dump_c06_s4_derived;
  if (p != NULL) {
    p->inc();
    p->print();
  }

  return 0;
}
```

直接看构造函数的汇编：

```
(gdb) disassemble 0x400802
Dump of assembler code for function _ZN27xuzhina_dump_c06_s4_derivedC2Ev:
   0x0000000000400802 <+0>:     push   %rbp
   0x0000000000400803 <+1>:     mov    %rsp,%rbp
   0x0000000000400806 <+4>:     sub    $0x10,%rsp
   0x000000000040080a <+8>:     mov    %rdi,-0x8(%rbp)
   0x000000000040080e <+12>:    mov    -0x8(%rbp),%rax
   0x0000000000400812 <+16>:    mov    %rax,%rdi
   0x0000000000400815 <+19>:    callq  0x4007a0 <_ZN24xuzhina_dump_c06_s4_baseC2Ev>
   0x000000000040081a <+24>:    mov    -0x8(%rbp),%rax
   0x000000000040081e <+28>:    movq   $0x4009d0,(%rax)
   0x0000000000400825 <+35>:    mov    -0x8(%rbp),%rax
   0x0000000000400829 <+39>:    movl   $0x0,0xc(%rax)
   0x0000000000400830 <+46>:    mov    -0x8(%rbp),%rax
   0x0000000000400834 <+50>:    mov    0xc(%rax),%eax
   0x0000000000400837 <+53>:    lea    (%rax,%rax,1),%edx
   0x000000000040083a <+56>:    mov    -0x8(%rbp),%rax
   0x000000000040083e <+60>:    mov    %edx,0x10(%rax)
   0x0000000000400841 <+63>:    leaveq 
   0x0000000000400842 <+64>:    retq   
End of assembler dump.
(gdb) disassemble 0x4007a0
Dump of assembler code for function _ZN24xuzhina_dump_c06_s4_baseC2Ev:
   0x00000000004007a0 <+0>:     push   %rbp
   0x00000000004007a1 <+1>:     mov    %rsp,%rbp
   0x00000000004007a4 <+4>:     mov    %rdi,-0x8(%rbp)
   0x00000000004007a8 <+8>:     mov    -0x8(%rbp),%rax
   0x00000000004007ac <+12>:    movq   $0x400a10,(%rax)
   0x00000000004007b3 <+19>:    mov    -0x8(%rbp),%rax
   0x00000000004007b7 <+23>:    movl   $0x1,0x8(%rax)
   0x00000000004007be <+30>:    pop    %rbp
   0x00000000004007bf <+31>:    retq   
End of assembler dump.
```

可以看出，基类的虚函数表指针为 0x400a10，子类的虚函数表指针为 0x4009d0。

```
(gdb) tbreak *0x00000000004007b7
Temporary breakpoint 2 at 0x4007b7
(gdb) c
Continuing.
(gdb) i r rax  
rax            0x602010 6299664
(gdb) x /2x 0x602010  
0x602010:       0x00400a10      0x00000000  // 0x602010 存放了父类的虚函数指针
(gdb) tbreak *0x000000000040081e 
Temporary breakpoint 3 at 0x40081e
(gdb) c
Continuing.

Temporary breakpoint 3, 0x000000000040081e in xuzhina_dump_c06_s4_derived::xuzhina_dump_c06_s4_derived() ()
(gdb) i r rax
rax            0x602010 6299664
(gdb) si
0x0000000000400825 in xuzhina_dump_c06_s4_derived::xuzhina_dump_c06_s4_derived() ()
(gdb) x /2x 0x602010
0x602010:       0x004009d0      0x00000000 // 0x602010 被覆盖成了子类的虚函数指针
```

发现子类构造函数中会覆盖了父类的虚函数表指针。

那如果子类没有重写父类的虚函数的情况呢？修改 xuzhina_dump_c06_s4_derived 如下：

```c++
// 不重写 inc()
class xuzhina_dump_c06_s4_derived: public xuzhina_dump_c06_s4_base {
  private:
    int m_b;
    int m_c;
  public:
    xuzhina_dump_c06_s4_derived() {
      m_b = 0;
      m_c = 2*m_b;
    }

    virtual void mul() {
      m_c *= m_b;
    }

    virtual void print() {
      printf("m_b:%d, m_c:%d\n", m_b, m_c);
    }

    virtual void dec() {
      m_c -= m_b;
    }
};
```

gdb 看下：

```
(gdb) disassemble 0x400802
Dump of assembler code for function _ZN27xuzhina_dump_c06_s4_derivedC2Ev:
   0x0000000000400802 <+0>:     push   %rbp
   0x0000000000400803 <+1>:     mov    %rsp,%rbp
   0x0000000000400806 <+4>:     sub    $0x10,%rsp
   0x000000000040080a <+8>:     mov    %rdi,-0x8(%rbp)
   0x000000000040080e <+12>:    mov    -0x8(%rbp),%rax
   0x0000000000400812 <+16>:    mov    %rax,%rdi
   0x0000000000400815 <+19>:    callq  0x4007a0 <_ZN24xuzhina_dump_c06_s4_baseC2Ev>
   0x000000000040081a <+24>:    mov    -0x8(%rbp),%rax
   0x000000000040081e <+28>:    movq   $0x400990,(%rax)
   0x0000000000400825 <+35>:    mov    -0x8(%rbp),%rax
   0x0000000000400829 <+39>:    movl   $0x0,0xc(%rax)
   0x0000000000400830 <+46>:    mov    -0x8(%rbp),%rax
   0x0000000000400834 <+50>:    mov    0xc(%rax),%eax
   0x0000000000400837 <+53>:    lea    (%rax,%rax,1),%edx
   0x000000000040083a <+56>:    mov    -0x8(%rbp),%rax
   0x000000000040083e <+60>:    mov    %edx,0x10(%rax)
   0x0000000000400841 <+63>:    leaveq 
   0x0000000000400842 <+64>:    retq   
End of assembler dump.
(gdb) disassemble 0x4007a0
Dump of assembler code for function _ZN24xuzhina_dump_c06_s4_baseC2Ev:
   0x00000000004007a0 <+0>:     push   %rbp
   0x00000000004007a1 <+1>:     mov    %rsp,%rbp
   0x00000000004007a4 <+4>:     mov    %rdi,-0x8(%rbp)
   0x00000000004007a8 <+8>:     mov    -0x8(%rbp),%rax
   0x00000000004007ac <+12>:    movq   $0x4009d0,(%rax)
   0x00000000004007b3 <+19>:    mov    -0x8(%rbp),%rax
   0x00000000004007b7 <+23>:    movl   $0x1,0x8(%rax)
   0x00000000004007be <+30>:    pop    %rbp
   0x00000000004007bf <+31>:    retq   
End of assembler dump.
(gdb) x /4wx 0x4009d0  // 父类虚函数表指针
0x4009d0 <_ZTV24xuzhina_dump_c06_s4_base+16>:   0x004007c0      0x00000000      0x004007dc      0x00000000
(gdb) x /4wx 0x400990  // 子类虚函数表指针
0x400990 <_ZTV27xuzhina_dump_c06_s4_derived+16>:        0x004007c0      0x00000000      0x00400866      0x00000000
(gdb) info symbol 0x004007c0
xuzhina_dump_c06_s4_base::inc() in section .text  // 子类的虚函数表指针中的 inc() 函数地址与父类中的虚函数表中一致
```

可以看出，没有被重写的虚函数地址会被拷贝到子类的虚函数表中。