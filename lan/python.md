<!-- TOC -->

- [1 不可变](#1-不可变)
    - [1.1 参数传递（类似Java）](#11-参数传递类似java)
- [2 类方法和静态方法](#2-类方法和静态方法)
- [3 单下划线、双下划线、头尾双下划线说明：](#3-单下划线双下划线头尾双下划线说明)
- [4 私有属性](#4-私有属性)
- [5 可变参数](#5-可变参数)
    - [5.1 可变参数](#51-可变参数)
    - [5.2 关键字参数](#52-关键字参数)
    - [5.2 关键字参数](#52-关键字参数)
    - [5.3 命名关键字参数](#53-命名关键字参数)
- [6 main方法](#6-main方法)
- [7 静态语言 vs 动态语言](#7-静态语言-vs-动态语言)
    - [7.1 动态绑定](#71-动态绑定)
- [8 元类](#8-元类)
    - [8.1 type](#81-type)
    - [8.2 metaclass](#82-metaclass)
- [metaclass是类的模板，所以必须从`type`类型派生：](#metaclass是类的模板所以必须从type类型派生)
- [metaclass是类的模板，所以必须从`type`类型派生：](#metaclass是类的模板所以必须从type类型派生)
- [9 反射](#9-反射)

<!-- /TOC -->

# 1 不可变
在 python 中，strings, tuples, 和 numbers 是不可更改的对象，而 list,dict 等则是可以修改的对象。
## 1.1 参数传递（类似Java）
- 不可变类型：类似 c++ 的值传递，如 整数、字符串、**元组**。比如在 fun（a）内部修改 a 的值，只是修改另一个复制的对象，不会影响 a 本身。
- 可变类型：类似 c++ 的引用传递，如 列表，字典。如 fun（la），则是将 la 真正的传过去，修改后fun外部的la也会受影响

默认参数必须指向不变对象！

# 2 类方法和静态方法
在Python中，类方法（Class Method）与静态方法（Static Method）类似，都是属于类的方法，可以通过类名直接调用。然而，它们在一些方面有所不同。

主要区别如下：
1. 第一个参数的不同：在类方法中，第一个参数被约定为类本身，通常命名为cls；而在静态方法中，没有隐式的第一个参数。
2. 访问类属性的能力：类方法可以访问和修改类的属性，也可以通过类名调用其他类方法；而静态方法只能访问类的静态属性，不能访问或修改其他类属性。

# 3 单下划线、双下划线、头尾双下划线说明：
- \_\_foo\_\_: 定义的是特殊方法，一般是系统定义名字 ，类似 \_\_init__() 之类的。
- _foo: 以单下划线开头的表示的是 protected 类型的变量，即保护类型只能允许其本身与子类进行访问，不能用于 from module import *
- __foo: 双下划线的表示的是私有类型(private)的变量, 只能是允许这个类本身进行访问了。
- 单下划线_：有时用作临时或无意义变量的名称（“不关心”）。此外还能表示Python REPL会话中上一个表达式的结果。

# 4 私有属性
双下划线前缀会让Python解释器重写属性名称，以避免子类中的命名冲突。

这也称为名称改写（name mangling），即解释器会更改变量的名称，防止子类覆盖这些变量。

所以对于双下划线可以使用 object._className__attrName（ 对象名._类名__私有属性名 ）访问属性，参考以下实例：

```python
class Runoob:
    __site = "www.runoob.com"

runoob = Runoob()
print runoob._Runoob__site

```
执行以上代码，执行结果如下：
```
www.runoob.com
```

# 5 可变参数
## 5.1 可变参数
\* 代表 tuple
** 代表 dict

```python
def calc(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum

>>> calc(1, 2)
>>> nums = [1, 2, 3]
>>> calc(*nums)
```
## 5.2 关键字参数
```py
def person(name, age, **kw):
    print('name:', name, 'age:', age, 'other:', kw)
>>> person('Adam', 45, gender='M', job='Engineer')
```

## 5.3 命名关键字参数
如果要限制关键字参数的名字，就可以用命名关键字参数，例如，只接收city和job作为关键字参数。这种方式定义的函数如下：
```py
def person(name, age, *, city, job):
    print(name, age, city, job)

>>> person('Jack', 24, city='Beijing', job='Engineer')
Jack 24 Beijing Engineer
```

# 6 main方法
```py
if __name__=='__main__':
```
当你直接运行这个脚本时，__name__变量的值将是__main__，这样main函数就会被调用。如果脚本被导入为模块，__name__的值将是模块的名称，而main函数就不会被调用。

# 7 静态语言 vs 动态语言
对于静态语言（例如Java）来说，如果需要传入Animal类型，则传入的对象必须是Animal类型或者它的子类，否则，将无法调用run()方法。

对于Python这样的动态语言来说，则不一定需要传入Animal类型。我们只需要保证传入的对象有一个run()方法就可以了
1. 类型检查：
   - 静态类型：编译时
   - 动态类型：在运行时进行类型检查，变量的类型可以在**运行时根据赋值操作动态推断或改变**。

2. 类型声明：
   - 静态类型：要求在变量声明或函数签名中**明确指定**变量的类型。
   - 动态类型：通常不需要显式声明变量的类型，类型是根据赋值操作**自动推断**的。

3. 开发效率：
   - 静态类型：编译器可以提供更好的错误检查和类型推断，辅助工具如代码补全、重构等支持开发效率，适合大型项目和团队开发。
   - 动态类型：通常编写的代码更简洁，无需显式类型声明，适合快速原型开发、脚本编写和小型项目。

4. 性能：
   - 静态类型：**编译器可以进行更多的优化**，生成更高效的代码，因为它在**编译时了解变量的类型信息**。
   - 动态类型：通常在运行时需要进行类型检查和类型转换，可能会有一些运行时开销，但现代的动态类型语言也在尽力优化性能。

需要注意的是，并非所有编程语言都是纯粹的静态类型或动态类型。有些语言提供了一定的灵活性，例如Python具有动态类型特性，但也可以通过类型注解实现静态类型检查。

## 7.1 动态绑定
动态绑定属性和方法
```py
class Student(object):
    pass

>>> s = Student()
>>> s.name = 'Michael' # 动态给实例绑定一个属性
>>> print(s.name)
Michael
```

```py
>>> def set_age(self, age): # 定义一个函数作为实例方法
...     self.age = age
...
>>> from types import MethodType
>>> s.set_age = MethodType(set_age, s) # 给实例绑定一个方法
>>> s.set_age(25) # 调用实例方法
>>> s.age # 测试结果
25
```
可以用__slots__限制绑定


# 8 元类
## 8.1 type
```py
>>> def fn(self, name='world'): # 先定义函数
...     print('Hello, %s.' % name)
...
>>> Hello = type('Hello', (object,), dict(hello=fn)) # 创建Hello class
>>> h = Hello()
>>> h.hello()
Hello, world.
>>> print(type(Hello))
<class 'type'>
>>> print(type(h))
<class '__main__.Hello'>
```
要创建一个class对象，type()函数依次传入3个参数：

1. class的名称；
2. 继承的父类集合，注意Python支持多重继承，如果只有一个父类，别忘了tuple的单元素写法；
3. class的方法名称与函数绑定，这里我们把函数fn绑定到方法名hello上。
通过type()函数创建的类和直接写class是完全一样的，因为Python解释器遇到class定义时，仅仅是扫描一下class定义的语法，然后调用type()函数创建出class。

## 8.2 metaclass
[元类](https://www.liaoxuefeng.com/wiki/1016959663602400/1017592449371072)
获取类的属性，各类增加属性、方法

感觉功能类似于Java反射、动态代理、字节码操作库

```py
# metaclass是类的模板，所以必须从`type`类型派生：
class ListMetaclass(type):
    def __new__(cls, name, bases, attrs):
        attrs['add'] = lambda self, value: self.append(value)
        return type.__new__(cls, name, bases, attrs)
```

# 9反射
- dir() 函数：dir() 函数用于获取对象的所有属性和方法的名称列表。
- getattr()、setattr() 和 delattr() 函数：这些函数用于动态获取、设置和删除对象的属性。
    - getattr(obj, name) 可以获取对象 obj 的属性 name 的值；
    - setattr(obj, name, value) 可以设置对象 obj 的属性 name 的值为 value；
    - delattr(obj, name) 可以删除对象 obj 的属性 name。
- hasattr() 函数：hasattr(obj, name) 函数用于检查对象 obj 是否具有属性 name。如果对象拥有该属性，则返回 True；否则返回 False。
- \_\_dict__ 属性：对象的实例属性字典
- type() 函数：type(obj) 函数返回对象 obj 的类型。你可以使用它来获取对象的类，进而获取类的属性和方法信息。
