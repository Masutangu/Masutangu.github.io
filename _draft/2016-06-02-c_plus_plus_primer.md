---
layout: post
date: 2016-06-02T19:27:03+08:00
draft: true
title: 《 C++ Primer 》读书笔记
---

### plain char, signed char , and unsigned char

char相当于 signed char 或者 unsigned char ，但是这取决于编译器的实现。
而unsigned char和signed char的区别主要在于高位扩展，请看下面代码：

    #include <stdio.h>

    void convert(unsigned char v)
    {
        char c = v;
        unsigned char uc = v;
        unsigned int ui = c, uj = uc;
        int i = c, j = uc;
        
        printf("%%c: [%c], [%c]\n", c, uc);
        printf("%%X: [%X], [%X]\n", c, uc);
        printf("%%u: [%u], [%u]\n", ui, uj);
        printf("%%d: [%d], [%d]\n", i, j);    
    }

    int main()
    {
        convert(0x80);
        convert(0x7F);
        return 0;
    }

输出结果：
    %c: [?], [?]
    %X: [FFFFFF80], [80]
    %u: [4294967168], [128]
    %d: [-128], [128]
    %c: [], []
    %X: [7F], [7F]
    %u: [127], [127]
    %d: [127], [127]
    
如果希望写出的代码在不同环境下有同样的效果，需要显式指明使用 signed char 或 unsigned char 的其中一种。

### const 变量

> There are three exceptions to the rule that headers should not contain definitions: classes, const objects whose value is known at compile time, and inline functions

默认情况下，const对象是定义该变量的文件的局部变量。
    //common.h
    #ifndef _COMMON_H_
    #define _COMMON_H_
    #include <stdlib.h>     /* srand, rand */
   
    const int a = rand()%100;

    #endif
    
    
    //test.h
    #ifndef _TEST_H_
    #define _TEST_H_

    extern int b;

    #endif
    
    
    //test.cpp
    #include "test.h"
    #include <stdlib.h>     /* srand, rand */

    int b = rand()%100;
    
    
    //print.h
    #ifndef _PRINT_H_
    #define _PRINT_H_

    void print_a();

    #endif


    //print.cpp
    #include "common.h"
    #include <iostream>

    void print_a()
    {
        std::cout << "call print_a()" << std::endl;
        std::cout << "a=" << a << std::endl;
    }
    
    
    //main.cpp
    #include <iostream>
    #include "common.h"
    #include "print.h"

    int main()
    {
        std::cout << "a:" << a << std::endl;
        print_a();
        return 0;
    }
    
上面这个例子，打印出来变量a的值不相同。证明了const变量a在main.cpp文件和print.cpp文件中是不同的局部变量。因此const变量可以定义在头文件中，不会有重复定义的问题。
验证一下，使用readelf -s main.o查看main.o文件的符号表，可以看到变量a为局部变量：
<img src="/assets/images/c_plus_plus_primer/illustration-1.png" width="800" />
同理查看print.o文件的符号表，可以看到变量a同样是局部变量：
<img src="/assets/images/c_plus_plus_primer/illustration-2.png" width="800" />

因此头文件可以定义const变量，不会引起重复定义的问题：
> There are three exceptions to the rule that headers should not contain definitions: classes, const objects whose value is known at compile time, and inline functions

> Nonconst variables are extern by default. To make a const variable accessible to other files we must explicitly specify that it is extern .
非const变量默认是extern的，在该源文件以外的源文件可以添加其声明，并使用该变量。const变量默认是非extern的，所以只能在本文件中使用，不能在该文件以外的源文件中使用，如果想在别的文件中使用，必须显式定义为extern。
    //common.h
    #ifndef _COMMON_H_
    #define _COMMON_H_
    #include <stdlib.h>     /* srand, rand */
   
    extern int b;
    extern const int c;
    
    #endif
    
    
    //test.cpp
    #include <stdlib.h>     /* srand, rand */

    int b = rand()%100;       //默认是extern
    extern const int c = 100; //必须要加extern声明，否则不能在main.cpp中使用
    
    
    //main.cpp
    #include <iostream>
    #include "common.h"

    int main()
    {
        std::cout << "b:" << b << std::endl;
        std::cout << "c:" << c << std::endl;
        return 0;
    }


### using 指令
一般情况下，避免将 using 指令放置在头文件 (*.h) 中，因为任何包含该标头的文件都会将命名空间中的所有内容引入范围，这将导致非常难以调试的名称隐藏和名称冲突问题。在头文件中，始终使用完全限定名。如果这些名称太长，可以使用命名空间别名将其缩短。

### char* 和 char [] 的区别
[How do char\[\] and char * differ?](https://www.quora.com/How-do-char-and-char-*-differ)
    
    char *p1 = "Hello";
    p1[0] = 'A'; // Error: Segmentation fault
    char chr = 'Z';
    p1 = &chr; // OK: Value of p1 can be changed

p1是左值，指向常量"Hello"的地址。因此p1指向的内容不能变，但可以改变p1指向的地址。

    char p2[] = "Hello";
    p2[0] = 'A'; // Valid
    char chr = 'Z';
    p2 = &chr;  // Error: p2 is not a valid L-Value

p2是常量，指向一块连续的内存。因此p2指向的内容可以变，但p2指向的地址不能变。

    char *p1 = "Hello World";
    while(*p1) {
        cout << *p1++; // Valid, p1 is a L-Value
    }
    char p2[] = "Hello World";
    while (*p2) {
        cout << *p2++; // Error, p2 is not a L-Value
    }

同理，p2是常量，始终指向&p2\[0\]，因此不能自增。

## c_str() 
As long as the string isn't destroyed or modified, using c_str() is OK. If the string is modified using a previously returned c_str() is implementation defined.
    
## static_cast，dynamic_cast，const_cast，reinterpret_cast
* static_cast: 可以实现C++中内置基本数据类型之间的相互转换，但只能在有相互联系的类型中进行相互转换。
* const_cast: 可以使一个本来不是const类型的数据转换成const类型的，或者把const属性去掉。
* reinterpret_cast: 可以转化任何内置的数据类型为其他任何的数据类型，也可以转化任何指针类型为其他的类型。甚至可以转化内置的数据类型为指针，无须考虑类型安全或者常量的情形。
* dynamic_cast: 
    * 其他三种都是编译时完成的，dynamic_cast是运行时处理的，运行时要进行类型检查。
    * 不能用于内置的基本数据类型的强制转换。
    * 转换如果成功的话返回的是指向类的指针或引用，转换失败的话则会返回NULL。
    * 用dynamic_cast进行转换的，基类中一定要有虚函数，否则编译不通过。需要检测有虚函数的原因：这是由于运行时类型检查需要运行时类型信息，而这个信息存储在类的虚函数表中，只有定义了虚函数的类才有虚函数表。
    * 在类的转换时，在类层次间进行上行转换时，dynamic_cast和static_cast的 效果是一样的。在进行下行转换时，dynamic_cast具有类型检查的功能，比 static_cast 更安全。向下转换即将父类指针转化子类指针。向下转换的成功与否与将要转换的类型有关，即要转换的指针指向的对象的实际类型与转换以后的对象类型一定要相同，否则转换失败。


## 初始化
内置类型：
> Variables defined outside any function body are initialized to zero. Variables of built-in type defined inside the body of a function are uninitialized .

静态数组：
> If we do not supply element initializers, then the elements are initialized in the same way that variables are initialized.
* Elements of an array of built-in type defined outside the body of a function are initialized to zero.
* Elements of an array of built-in type defined inside the body of a function are uninitialized.
* Regardless of where the array is defined, if it holds elements of a class type, then the elements are initialized by the default constructor for that class if it has one. If the class does not have a default constructor, then the elements must be explicitly initialized.

动态数组：
> When we allocate an array of objects of a class type, then that type's default constructor (Section 2.3.4 , p. 50 ) is used to initialize each element. If the array holds elements of built-in type, then the elements are uninitialized.
> Alternatively, we can value-initialize (Section 3.3.1 , p. 92 ) the elements by following the array size by an empty pair of parentheses.

类合成构造函数：
> The compiler-created default constructor is known as a synthesized default constructor . It initializes each member using the same rules as are applied for variable initializations (Section 2.3.4 , p. 50 ). Members that are of class type, such as isbn , are initialized by using the default constructor of the member's own class. The initial value of members of built-in type depend on how the object is defined. If the object is defined at global scope (outside any function) or is a local static object, then these members will be initialized to 0. If the object is defined at local scope, these members are uninitialized.

当类只包括类成员时，合成构造函数才有作用。
> The synthesized default constructor often suffices for classes that contain only members of class type. Classes with members of built- in or compound type should usually define their own default constructors to initialize those members.

## const 函数参数
const非引用参数会被编译器忽略。
> What may be surprising, is that although the parameter is a const inside the function, the compiler otherwise treats the definition of fcn as if we had defined the parameter as a plain int
> When the parameter is copied, whether the parameter is const is irrelevantthe function executes on a copy. Nothing the function does can change the argument. As a result, we can pass a const object to either a const or nonconst parameter. The two parameters are indistinguishable.


当函数使用引用参数且不修改参数时，应该使用const引用，这样更加灵活。
> Parameters that do not change the value of the corresponding argument should be const references. Defining such parameters as nonconst references needlessly restricts the usefulness of a function.
> Reference parameters that are not changed should be references to const . Plain, nonconst reference parameters are less flexible. Such parameters may not be initialized by const objects, or by arguments that are literals or expressions that yield rvalues.

    string::size_type find_char(string &s, char c)
    {
        string::size_type i = 0;
        while (i != s.size() && s[i] != c)
        ++i; // not found, look at next character return i;
    }
    
    if (find_char("Hello World", 'o')) // 报错，非const引用不会做隐式转换
    
可以参照const和非const引用初始化的不同（const引用允许类型隐式转换）。

## 作用域
局部变量会覆盖外层的函数名：

> A name declared local to a function hides the same name declared in the global scope. The same is true for function names as for variable names.

同理，局部函数会覆盖外层的函数名，所以重载函数需要在同一层作用域：

> Normal scoping rules apply to names of overloaded functions. If we declare a function locally, that function hides rather than overloads the same function declared in an outer scope. As a consequence, declarations for every version of an overloaded function must appear in the same scope.


## 容器的初始化方式

* Intializing a Container as a Copy of Another Container
When we copy one container into another, the types must match exactly: The container type and element type must be the same.
容器类型和元素类型需要完全相同。

* Initializing as a Copy of a Range of Elements
When we use iterators, there is no requirement that the container types be identical. The element types in the containers can differ as long as they are compatible.
不需要是相同的容器类型，元素类型要求是可以转换的。

## 关联容器的 strict weak ordering
永远让比较函数对相等的值返回false。

> Such a comparison function must always yield false when we compare a key with itself. Moreover, if we compare two keys, they cannot both be "less than" each other, and if k1 is "less than" k2 , which in turn is "less than" k3 , then k1 must be "less than" k3 .

## 复制构造函数不应该声明为explicit
摘自stackoverflow：
> The explicit copy constructor means that the copy constructor will not be called implicitly, which is what happens in the expression:

> ```CustomString s = CustomString("test");```
> This expression literally means: create a temporary CustomString using the constructor that takes a const char*. Implicitly call the copy constructor of CustomString to copy from that temporary into s.

> Now, if the code was correct (i.e. if the copy constructor was not explicit), the compiler would avoid the creation of the temporary and elide the copy by constructing s directly with the string literal. But the compiler must still check that the construction can be done and fails there.

> You can call the copy constructor explicitly:

> ```CustomString s( CustomString("test") );```