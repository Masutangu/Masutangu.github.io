---
layout: post
date: 2018-11-21T21:28:19+08:00
title: Core dump 原理探究学习笔记（七）
tags: 
  - 读书笔记
  - 汇编
---

本系列文章是读[《coredump问题原理探究》](https://blog.csdn.net/xuzhina/article/category/1322964)的读书笔记。
 
## Map

```c++
#include <map>

int main() {
  std::map<int,int> m;

  m[5] = 6;
  m[8] = 20;
  m[2] = 80;

  return 0;
}
```

参考 stl 的源码：

```c++
// stl_map.h
template <typename _Key, typename _Tp, typename _Compare = std::less<_Key>,
          typename _Alloc = std::allocator<std::pair<const _Key, _Tp> > >
class map {
  public:
    typedef _Compare key_compare;
  public:
   map() : _M_t() { }
  private:
    typedef _Rb_tree<key_type, value_type, _Select1st<value_type>,
		       key_compare, _Pair_alloc_type> _Rep_type;

      /// The actual tree structure.
    _Rep_type _M_t;
};

// stl_tree.h
struct _Rb_tree_node_base
{
  typedef _Rb_tree_node_base* _Base_ptr;

  _Rb_tree_color  _M_color;
  _Base_ptr       _M_parent;
  _Base_ptr       _M_left;
  _Base_ptr       _M_right;
};

template<typename _Key, typename _Val, typename _KeyOfValue,
	   typename _Compare, typename _Alloc = allocator<_Val> >
class _Rb_tree {
  public:
    _Rb_tree() { }

  protected:
    template<typename _Key_compare, 
        bool _Is_pod_comparator = __is_pod(_Key_compare)>
    struct _Rb_tree_impl : public _Node_allocator {
        _Key_compare          _M_key_compare;
        _Rb_tree_node_base    _M_header;
        size_type             _M_node_count; // Keeps track of size of tree.

        _Rb_tree_impl()
        : _Node_allocator(), _M_key_compare(), _M_header(),
          _M_node_count(0)
        { _M_initialize(); }

      private:
        void _M_initialize() {
          this->_M_header._M_color = _S_red;
          this->_M_header._M_parent = 0;
          this->_M_header._M_left = &this->_M_header;
          this->_M_header._M_right = &this->_M_header;
         }     
      };

    _Rb_tree_impl<_Compare> _M_impl;
};

template<typename _Val>
struct _Rb_tree_node : public _Rb_tree_node_base {
  typedef _Rb_tree_node<_Val>* _Link_type;
  _Val _M_value_field;
};
```

可以看出 map 的成员变量为：_M_key_compare、_M_header._M_color、_M_header._M_parent、_M_header._M_left、_M_header._M_right、_M_node_count。
对照构造函数的汇编：

```
(gdb) set print asm-demangle 
(gdb) disassemble main
Dump of assembler code for function main:
   0x0000000000400920 <+0>:     push   %rbp
   0x0000000000400921 <+1>:     mov    %rsp,%rbp
   0x0000000000400924 <+4>:     push   %rbx
   0x0000000000400925 <+5>:     sub    $0x48,%rsp
   0x0000000000400929 <+9>:     lea    -0x50(%rbp),%rax
   0x000000000040092d <+13>:    mov    %rax,%rdi
   0x0000000000400930 <+16>:    callq  0x4009f8 <std::map<int, int, std::less<int>, std::allocator<std::pair<int const, int> > >::map()>
```

```
(gdb) disassemble 0x4009f8
Dump of assembler code for function _ZNSt3mapIiiSt4lessIiESaISt4pairIKiiEEEC2Ev:
   0x00000000004009f8 <+0>:     push   %rbp
   0x00000000004009f9 <+1>:     mov    %rsp,%rbp
   0x00000000004009fc <+4>:     sub    $0x10,%rsp
   0x0000000000400a00 <+8>:     mov    %rdi,-0x8(%rbp)
   0x0000000000400a04 <+12>:    mov    -0x8(%rbp),%rax
   0x0000000000400a08 <+16>:    mov    %rax,%rdi
   0x0000000000400a0b <+19>:    callq  0x400b9e <std::_Rb_tree<int, std::pair<int const, int>, std::_Select1st<std::pair<int const, int> >, std::less<int>, std::allocator<std::pair<int const, int> > >::_Rb_tree()>
```

```
(gdb) disassemble 0x400b9e
Dump of assembler code for function _ZNSt8_Rb_treeIiSt4pairIKiiESt10_Select1stIS2_ESt4lessIiESaIS2_EEC2Ev:
   0x0000000000400b9e <+0>:     push   %rbp
   0x0000000000400b9f <+1>:     mov    %rsp,%rbp
   0x0000000000400ba2 <+4>:     sub    $0x10,%rsp
   0x0000000000400ba6 <+8>:     mov    %rdi,-0x8(%rbp)
   0x0000000000400baa <+12>:    mov    -0x8(%rbp),%rax
   0x0000000000400bae <+16>:    mov    %rax,%rdi
   0x0000000000400bb1 <+19>:    callq  0x400d72 <std::_Rb_tree<int, std::pair<int const, int>, std::_Select1st<std::pair<int const, int> >, std::less<int>, std::allocator<std::pair<int const, int> > >::_Rb_tree_impl<std::less<int>, false>::_Rb_tree_impl()>
```

```
(gdb) disassemble 0x400d72
Dump of assembler code for function _ZNSt8_Rb_treeIiSt4pairIKiiESt10_Select1stIS2_ESt4lessIiESaIS2_EE13_Rb_tree_implIS6_Lb0EEC2Ev:
   0x0000000000400d72 <+0>:     push   %rbp
   0x0000000000400d73 <+1>:     mov    %rsp,%rbp
   0x0000000000400d76 <+4>:     sub    $0x10,%rsp
   0x0000000000400d7a <+8>:     mov    %rdi,-0x8(%rbp)
   0x0000000000400d7e <+12>:    mov    -0x8(%rbp),%rax
   0x0000000000400d82 <+16>:    mov    %rax,%rdi
   0x0000000000400d85 <+19>:    callq  0x400f8c <std::allocator<std::_Rb_tree_node<std::pair<int const, int> > >::allocator()>
   0x0000000000400d8a <+24>:    mov    -0x8(%rbp),%rax
   0x0000000000400d8e <+28>:    movl   $0x0,0x8(%rax)    // _M_header._M_color 
   0x0000000000400d95 <+35>:    mov    -0x8(%rbp),%rax
   0x0000000000400d99 <+39>:    movq   $0x0,0x10(%rax)   // _M_header._M_parent 
   0x0000000000400da1 <+47>:    mov    -0x8(%rbp),%rax
   0x0000000000400da5 <+51>:    movq   $0x0,0x18(%rax)   // _M_header._M_left 
   0x0000000000400dad <+59>:    mov    -0x8(%rbp),%rax
   0x0000000000400db1 <+63>:    movq   $0x0,0x20(%rax)   // _M_header._M_right 
   0x0000000000400db9 <+71>:    mov    -0x8(%rbp),%rax
   0x0000000000400dbd <+75>:    movq   $0x0,0x28(%rax)   // _M_node_count(0)
   0x0000000000400dc5 <+83>:    mov    -0x8(%rbp),%rax
   0x0000000000400dc9 <+87>:    mov    %rax,%rdi
   0x0000000000400dcc <+90>:    callq  0x400fa6 <std::_Rb_tree<int, std::pair<int const, int>, std::_Select1st<std::pair<int const, int> >, std::less<int>, std::allocator<std::pair<int const, int> > >::_Rb_tree_impl<std::less<int>, false>::_M_initialize()>
```

```
(gdb) disassemble 0x400fa6
Dump of assembler code for function _ZNSt8_Rb_treeIiSt4pairIKiiESt10_Select1stIS2_ESt4lessIiESaIS2_EE13_Rb_tree_implIS6_Lb0EE13_M_initializeEv:
   0x0000000000400fa6 <+0>:     push   %rbp
   0x0000000000400fa7 <+1>:     mov    %rsp,%rbp
   0x0000000000400faa <+4>:     mov    %rdi,-0x8(%rbp)
   0x0000000000400fae <+8>:     mov    -0x8(%rbp),%rax
   0x0000000000400fb2 <+12>:    movl   $0x0,0x8(%rax)    // this->_M_header._M_color = _S_red;
   0x0000000000400fb9 <+19>:    mov    -0x8(%rbp),%rax   
   0x0000000000400fbd <+23>:    movq   $0x0,0x10(%rax)   // this->_M_header._M_parent = 0;
   0x0000000000400fc5 <+31>:    mov    -0x8(%rbp),%rax   
   0x0000000000400fc9 <+35>:    lea    0x8(%rax),%rdx   
   0x0000000000400fcd <+39>:    mov    -0x8(%rbp),%rax   
   0x0000000000400fd1 <+43>:    mov    %rdx,0x18(%rax)   // this->_M_header._M_left = &this->_M_header;
   0x0000000000400fd5 <+47>:    mov    -0x8(%rbp),%rax
   0x0000000000400fd9 <+51>:    lea    0x8(%rax),%rdx   
   0x0000000000400fdd <+55>:    mov    -0x8(%rbp),%rax
   0x0000000000400fe1 <+59>:    mov    %rdx,0x20(%rax)   // this->_M_header._M_right = &this->_M_header;
   0x0000000000400fe5 <+63>:    pop    %rbp
   0x0000000000400fe6 <+64>:    retq   
End of assembler dump.
```

在构造函数调用处打断点：

```
(gdb) tbreak *0x0000000000400930
Temporary breakpoint 1 at 0x400930
(gdb) r
(gdb) x /12wx 0x7fffffffe420
0x7fffffffe420: 0x00000001      0x00007fff      0xf7203890      0x00007fff
0x7fffffffe430: 0x00000001      0x00000000      0x00401bbd      0x00000000
0x7fffffffe440: 0x00000000      0x00000000      0x00000000      0x00000000
(gdb) ni
0x0000000000400935 in main ()
(gdb) x /12wx $rbp-0x50
0x7fffffffe420: 0x00000001      0x00007fff      0x00000000      0x00007fff
0x7fffffffe430: 0x00000000      0x00000000      0xffffe428      0x00007fff
0x7fffffffe440: 0xffffe428      0x00007fff      0x00000000      0x00000000
```

可见初始化后成员变量的值为：

|_M_key_compare      |  0x00007fff00000001  |
|_M_header._M_color  |  0x00007fff00000000  |
|_M_header._M_parent |  0x0000000000000000  |
|_M_header._M_left   |  0x00007fffffffe428  |  
|_M_header._M_right  |  0x00007fffffffe428  |
|_M_node_count       |  0x0000000000000000  |

其中 _M_left 和 _M_right 都指向 &_M_header。


看看插入第一个数据的汇编代码：

```
0x0000000000400935 <+21>:    movl   $0x5,-0x1c(%rbp)
0x000000000040093c <+28>:    lea    -0x1c(%rbp),%rdx
0x0000000000400940 <+32>:    lea    -0x50(%rbp),%rax
0x0000000000400944 <+36>:    mov    %rdx,%rsi
0x0000000000400947 <+39>:    mov    %rax,%rdi
0x000000000040094a <+42>:    callq  0x400a86 <std::map<int, int, std::less<int>, std::allocator<std::pair<int const, int> > >::operator[](int const&)>
0x000000000040094f <+47>:    movl   $0x6,(%rax)
```

调用处打断点：

```
(gdb) tbreak *0x0000000000400955
Temporary breakpoint 3 at 0x400955
(gdb) c
Continuing.

Temporary breakpoint 3, 0x0000000000400955 in main ()
(gdb) x /12wx $rbp-0x50
0x7fffffffe420: 0x00000001      0x00007fff      0x00000000      0x00007fff
0x7fffffffe430: 0x00604010      0x00000000      0x00604010      0x00000000
0x7fffffffe440: 0x00604010      0x00000000      0x00000001      0x00000000
(gdb) x /10wx 0x00604010
0x604010:       0x00000001      0x00000000      0xffffe428      0x00007fff
0x604020:       0x00000000      0x00000000      0x00000000      0x00000000
0x604030:       0x00000005      0x00000006
```

head节点：

|_M_key_compare      |  0x00007fff00000001  |
|_M_header._M_color  |  0x00007fff00000000  |
|_M_header._M_parent |  0x0000000000604010  |
|_M_header._M_left   |  0x0000000000604010  |  
|_M_header._M_right  |  0x0000000000604010  |
|_M_node_count       |  0x0000000x00000001  |

第一个节点（0x00604010）：

|_M_color              |  0x00000x0000000001  |
|_M_parent             |  0x00007fffffffe428  |
|_M_left               |  0x0000000000000000  |  
|_M_right              |  0x0000000000000000  |
|_M_value_field.first  |  0x00000005          |
|_M_value_field.second |  0x00000006          |

接着看看插入第二个元素之后：

```
(gdb) tbreak *0x0000000000400975
Temporary breakpoint 4 at 0x400975
(gdb) c
Continuing.

Temporary breakpoint 4, 0x0000000000400975 in main ()
(gdb) x /12wx $rbp-0x50
0x7fffffffe420: 0x00000001      0x00007fff      0x00000000      0x00007fff
0x7fffffffe430: 0x00604010      0x00000000      0x00604010      0x00000000
0x7fffffffe440: 0x00604040      0x00000000      0x00000002      0x00000000
(gdb)  x /10wx 0x00604010
0x604010:       0x00000001      0x00000000      0xffffe428      0x00007fff
0x604020:       0x00000000      0x00000000      0x00604040      0x00000000
0x604030:       0x00000005      0x00000006
(gdb)  x /10wx 0x00604040
0x604040:       0x00000000      0x00000000      0x00604010      0x00000000
0x604050:       0x00000000      0x00000000      0x00000000      0x00000000
0x604060:       0x00000008      0x00000014
```

head节点：

|_M_key_compare      |  0x00007fff00000001                             |
|_M_header._M_color  |  0x00007fff00000000                             |
|_M_header._M_parent |  0x0000000000604010                             |
|_M_header._M_left   |  0x0000000000604010                             |  
|_M_header._M_right  |  0x0000000000604040  // 变成 0x0000000000604040 |
|_M_node_count       |  0x0000000x00000001                             |


第一个节点（0x00604010）：

|_M_color              |  0x00000x0000000001                            |
|_M_parent             |  0x00007fffffffe428                            |
|_M_left               |  0x0000000000000000                            |  
|_M_right              |  0x0000000000604040  // 变成 0x0000000000604040 | 
|_M_value_field.first  |  0x00000005                                    |
|_M_value_field.second |  0x00000006                                    |


第二个节点（0x00604040）：


|_M_color              |  0x00000x0000000001  |
|_M_parent             |  0x0000000000604010  |
|_M_left               |  0x0000000000000000  |  
|_M_right              |  0x0000000000000000  |
|_M_value_field.first  |  0x00000008          |
|_M_value_field.second |  0x00000014          |

最后看看插入第三个元素之后：

```
(gdb) tbreak *0x0000000000400995
Temporary breakpoint 5 at 0x400995

(gdb) c
Continuing.

Temporary breakpoint 5, 0x0000000000400995 in main ()
(gdb) x /12wx $rbp-0x50
0x7fffffffe420: 0x00000001      0x00007fff      0x00000000      0x00007fff
0x7fffffffe430: 0x00604010      0x00000000      0x00604070      0x00000000
0x7fffffffe440: 0x00604040      0x00000000      0x00000003      0x00000000
(gdb) x /10wx 0x00604010
0x604010:       0x00000001      0x00000000      0xffffe428      0x00007fff
0x604020:       0x00604070      0x00000000      0x00604040      0x00000000
0x604030:       0x00000005      0x00000006
(gdb) x /10wx 0x00604040
0x604040:       0x00000000      0x00000000      0x00604010      0x00000000
0x604050:       0x00000000      0x00000000      0x00000000      0x00000000
0x604060:       0x00000008      0x00000014
(gdb) x /10wx 0x00604070
0x604070:       0x00000000      0x00000000      0x00604010      0x00000000
0x604080:       0x00000000      0x00000000      0x00000000      0x00000000
0x604090:       0x00000002      0x00000050
```

关系图如下：

<img src="/assets/images/coredump-note-7/illustration-1.png" width="800" />


## coredump 分析
```
(gdb) set print asm-demangle
(gdb) bt
#0  0x0000000000000000 in ?? ()
#1  0x0000000000400ea0 in main ()
(gdb) i r 
rax            0x0      0
rbx            0x0      0
rcx            0x7fff070007b1   140733310830513
rdx            0xa      10
rsi            0x0      0
rdi            0x0      0
rbp            0x7fff06ffe820   0x7fff06ffe820
rsp            0x7fff06ffe788   0x7fff06ffe788
r8             0x7f1790ee2060   139739192500320
r9             0xa      10
r10            0x0      0
r11            0x0      0
r12            0x0      0
r13            0x7fff06ffe900   140733310822656
r14            0x0      0
r15            0x0      0
rip            0x0      0x0
eflags         0x10206  [ PF IF RF ]
cs             0x33     51
ss             0x2b     43
ds             0x0      0
es             0x0      0
fs             0x0      0
gs             0x0      0
```

rip 为 0，根据之前的经验，是调用了空的函数指针。查看 core 附近的汇编：

```
Dump of assembler code for function main:
   0x0000000000400ced <+0>:     push   %rbp
   0x0000000000400cee <+1>:     mov    %rsp,%rbp
   0x0000000000400cf1 <+4>:     push   %r12
   0x0000000000400cf3 <+6>:     push   %rbx
   0x0000000000400cf4 <+7>:     add    $0xffffffffffffff80,%rsp
   0x0000000000400cf8 <+11>:    mov    %edi,-0x84(%rbp)
   0x0000000000400cfe <+17>:    mov    %rsi,-0x90(%rbp)
   0x0000000000400d05 <+24>:    cmpl   $0x3,-0x84(%rbp)
   0x0000000000400d0c <+31>:    jg     0x400d22 <main+53>
   0x0000000000400d0e <+33>:    mov    $0x402310,%edi
   0x0000000000400d13 <+38>:    callq  0x400a80 <puts@plt>
   0x0000000000400d18 <+43>:    mov    $0xffffffff,%ebx
   0x0000000000400d1d <+48>:    jmpq   0x400ec6 <main+473>
   0x0000000000400d22 <+53>:    lea    -0x80(%rbp),%rax
   0x0000000000400d26 <+57>:    mov    %rax,%rdi
   0x0000000000400d29 <+60>:    callq  0x400fae <std::map<std::string, int (*)(int, int), std::less<std::string>, std::allocator<std::pair<std::string const, int (*)(int, int)> > >::map()>
   0x0000000000400d2e <+65>:    lea    -0x41(%rbp),%rax
   0x0000000000400d32 <+69>:    mov    %rax,%rdi
   0x0000000000400d35 <+72>:    callq  0x400b80 <_ZNSaIcEC1Ev@plt>
   0x0000000000400d3a <+77>:    lea    -0x41(%rbp),%rdx
   0x0000000000400d3e <+81>:    lea    -0x50(%rbp),%rax
   0x0000000000400d42 <+85>:    mov    $0x402326,%esi
   0x0000000000400d47 <+90>:    mov    %rax,%rdi
   0x0000000000400d4a <+93>:    callq  0x400b00 <_ZNSsC1EPKcRKSaIcE@plt>
   0x0000000000400d4f <+98>:    lea    -0x50(%rbp),%rdx
   0x0000000000400d53 <+102>:   lea    -0x80(%rbp),%rax
   0x0000000000400d57 <+106>:   mov    %rdx,%rsi
   0x0000000000400d5a <+109>:   mov    %rax,%rdi
   0x0000000000400d5d <+112>:   callq  0x401056 <std::map<std::string, int (*)(int, int), std::less<std::string>, std::allocator<std::pair<std::string const, int (*)(int, int)> > >::operator[](std::string const&)>
   0x0000000000400d62 <+117>:   movq   $0x400cb0,(%rax)
   0x0000000000400d69 <+124>:   lea    -0x50(%rbp),%rax
   0x0000000000400d6d <+128>:   mov    %rax,%rdi
   0x0000000000400d70 <+131>:   callq  0x400af0 <_ZNSsD1Ev@plt>
   0x0000000000400d75 <+136>:   lea    -0x41(%rbp),%rax
   0x0000000000400d79 <+140>:   mov    %rax,%rdi
   0x0000000000400d7c <+143>:   callq  0x400b30 <_ZNSaIcED1Ev@plt>
   0x0000000000400d81 <+148>:   lea    -0x31(%rbp),%rax
   0x0000000000400d85 <+152>:   mov    %rax,%rdi
   0x0000000000400d88 <+155>:   callq  0x400b80 <_ZNSaIcEC1Ev@plt>
   0x0000000000400d8d <+160>:   lea    -0x31(%rbp),%rdx
   0x0000000000400d91 <+164>:   lea    -0x40(%rbp),%rax
   0x0000000000400d95 <+168>:   mov    $0x402328,%esi
   0x0000000000400d9a <+173>:   mov    %rax,%rdi
   0x0000000000400d9d <+176>:   callq  0x400b00 <_ZNSsC1EPKcRKSaIcE@plt>
   0x0000000000400da2 <+181>:   lea    -0x40(%rbp),%rdx
   0x0000000000400da6 <+185>:   lea    -0x80(%rbp),%rax
   0x0000000000400daa <+189>:   mov    %rdx,%rsi
   0x0000000000400dad <+192>:   mov    %rax,%rdi
   0x0000000000400db0 <+195>:   callq  0x401056 <std::map<std::string, int (*)(int, int), std::less<std::string>, std::allocator<std::pair<std::string const, int (*)(int, int)> > >::operator[](std::string const&)>
   0x0000000000400db5 <+200>:   movq   $0x400cc4,(%rax)
   0x0000000000400dbc <+207>:   lea    -0x40(%rbp),%rax
   0x0000000000400dc0 <+211>:   mov    %rax,%rdi
   0x0000000000400dc3 <+214>:   callq  0x400af0 <_ZNSsD1Ev@plt>
   0x0000000000400dc8 <+219>:   lea    -0x31(%rbp),%rax
   0x0000000000400dcc <+223>:   mov    %rax,%rdi
   0x0000000000400dcf <+226>:   callq  0x400b30 <_ZNSaIcED1Ev@plt>
   0x0000000000400dd4 <+231>:   lea    -0x21(%rbp),%rax
   0x0000000000400dd8 <+235>:   mov    %rax,%rdi
   0x0000000000400ddb <+238>:   callq  0x400b80 <_ZNSaIcEC1Ev@plt>
   0x0000000000400de0 <+243>:   lea    -0x21(%rbp),%rdx
   0x0000000000400de4 <+247>:   lea    -0x30(%rbp),%rax
   0x0000000000400de8 <+251>:   mov    $0x40232a,%esi
   0x0000000000400ded <+256>:   mov    %rax,%rdi
   0x0000000000400df0 <+259>:   callq  0x400b00 <_ZNSsC1EPKcRKSaIcE@plt>
   0x0000000000400df5 <+264>:   lea    -0x30(%rbp),%rdx
   0x0000000000400df9 <+268>:   lea    -0x80(%rbp),%rax
   0x0000000000400dfd <+272>:   mov    %rdx,%rsi
   0x0000000000400e00 <+275>:   mov    %rax,%rdi
   0x0000000000400e03 <+278>:   callq  0x401056 <std::map<std::string, int (*)(int, int), std::less<std::string>, std::allocator<std::pair<std::string const, int (*)(int, int)> > >::operator[](std::string const&)>
   0x0000000000400e08 <+283>:   movq   $0x400cda,(%rax)
   0x0000000000400e0f <+290>:   lea    -0x30(%rbp),%rax
   0x0000000000400e13 <+294>:   mov    %rax,%rdi
   0x0000000000400e16 <+297>:   callq  0x400af0 <_ZNSsD1Ev@plt>
   0x0000000000400e1b <+302>:   lea    -0x21(%rbp),%rax
   0x0000000000400e1f <+306>:   mov    %rax,%rdi
   0x0000000000400e22 <+309>:   callq  0x400b30 <_ZNSaIcED1Ev@plt>
   0x0000000000400e27 <+314>:   lea    -0x11(%rbp),%rax
   0x0000000000400e2b <+318>:   mov    %rax,%rdi
   0x0000000000400e2e <+321>:   callq  0x400b80 <_ZNSaIcEC1Ev@plt>
   0x0000000000400e33 <+326>:   mov    -0x90(%rbp),%rax
   0x0000000000400e3a <+333>:   add    $0x10,%rax
   0x0000000000400e3e <+337>:   mov    (%rax),%rcx
   0x0000000000400e41 <+340>:   lea    -0x11(%rbp),%rdx
   0x0000000000400e45 <+344>:   lea    -0x20(%rbp),%rax
   0x0000000000400e49 <+348>:   mov    %rcx,%rsi
   0x0000000000400e4c <+351>:   mov    %rax,%rdi
   0x0000000000400e4f <+354>:   callq  0x400b00 <_ZNSsC1EPKcRKSaIcE@plt>
   0x0000000000400e54 <+359>:   lea    -0x20(%rbp),%rdx
   0x0000000000400e58 <+363>:   lea    -0x80(%rbp),%rax
   0x0000000000400e5c <+367>:   mov    %rdx,%rsi
   0x0000000000400e5f <+370>:   mov    %rax,%rdi
   0x0000000000400e62 <+373>:   callq  0x401056 <std::map<std::string, int (*)(int, int), std::less<std::string>, std::allocator<std::pair<std::string const, int (*)(int, int)> > >::operator[](std::string const&)>
   0x0000000000400e67 <+378>:   mov    (%rax),%rbx
   0x0000000000400e6a <+381>:   mov    -0x90(%rbp),%rax
   0x0000000000400e71 <+388>:   add    $0x18,%rax
   0x0000000000400e75 <+392>:   mov    (%rax),%rax
   0x0000000000400e78 <+395>:   mov    %rax,%rdi
   0x0000000000400e7b <+398>:   callq  0x400b10 <atoi@plt>
   0x0000000000400e80 <+403>:   mov    %eax,%r12d
   0x0000000000400e83 <+406>:   mov    -0x90(%rbp),%rax
   0x0000000000400e8a <+413>:   add    $0x8,%rax
   0x0000000000400e8e <+417>:   mov    (%rax),%rax
   0x0000000000400e91 <+420>:   mov    %rax,%rdi
   0x0000000000400e94 <+423>:   callq  0x400b10 <atoi@plt>
   0x0000000000400e99 <+428>:   mov    %r12d,%esi
   0x0000000000400e9c <+431>:   mov    %eax,%edi
   0x0000000000400e9e <+433>:   callq  *%rbx
=> 0x0000000000400ea0 <+435>:   mov    %eax,%ebx
   ...
(gdb) i r rbx
rbx            0x0      0
(gdb) 
```

确定是 rbx 寄存器为空地址引起的，rbx 存的是 <+373> 调用 operator[] 的返回值。分析下 operator[] 传的是哪个 key 值：

```
(gdb) x /2wx $rbp-0x20
0x7ffdcb892580: 0x01fce178      0x00000000
(gdb) x /s 0x01fce178
0x1fce178:      "b"
```

再根据 <+112> <+195> <+278> 可以看出 map 中不存在 key 为 "b" 的元素（分别插入了 "+"，"-"，"*"）：

```
(gdb) x /2wx $rbp-0x50
0x7ffdcb892550: 0x01fce028      0x00000000
(gdb) x /s 0x01fce028
0x1fce028:      "+"
(gdb) x /2wx $rbp-0x40
0x7ffdcb892560: 0x01fce098      0x00000000
(gdb) x /s 0x01fce098
0x1fce098:      "-"
(gdb) x /2wx $rbp-0x30
0x7ffdcb892570: 0x01fce108      0x00000000
(gdb) x /s 0x01fce108
0x1fce108:      "*"
```

对比源码：

```c++
#include <map>
#include <string>
#include <stdio.h>
#include <stdlib.h>

typedef int (*oper)(int a, int b );

int add(int a, int b) {
    return a + b;
}

int sub(int a, int b) {
    return a - b;
}

int mul(int a, int b) {
    return a * b;
}

int main(int argc, char* argv[]) {
    if (argc < 4) {
        printf( "parameter less than 4\n" );
        return -1;
    }

    std::map< std::string, oper> operMap;
    operMap["+"] = &add;
    operMap["-"] = &sub;
    operMap["*"] = &mul;

    return operMap[argv[2]](atoi(argv[1]), atoi(argv[3]));
}
```

执行 ```./test a b c``` 导致 core。