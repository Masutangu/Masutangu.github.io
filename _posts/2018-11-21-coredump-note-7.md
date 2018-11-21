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