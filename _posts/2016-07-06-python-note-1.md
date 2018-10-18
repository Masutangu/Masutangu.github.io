---
layout: post
date: 2016-07-06T18:56:31+08:00
title: Python 笔记
tags: 
  - 编程语言
  - Python
---

这篇文章整理了python相关的资料，包括性能优化、常见错误和高级用法。

*注：本文内容整理自网上博客，《Python Cookbook》等，非原创。*

## 性能优化
### 1. 字典和列表

Python 字典中使用了 hash table，因此查找操作的复杂度为 O(1)，而 list 实际是个数组，在 list 中，查找需要遍历整个 list，其复杂度为 O(n)，因此对成员的查找访问等操作字典要比 list 更快。

```python
from time import time
t = time()
list = ['a','b','is','python','jason','hello','hill','with','phone','test',
'dfdf','apple','pddf','ind','basic','none','baecr','var','bana','dd','wrd']
#list = dict.fromkeys(list,True)
print list
filter = []
for i in range (1000000):
    for find in ['is','hat','new','list','old','.']:
        if find not in list:
            filter.append(find)
print "total run time:"
print time()-t
```
将list转化为dict后速度提升了将近一半。

### 2. 集合和列表
set 的 union， intersection，difference 操作要比 list 的迭代要快。因此如果涉及到求 list 交集，并集或者差的问题可以转换为 set 来操作。

```python
# 使用list：
from time import time
t = time()
lista=[1,2,3,4,5,6,7,8,9,13,34,53,42,44]
listb=[2,4,6,9,23]
intersection=[]
for i in range (1000000):
for a in lista:
    for b in listb:
        if a == b:
            intersection.append(a)

print "total run time:"
print time()-t

# 使用set：
from time import time
t = time()
lista=[1,2,3,4,5,6,7,8,9,13,34,53,42,44]
listb=[2,4,6,9,23]
intersection=[]
for i in range (1000000):
    list(set(lista)&set(listb))
print "total run time:"
print time()-t
```

### 3. 字符串的优化
* 在字符串连接的使用尽量使用 join() 而不是 +。
* 当对字符串可以使用正则表达式或者内置函数来处理的时候，选择内置函数。如 str.isalpha()，str.isdigit()，str.startswith((‘x’, ‘yz’))，str.endswith((‘x’, ‘yz’))
* 对字符进行格式化比直接串联读取要快，因此要

    使用：```out = "<html>%s%s%s%s</html>" % (head, prologue, query, tail)```

    避免：```out = "<html>" + head + prologue + query + tail + "</html>"```

### 4. 使用列表解析和生成器表达式
列表解析要比在循环中重新构建一个新的 list 更为高效，因此我们可以利用这一特性来提高运行的效率。

```python
from time import time
t = time()
list = ['a','b','is','python','jason','hello','hill','with','phone','test',
'dfdf','apple','pddf','ind','basic','none','baecr','var','bana','dd','wrd']
total=[]
for i in range (1000000):
for w in list:
    total.append(w)
print "total run time:"
print time()-t

# 使用列表解析：
for i in range (1000000):
    a = [w for w in list]
```
在上述例子上中代码 ```a = [w for w in list]``` 修改为 ```a = (w for w in list)```，运行时间将进一步减少。

### 5. 其他优化技巧
* 如果需要交换两个变量的值使用 a,b=b,a 而不是借助中间变量 t=a;a=b;b=t；
* 在循环的时候使用 xrange 而不是 range；使用 xrange 可以节省大量的系统内存，因为 xrange() 在序列中每次调用只产生一个整数元素。而 range() 將直接返回完整的元素列表，用于循环时会有不必要的开销。在 python3 中 xrange 不再存在，里面 range 提供一个可以遍历任意长度的范围的 iterator。
* 使用局部变量，避免”global” 关键字。python 访问局部变量会比全局变量要快得多，因 此可以利用这一特性提升性能。
* if done is not None 比语句 if done != None 更快，读者可以自行验证；
* 在耗时较多的循环中，可以把函数的调用改为内联的方式；
* 使用级联比较 “x < y < z” 而不是 “x < y and y < z”；
* while 1 要比 while True 更快（当然后者的可读性更好）；
* build in 函数通常较快，add(a,b) 要优于 a+b。

## 常见错误
### 1. range的使用

不恰当的使用range，容易出bug：

```python
for i range(len(alist)):
    print alist[i]
```

正确的做法：

```python
for item in alist:
    print item
```

不恰当使用range的理由：

* 需要在循环中使用索引：

    ```python
    for index, value in enumerate(alist):
        print index, value
    ```
* 需要同时迭代两个循环：

    ```python
    for word, number in zip(words, numbers):
        print word, number
    ```
* 需要迭代序列的一部分：

    ```python
    for word in words[1:]: # 不包括第一个元素
        print word
    ```
**range的正确用法是生成一个数字序列，而不是生成索引：**

    ```python
    # Print foo(x) for 0<=x<5
    for x in range(5):
        print foo(x)
    ```

### 2. 变量泄漏
* 循环

    错误的代码：

    ```python
    for idx, value in enumerate(y):
        if value > max_value:
            break

    processList(y, idx)
    ```
    当y为空，processList将会抛出异常，原因是idx没有定义。

    正确的处理方式：**哨兵模式**，在循环前为idx设置一些特殊的值。

    ```python
    idx ＝ None
    for idx, value in enumerate(y):
        if value > max_value:
            break

    if idex:
        processList(y, idx)
    ```
* 外作用域

    错误的代码：

    ```python
    import sys

    # See the bug in the function declaration?
    def print_file(filenam):
        """Print every line of a file."""
        with open(filename) as input_file:
            for line in input_file:
                print line.strip()

    if __name__ == "__main__":
        filename = sys.argv[1]
        print_file(filename)
    ```
    此时函数定义中的参数被错误的写为filenam，但是程序依然可以运行。因为print_file的外部作用域存在一个filename的变量。

    正确的做法：**外部作用域的全局变量命名要明显**，例如IN_ALL_CAPS。

### 3. 循环的数据结构导致循环

如果在一个对象中发现一个循环，python会输出一个[...]。

```python
mylist ＝ ［'test']
mylist.append(mylist)
#此时会打印['test',[...]]
print mylist
```

### 4. 赋值创建引用

python中赋值语句不会创建对象副本,只会创建引用：

```python
arr = [1, 2, 3, 4]
arr_cp = arr
arr_cp[0] = 100
#print打印出[100, 2, 3, 4]
print arr
```

### 5. 静态识别局部变量

python默认将一个在函数中赋值的变量名视为局部变量，存在于该函数的作用域并当函数运行时才存在。python是在编译def代码时去静态识别局部变量的。

错误的代码：

```python
a ＝ 100

＃你可能想先打印a的值，再对a的值进行修改
def myfunc():
    print a
    a = 200
```

因为在预编译的时候python发现函数中对a有赋值，因此把a当作局部变量。而运行到‘print a’语句的时候，局部变量a尚未赋值，因此会报错。

正确的代码：

```python
a ＝ 100

def myfunc():
    global a
    print a
    a = 200
```
更隐晦的错误代码：

```python
myVar = 1

def myfunc():
    myVar += 1
```
```myVar += 1```其实相当于```myVar = myVar + 1```，python检测到myVar变量有赋值操作，因此将myVar添加到局部命名空间中。当执行到```myVar += 1```时会读取myVar的值，此时该变量尚未有值关联，因此会报错。

### 6. 默认参数

在执行def语句时，默认参数的值只被解析并保存一次。因此在修改可变的默认变量时可能会出现意想不到的效果。

错误的代码：

```python
def saver(x=[]):
    x.append(1)
    print x

saver() # 打印[1]
saver() # 打印[1,1]
saver() # 打印[1,1,1]
```
因为默认参数只被解析并保存一次。因此可变的默认参数在每次函数调用都会保存状态。

正确的代码：

```python
def saver(x=None):
    if x is None: x = []
    x.append(1)
    print x
```

def是python中的可执行语句。默认参数在def的语句环境里被计算。如果你执行了def语句多次，每次它都将会创建一个新的函数对象。

看看stackoverflow的一个例子：

```python
flist = []

for i in xrange(3):
    def func(x): return x * i
    flist.append(func)

for f in flist:
    print f(2) ＃expect 0 2 4 but print 4 4 4
```

我们可以借助默认参数的机制，在执行def时解析默认参数的值：

```python
flist=[]
for i in xrange(3):
    def func(x,i=i): return x*i
    flist.append(func)

for f in flist:
    print f(2)
```

默认参数还可以用来做缓存：

```python
def calculate(a, b, c, memo={}):
    try:
        value = memo[a, b, c] # return already calculated value
    except KeyError:
        value = heavy_calculation(a, b, c)
    memo[a, b, c] = value # update the memo dictionary
    return value
```

牢记：当Python执行一条def语句时， 它会使用已经准备好的东西（包括函数的代码对象和函数的上下文属性），创建了一个新的函数对象。同时，计算了函数的默认参数值。

### 7. 谨慎使用super()

原文[Python's Super is nifty, but you can't use it](https://fuhm.net/super-harmful/)

作者提出两个观点：

* People omit calls to super(...).__init__ if the only superclass is 'object', as, after all, object.__init__ doesn't do anything! However, this is very incorrect. Doing so will cause other classes' __init__ methods to not be called.
* People think they know what arguments their method will get, and what arguments they should pass along to super. This is also incorrect.
先看第二点，比较好理解，代码如下：

```python
class A(object):
    def __init__(self):
        print "A"
        super(A, self).__init__()

class B(object):
    def __init__(self):
        print "B"
        super(B, self).__init__()

class C(A):
    def __init__(self, arg):
        print "C","arg=",arg
        super(C, self).__init__()

class D(B):
    def __init__(self, arg):
        print "D", "arg=",arg
        super(D, self).__init__()

class E(C,D):
    def __init__(self, arg):
        print "E", "arg=",arg
        super(E, self).__init__(arg)

print "MRO:", [x.__name__ for x in E.__mro__]
E(10)
```
看着很正确，执行下报错：

```python
MRO: ['E', 'C', 'A', 'D', 'B', 'object']
E arg= 10
C arg= 10
A

Traceback (most recent call last):
File "C:\Users\mustangmo\Desktop\test1.py", line 27, in <module>
    E(10)
File "C:\Users\mustangmo\Desktop\test1.py", line 24, in __init__
    super(E, self).__init__(arg)
File "C:\Users\mustangmo\Desktop\test1.py", line 14, in __init__
    super(C, self).__init__()
File "C:\Users\mustangmo\Desktop\test1.py", line 4, in __init__
    super(A, self).__init__()
TypeError: __init__() takes exactly 2 arguments (1 given)
```

原因是：MRO: ['E', 'C', 'A', 'D', 'B', 'object']，A的下一个是D，因此super(A, self)方法调用的是D的__init__方法，D的__init__方法需要一个参数，因此报错了。

再看第一点，如果父类是object的话，不调用super().__init__可能会导致问题，例子如下：

```python
class A(object):
    def __init__(self, *args, **kwargs):
        print "A"
        #super(A, self).__init__(*args, **kwargs) 注释掉

class B(object):
    def __init__(self, *args, **kwargs):
        print "B"
        #super(B, self).__init__(*args, **kwargs) 注释掉

class C(A):
    def __init__(self, arg, *args, **kwargs):
        print "C","arg=",arg
        super(C, self).__init__(arg, *args, **kwargs)

class D(B):
    def __init__(self, arg, *args, **kwargs):
        print "D", "arg=",arg
        super(D, self).__init__(arg, *args, **kwargs)

class E(C,D):
    def __init__(self, arg, *args, **kwargs):
        print "E", "arg=",arg
        super(E, self).__init__(arg, *args, **kwargs)

print "MRO:", [x.__name__ for x in E.__mro__]
E(10)
```python
输出结果：

```
MRO: ['E', 'C', 'A', 'D', 'B', 'object']
E arg= 10
C arg= 10
A
```
可以发现D和B都没有输出，也就是说如果没有调用父类为object类的super.__init__()，会导致其他类（在本例中为D和B）的__init__()不执行。按理来说，调用了类E的super.__init__()函数，应该会同时调用E的父类C和D的__init__()函数。但是由于MRO是以super()调用来驱动的，上诉例子中，执行到A时，由于没有调用super的init()函数了，因此整个链路就停了。
总结：

* 一定要调用父类为object的类的super.__init__()函数
* 调用的super()返回不一定是父类，因此super调用最好保持参数一致

另附一篇也是关于super的文章[Python’s super() considered super!](http://rhettinger.wordpress.com/2011/05/26/super-considered-super/)

### 8. string转换为dict

```python
str = ‘{ "key" : null}'
mydict = eval(str)
```
eval 可能会报错，因为 json 的语义跟 Python 的 dict 不完全一样, 如果 json 串里面出现一个 null 就报错了.
因此合适的方法是采用如下写法：

```python
json.loads()
mydict = json.loads(str)
```

## 高级特性

### 1. 闭包

```python
def return_func_that_prints_list(z):
    def f():
        print z
    return f

z = [1, 2]
g = return_func_that_prints_list(z)
g()  # print [1, 2]

z.append(3)
g()  # print [1, 2, 3]
z = [1]
g()  # print [1, 2, 3]
```

【译者】：z.append(3)时，g()内部的引用和z仍然指向一个变量，而z=[1]之后，两者就不再指向一个变量了。

关于闭包，stack overflow http://stackoverflow.com/questions/4020419/closures-in-python

### 2. wraps

**给decorator加上wraps以保留原有函数的名称和docstring：**

```python
from functools import wraps
def my_decorator(f):
    @wraps(f)
    def wrapper(*args, **kwds):
        print 'Calling decorated function'
        return f(*args, **kwds)
    return wrapper


@my_decorator
def example():
    """Docstring"""
    print 'Called example function'
    
example()  # print 'Calling decorated function' 'Called example function'
example.__name__  # print 'example'
example.__doc__  # print 'Docstring'
```

**Without the use of this decorator factory, the name of the example function would have been 'wrapper', and the docstring of the original example() would have been lost.**

### 3. class decorator

**给类的所有函数添加decorator：**

```python
def logged(time_format, name_prefix=""):
    def decorator(func):
        if hasattr(func, '_logged_decorator') and func._logged_decorator:
            return func

        @wraps(func)
        def decorated_func(*args, **kwargs):
            start_time = time.time()
            print "- Running '%s' on %s " % (
                                            name_prefix + func.__name__,
                                            time.strftime(time_format)
                                )
            result = func(*args, **kwargs)
            end_time = time.time()
            print "- Finished '%s', execution time = %0.3fs " % (
                                            name_prefix + func.__name__,
                                            end_time - start_time
                                )

            return result
        decorated_func._logged_decorator = True
        return decorated_func
    return decorator

def log_method_calls(time_format):
    def decorator(cls):
        for attr in dir(cls):
            if attr.startswith('__’): #过滤掉以双下划线开头的attributes
                continue
            a = getattr(cls, attr)
            if hasattr(a, '__call__’): #如果包含__call__属性，说明是函数
                decorated_a = logged(time_format, cls.__name__ + ".")(a)
                setattr(cls, attr, decorated_a)
        return cls
    return decorator

@log_method_calls("%b %d %Y - %H:%M:%S")
class A(object):
    def test1(self):
        print "test1"

@log_method_calls("%b %d %Y - %H:%M:%S")
class B(A):
    def test1(self):
        super(B, self).test1()
        print "child test1"

    def test2(self):
        print "test2"

b = B()
b.test1()
b.test2()
```

输出如下：

```python
- Running 'B.test1' on Jul 24 2013 - 14:15:03
- Running 'A.test1' on Jul 24 2013 - 14:15:03
test1
- Finished 'A.test1', execution time = 0.000s
child test1
- Finished 'B.test1', execution time = 1.001s
- Running 'B.test2' on Jul 24 2013 - 14:15:04
test2
- Finished 'B.test2', execution time = 2.001s
```

### 4. descriptor

* 用法1：Type Checking

```python
# Descriptor for a type-checked attribute
class Typed:
    def __init__(self, name, expected_type):
        self.name = name
        self.expected_type = expected_type
    def __get__(self, instance, cls):
        if instance is None:
            return self
        else:
            return instance.__dict__[self.name]
    def __set__(self, instance, value):
        if not isinstance(value, self.expected_type):
            raise TypeError('Expected ' + str(self.expected_type))
        instance.__dict__[self.name] = value

    def __delete__(self, instance):
        del instance.__dict__[self.name]


# Class decorator that applies it to selected attributes
def typeassert(**kwargs):
    def decorate(cls):
        for name, expected_type in kwargs.items():
            # Attach a Typed descriptor to the class
            setattr(cls, name, Typed(name, expected_type))
        return cls
    return decorate

# Example use
@typeassert(name=str, shares=int, price=float)
class Stock:
    def __init__(self, name, shares, price):
        self.name = name
        self.shares = shares
        self.price = price
```
Finally, it should be stressed that you would probably not write a descriptor if you simply want to customize the access of a single attribute of a specific class. Descriptors are more useful in situations where there will be a lot of code reuse (i.e., you want to use the functionality provided by the descriptor in hundreds of places in your code or provide it as a library feature).

* 用法2：Lazily Computed Properties

```python
class lazyproperty:
    def __init__(self, func):
        self.func = func
    def __get__(self, instance, cls):
        if instance is None:
            return self
        else:
            value = self.func(instance)
            setattr(instance, self.func.__name__, value)
            return value
```

If a descriptor only defines a __get__() method, it has a much weaker binding than usual. In particular, the __get__() method only fires if the attribute being accessed is not in the underlying instance dictionary.