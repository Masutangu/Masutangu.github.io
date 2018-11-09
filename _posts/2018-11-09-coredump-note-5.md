---
layout: post
date: 2018-11-09T13:37:39+08:00
title: Core dump 原理探究学习笔记（五）
tags: 
  - 读书笔记
  - 汇编
---

本系列文章是读[《coredump问题原理探究》](https://blog.csdn.net/xuzhina/article/category/1322964)的读书笔记。

## 多继承

```c++
#include <stdio.h>
class xuzhina_dump_c06_s5_mother {
  private:
    int m_age;
    int m_beauty;
  public:
    virtual void print() {
        printf("mother\n");
    }

    virtual void setBeauty(int age, int beauty) {
        m_age = age - 5;
        m_beauty = beauty;
    }
};

class xuzhina_dump_c06_s5_father {
  private:
    int m_strong;
    int m_age;
  public:
    virtual void print() {
        printf("father\n");
    }
    virtual void setStrong(int strong, int age) {
        m_strong = strong;
        m_age = age;
    }
};
 
class xuzhina_dump_c06_s5_child: public xuzhina_dump_c06_s5_father, 
                                 public xuzhina_dump_c06_s5_mother {
  private:
    bool m_newMind;
  public:
    virtual void print() {
        printf("child\n");
    }

    virtual void setGender(bool gender) {
      m_newMind = true;
      if (gender) {
        setBeauty(10, 10);
      } else {
        setStrong(20,20);
      }
    }
};
 
int main() {
  xuzhina_dump_c06_s5_child* child = new xuzhina_dump_c06_s5_child;
  child->setGender( false );
  child->print();

  xuzhina_dump_c06_s5_father* f = child;
  f->print();

  xuzhina_dump_c06_s5_mother* m = child;
  m->print();

  return 0;
}
```

```
(gdb) disassemble main
Dump of assembler code for function main:
   0x0000000000400740 <+0>:     push   %rbp
   0x0000000000400741 <+1>:     mov    %rsp,%rbp
   0x0000000000400744 <+4>:     push   %rbx
   0x0000000000400745 <+5>:     sub    $0x28,%rsp
   0x0000000000400749 <+9>:     mov    $0x28,%edi
   0x000000000040074e <+14>:    callq  0x400640 <_Znwm@plt>
   0x0000000000400753 <+19>:    mov    %rax,%rbx
   0x0000000000400756 <+22>:    mov    %rbx,%rdi
   0x0000000000400759 <+25>:    callq  0x400916 <_ZN25xuzhina_dump_c06_s5_childC2Ev>
   0x000000000040075e <+30>:    mov    %rbx,-0x18(%rbp)  // child 指针存放于 -0x18(%rbp) 
   0x0000000000400762 <+34>:    mov    -0x18(%rbp),%rax
   0x0000000000400766 <+38>:    mov    (%rax),%rax
   0x0000000000400769 <+41>:    add    $0x10,%rax
   0x000000000040076d <+45>:    mov    (%rax),%rax  // setGender 函数地址
   0x0000000000400770 <+48>:    mov    -0x18(%rbp),%rdx
   0x0000000000400774 <+52>:    mov    $0x0,%esi
   0x0000000000400779 <+57>:    mov    %rdx,%rdi
   0x000000000040077c <+60>:    callq  *%rax  // child->setGender( false );
   0x000000000040077e <+62>:    mov    -0x18(%rbp),%rax
   0x0000000000400782 <+66>:    mov    (%rax),%rax
   0x0000000000400785 <+69>:    mov    (%rax),%rax
   0x0000000000400788 <+72>:    mov    -0x18(%rbp),%rdx
   0x000000000040078c <+76>:    mov    %rdx,%rdi
   0x000000000040078f <+79>:    callq  *%rax  // child->print();
   0x0000000000400791 <+81>:    mov    -0x18(%rbp),%rax  // 取 child 地址 
   0x0000000000400795 <+85>:    mov    %rax,-0x20(%rbp)  // f = child; f 存储于 -0x20(%rbp)
   0x0000000000400799 <+89>:    mov    -0x20(%rbp),%rax
   0x000000000040079d <+93>:    mov    (%rax),%rax
   0x00000000004007a0 <+96>:    mov    (%rax),%rax
   0x00000000004007a3 <+99>:    mov    -0x20(%rbp),%rdx
   0x00000000004007a7 <+103>:   mov    %rdx,%rdi
   0x00000000004007aa <+106>:   callq  *%rax  // f->print();
   0x00000000004007ac <+108>:   cmpq   $0x0,-0x18(%rbp)
   0x00000000004007b1 <+113>:   je     0x4007bd <main+125>
   0x00000000004007b3 <+115>:   mov    -0x18(%rbp),%rax
   0x00000000004007b7 <+119>:   add    $0x10,%rax
   0x00000000004007bb <+123>:   jmp    0x4007c2 <main+130>
   0x00000000004007bd <+125>:   mov    $0x0,%eax
   0x00000000004007c2 <+130>:   mov    %rax,-0x28(%rbp)
   0x00000000004007c6 <+134>:   mov    -0x28(%rbp),%rax
   0x00000000004007ca <+138>:   mov    (%rax),%rax
   0x00000000004007cd <+141>:   mov    (%rax),%rax
   0x00000000004007d0 <+144>:   mov    -0x28(%rbp),%rdx
   0x00000000004007d4 <+148>:   mov    %rdx,%rdi
   0x00000000004007d7 <+151>:   callq  *%rax  // m->print();
   0x00000000004007d9 <+153>:   mov    $0x0,%eax
   0x00000000004007de <+158>:   add    $0x28,%rsp
   0x00000000004007e2 <+162>:   pop    %rbx
   0x00000000004007e3 <+163>:   pop    %rbp
   0x00000000004007e4 <+164>:   retq   
End of assembler dump.
```

关注这几行，对应的代码是 ```xuzhina_dump_c06_s5_mother* m = child;```

```
0x00000000004007ac <+108>:   cmpq   $0x0,-0x18(%rbp)
0x00000000004007b1 <+113>:   je     0x4007bd <main+125>
0x00000000004007b3 <+115>:   mov    -0x18(%rbp),%rax
0x00000000004007b7 <+119>:   add    $0x10,%rax  // child 偏移 0x10 
0x00000000004007bb <+123>:   jmp    0x4007c2 <main+130>
0x00000000004007bd <+125>:   mov    $0x0,%eax
0x00000000004007c2 <+130>:   mov    %rax,-0x28(%rbp)  // 偏移后赋值给 m
```

为什么这里不能和 ```xuzhina_dump_c06_s5_father* f = child;``` 一样，直接把 this 指针 -0x18(%rbp) 直接赋值，而是需要加上 $0x10 的偏移呢？看看子类的构造函数：

```
(gdb) disassemble 0x400916
Dump of assembler code for function _ZN25xuzhina_dump_c06_s5_childC2Ev:
   0x0000000000400916 <+0>:     push   %rbp
   0x0000000000400917 <+1>:     mov    %rsp,%rbp
   0x000000000040091a <+4>:     sub    $0x10,%rsp
   0x000000000040091e <+8>:     mov    %rdi,-0x8(%rbp)
   0x0000000000400922 <+12>:    mov    -0x8(%rbp),%rax
   0x0000000000400926 <+16>:    mov    %rax,%rdi
   0x0000000000400929 <+19>:    callq  0x4008ea <_ZN26xuzhina_dump_c06_s5_fatherC2Ev>
   0x000000000040092e <+24>:    mov    -0x8(%rbp),%rax
   0x0000000000400932 <+28>:    add    $0x10,%rax
   0x0000000000400936 <+32>:    mov    %rax,%rdi
   0x0000000000400939 <+35>:    callq  0x400900 <_ZN26xuzhina_dump_c06_s5_motherC2Ev>
   0x000000000040093e <+40>:    mov    -0x8(%rbp),%rax
   0x0000000000400942 <+44>:    movq   $0x400a30,(%rax)
   0x0000000000400949 <+51>:    mov    -0x8(%rbp),%rax
   0x000000000040094d <+55>:    movq   $0x400a58,0x10(%rax)
   0x0000000000400955 <+63>:    leaveq 
   0x0000000000400956 <+64>:    retq   
End of assembler dump.
```

可见子类 child 为两个父类 father 和 mother 的组合。从

```
0x0000000000400929 <+19>:    callq  0x4008ea <_ZN26xuzhina_dump_c06_s5_fatherC2Ev>
0x000000000040092e <+24>:    mov    -0x8(%rbp),%rax
0x0000000000400932 <+28>:    add    $0x10,%rax // 偏移 0x10
0x0000000000400936 <+32>:    mov    %rax,%rdi
0x0000000000400939 <+35>:    callq  0x400900 <_ZN26xuzhina_dump_c06_s5_motherC2Ev>
```
我们知道 father 对象大小为 0x10。


```
(gdb) x /4x 0x400a58
0x400a58 <_ZTV25xuzhina_dump_c06_s5_child+56>:  0x0040087a      0x00000000      0x004007fe      0x00000000
(gdb) x /4x 0x400a30
0x400a30 <_ZTV25xuzhina_dump_c06_s5_child+16>:  0x00400862      0x00000000      0x0040083e      0x00000000
(gdb) info symbol 0x0040087a
non-virtual thunk to xuzhina_dump_c06_s5_child::print() in section .text
(gdb) info symbol 0x004007fe
xuzhina_dump_c06_s5_mother::setBeauty(int, int) in section .text
(gdb) info symbol 0x00400862
xuzhina_dump_c06_s5_child::print() in section .text
```

结论，**多继承时，当将子类对象指针转换为基类指针，实际上是把子类对象中的基类对象的地址赋值给基类指针。**

## coredump 分析

```
(gdb) bt
#0  0x0000000000400821 in xuzhina_dump_c06_s5_ex_child::inheritFrom(char*, int) ()
#1  0x000000000040079d in main ()
(gdb) disassemble 
Dump of assembler code for function _ZN28xuzhina_dump_c06_s5_ex_child11inheritFromEPci:
   0x00000000004007ec <+0>:     push   %rbp
   0x00000000004007ed <+1>:     mov    %rsp,%rbp
   0x00000000004007f0 <+4>:     sub    $0x20,%rsp
   0x00000000004007f4 <+8>:     mov    %rdi,-0x8(%rbp)
   0x00000000004007f8 <+12>:    mov    %rsi,-0x10(%rbp)
   0x00000000004007fc <+16>:    mov    %edx,-0x14(%rbp)
   0x00000000004007ff <+19>:    mov    -0x8(%rbp),%rax
   0x0000000000400803 <+23>:    mov    (%rax),%rax
   0x0000000000400806 <+26>:    mov    (%rax),%rax
   0x0000000000400809 <+29>:    mov    -0x8(%rbp),%rdx
   0x000000000040080d <+33>:    mov    -0x10(%rbp),%rcx
   0x0000000000400811 <+37>:    mov    %rcx,%rsi
   0x0000000000400814 <+40>:    mov    %rdx,%rdi
   0x0000000000400817 <+43>:    callq  *%rax
   0x0000000000400819 <+45>:    mov    -0x8(%rbp),%rax
   0x000000000040081d <+49>:    mov    0x10(%rax),%rax
=> 0x0000000000400821 <+53>:    mov    (%rax),%rax
   0x0000000000400824 <+56>:    mov    -0x8(%rbp),%rdx
   0x0000000000400828 <+60>:    lea    0x10(%rdx),%rcx
   0x000000000040082c <+64>:    mov    -0x14(%rbp),%edx
   0x000000000040082f <+67>:    mov    %edx,%esi
   0x0000000000400831 <+69>:    mov    %rcx,%rdi
   0x0000000000400834 <+72>:    callq  *%rax
   0x0000000000400836 <+74>:    mov    -0x8(%rbp),%rax
   0x000000000040083a <+78>:    movl   $0x1,0x1c(%rax)
   0x0000000000400841 <+85>:    leaveq 
   0x0000000000400842 <+86>:    retq   
End of assembler dump.
```

this 指针存放在 -0x8(%rbp)。查看 core 附近的指令

```
(gdb) x /4x $rbp-0x8
0x7ffca5ea45c8: 0x01070010      0x00000000      0xa5ea4610      0x00007ffc
(gdb) x /4x 0x01070010+0x10
0x1070020:      0x6548646c      0x576f6c6c      0x646c726f      0x00000000  // 每个字节都小于0x80 猜测是 ascii 码
(gdb) x /s 0x01070010+0x10
0x1070020:      "ldHelloWorld"
```

分析 core 之后的指令：

```
   0x0000000000400819 <+45>:    mov    -0x8(%rbp),%rax
   0x000000000040081d <+49>:    mov    0x10(%rax),%rax
=> 0x0000000000400821 <+53>:    mov    (%rax),%rax
   0x0000000000400824 <+56>:    mov    -0x8(%rbp),%rdx
   0x0000000000400828 <+60>:    lea    0x10(%rdx),%rcx
   0x000000000040082c <+64>:    mov    -0x14(%rbp),%edx
   0x000000000040082f <+67>:    mov    %edx,%esi
   0x0000000000400831 <+69>:    mov    %rcx,%rdi
   0x0000000000400834 <+72>:    callq  *%rax
```

看起来 rax 应该保存的是虚函数的指针，然而上面发现 rax 寄存器的内容是 ```ldHelloWorld``` 字符串，意味着 this 指针 +0x10 偏移的虚函数指针被字符串所覆盖。

core 之前有调用虚函数的汇编代码：

```
0x00000000004007ff <+19>:    mov    -0x8(%rbp),%rax
0x0000000000400803 <+23>:    mov    (%rax),%rax
0x0000000000400806 <+26>:    mov    (%rax),%rax
0x0000000000400809 <+29>:    mov    -0x8(%rbp),%rdx
0x000000000040080d <+33>:    mov    -0x10(%rbp),%rcx
0x0000000000400811 <+37>:    mov    %rcx,%rsi
0x0000000000400814 <+40>:    mov    %rdx,%rdi
0x0000000000400817 <+43>:    callq  *%rax
```

猜测是调用该虚函数之后溢出覆盖了虚函数表导致 core 的，看看调用了哪个虚函数：

```
(gdb) x /4x $rbp-0x8
0x7ffca5ea45c8: 0x01070010      0x00000000      0xa5ea4610      0x00007ffc
(gdb) x /4wx 0x01070010
0x1070010:      0x00400970      0x00000000      0x6c6c6548      0x726f576f
(gdb) x /4wx 0x00400970
0x400970 <_ZTV28xuzhina_dump_c06_s5_ex_child+16>:       0x004007aa      0x00000000      0x004007ec      0x00000000
(gdb) info symbol 0x004007aa
xuzhina_dump_c06_s5_ex_father::setName(char*) in section .text of /data/home/tmp/test
```

查看下 setName 的汇编代码：

```
(gdb) disassemble 0x004007aa
Dump of assembler code for function _ZN29xuzhina_dump_c06_s5_ex_father7setNameEPc:
   0x00000000004007aa <+0>:     push   %rbp
   0x00000000004007ab <+1>:     mov    %rsp,%rbp
   0x00000000004007ae <+4>:     sub    $0x10,%rsp
   0x00000000004007b2 <+8>:     mov    %rdi,-0x8(%rbp)  // this 指针
   0x00000000004007b6 <+12>:    mov    %rsi,-0x10(%rbp)  // char* 参数
   0x00000000004007ba <+16>:    mov    -0x8(%rbp),%rax
   0x00000000004007be <+20>:    lea    0x8(%rax),%rdx  // this 指针 +0x8 偏移的成员变量
   0x00000000004007c2 <+24>:    mov    -0x10(%rbp),%rax
   0x00000000004007c6 <+28>:    mov    %rax,%rsi
   0x00000000004007c9 <+31>:    mov    %rdx,%rdi
   0x00000000004007cc <+34>:    callq  0x400630 <strcpy@plt>  // 把 char* strcpy 到 this + 0x8 的成员变量
   0x00000000004007d1 <+39>:    leaveq 
   0x00000000004007d2 <+40>:    retq   
End of assembler dump.
```

发现高危函数 ```strcpy```，setName 会把 char* 拷贝到 this 指针的 +0x8 偏移的成员变量。推断是因为 char* 超过该成员变量的大小导致覆盖到 this+0x10 的虚函数指针。


查下 setName 的参数 char* 的值：
```
0x00000000004007f8 <+12>:    mov    %rsi,-0x10(%rbp)  // xuzhina_dump_c06_s5_ex_child::inheritFrom 的第一个参数 存放在 -0x10(%rbp)
...
0x000000000040080d <+33>:    mov    -0x10(%rbp),%rcx
0x0000000000400811 <+37>:    mov    %rcx,%rsi  // -0x10(%rbp) 作为第一个参数传给 setName
0x0000000000400814 <+40>:    mov    %rdx,%rdi
0x0000000000400817 <+43>:    callq  *%rax
```

可以看出 char* 是 ```main``` 传给 ```xuzhina_dump_c06_s5_ex_child::inheritFrom``` 的第一个参数，切换到 f 1 看看：

```
(gdb) f 1
#1  0x000000000040079d in main ()
(gdb) disassemble
Dump of assembler code for function main:
   0x0000000000400740 <+0>:     push   %rbp
   0x0000000000400741 <+1>:     mov    %rsp,%rbp
   0x0000000000400744 <+4>:     push   %rbx
   0x0000000000400745 <+5>:     sub    $0x28,%rsp
   0x0000000000400749 <+9>:     mov    %edi,-0x24(%rbp)
   0x000000000040074c <+12>:    mov    %rsi,-0x30(%rbp)
   0x0000000000400750 <+16>:    cmpl   $0x1,-0x24(%rbp)
   0x0000000000400754 <+20>:    jg     0x40075d <main+29>
   0x0000000000400756 <+22>:    mov    $0xffffffff,%eax
   0x000000000040075b <+27>:    jmp    0x4007a2 <main+98>
   0x000000000040075d <+29>:    mov    $0x20,%edi
   0x0000000000400762 <+34>:    callq  0x400640 <_Znwm@plt>
   0x0000000000400767 <+39>:    mov    %rax,%rbx
   0x000000000040076a <+42>:    mov    %rbx,%rdi
   0x000000000040076d <+45>:    callq  0x400870 <_ZN28xuzhina_dump_c06_s5_ex_childC2Ev>
   0x0000000000400772 <+50>:    mov    %rbx,-0x18(%rbp)
   0x0000000000400776 <+54>:    mov    -0x18(%rbp),%rax
   0x000000000040077a <+58>:    mov    (%rax),%rax
   0x000000000040077d <+61>:    add    $0x8,%rax
   0x0000000000400781 <+65>:    mov    (%rax),%rax
   0x0000000000400784 <+68>:    mov    -0x30(%rbp),%rdx
   0x0000000000400788 <+72>:    add    $0x8,%rdx
   0x000000000040078c <+76>:    mov    (%rdx),%rsi
   0x000000000040078f <+79>:    mov    -0x18(%rbp),%rcx
   0x0000000000400793 <+83>:    mov    $0x1,%edx
   0x0000000000400798 <+88>:    mov    %rcx,%rdi
   0x000000000040079b <+91>:    callq  *%rax
=> 0x000000000040079d <+93>:    mov    $0x0,%eax
   0x00000000004007a2 <+98>:    add    $0x28,%rsp
   0x00000000004007a6 <+102>:   pop    %rbx
   0x00000000004007a7 <+103>:   pop    %rbp
   0x00000000004007a8 <+104>:   retq   
End of assembler dump.
```

从

```
   0x0000000000400784 <+68>:    mov    -0x30(%rbp),%rdx
   0x0000000000400788 <+72>:    add    $0x8,%rdx
   0x000000000040078c <+76>:    mov    (%rdx),%rsi  // char* 的地址为 -0x30(%rbp) + 0x8
   0x000000000040078f <+79>:    mov    -0x18(%rbp),%rcx
   0x0000000000400793 <+83>:    mov    $0x1,%edx
   0x0000000000400798 <+88>:    mov    %rcx,%rdi  // this 指针
   0x000000000040079b <+91>:    callq  *%rax
```

确定 char* 的地址为 -0x30(%rbp) + 0x8，gdb 打印下：

```
(gdb) x /2wx $rbp-0x30
0x7ffca5ea45e0: 0xa5ea46f8      0x00007ffc
(gdb) x /2wx 0x00007ffca5ea46f8+0x8
0x7ffca5ea4700: 0xa5ea67af      0x00007ffc
(gdb) x /s 0x00007ffca5ea67af  
0x7ffca5ea67af: "HelloWorldHelloWorld"
```

得出 char* 的值为 ```HelloWorldHelloWorld```，长度为 20，很明显 0x8 + 20 = 28 超过了 0x10，调用 strcpy 会导致字符串覆盖到 0x10 的虚函数指针。

源码：

```c++
#include <string.h>
class xuzhina_dump_c06_s5_ex_father {
  private:
    char m_name[8];
  public:
    virtual void setName(char* name) {
      strcpy(m_name, name);
    }
};

class xuzhina_dump_c06_s5_ex_mother {
  private:
    int m_nature;
  public:
    virtual void setNature(int nature) {
      m_nature = nature;
    }
};

class xuzhina_dump_c06_s5_ex_child: public xuzhina_dump_c06_s5_ex_father,
                                    public xuzhina_dump_c06_s5_ex_mother {
  private:
    int m_sweet;
  public:
    virtual void inheritFrom(char* lastName, int nature) {
        setName(lastName);
        setNature(nature);
        m_sweet = 1;
    }
};

int main(int argc, char* argv[])
{
  if (argc < 2) {
    return -1;
  }

  xuzhina_dump_c06_s5_ex_child* child = new xuzhina_dump_c06_s5_ex_child;
  child->inheritFrom(argv[1], 1);

  return 0;
}
```

```./test HelloWorldHelloWorld``` 触发 coredump。