--- 
layout: post
title: python学习之特性、静态方法、类方法
subtitle: property, staticmethod, classmethod之间的区别
date: 2018-07-06
author: YangSijie
header-img: img/post-bg-alibaba.jpg
catalog: true
tags:
    - python
---

# 特性(property)、静态方法(staticmethod)、类方法(classmethod)

### 1. property特性：

property是一种特殊的属性，访问它时会执行一段函数，然后返回值。

当一个==类函数==定义成property之后，用户使用`obj.func`，无法察觉运行的是一个==类变量==，还是==类函数==，遵循了**统一访问的原则**。

```python
class test_property(object):
    def __init__(self, a):
        self.a = a

    @property
    def func1(self):	# 将该函数定义为property
        return self.a + 1

x = test_property(2)
print(x.func1)			# 像类变量一样调用该函数
```

在C++中，可以使用自定义的`get`或者`set`函数去获取或设置class中的==私有数据==。在python中可以使用`property`方法去实现。如下代码所示：

```python
class test_property(object):
    def __init__(self, a):
        self._a = a

    @property
    def a(self):		# 相当于get方法
        return self._a

    @a.setter			# 相当于set方法
    def a(self, value):
        self._a = value

    @a.deleter			# 相当于del方法
    def a(self):
        raise TypeError('Can not delete.')


x = test_property(2)
print(x.a)	# 获取类变量
x.a = 10	# 为类变量赋值
print(x.a)
del x.a		# 删除类变量
```

> 打印出的结果如下：
>
> ```
> 2
> Traceback (most recent call last):
> 10
>   File "D:/pycharm/PycharmProjects/test/test_property.py", line 22, in <module>
>     del x.a
>   File "D:/pycharm/PycharmProjects/test/test_property.py", line 15, in a
>     raise TypeError('Can not delete.')
> TypeError: Can not delete.
> ```

### 2. staticmethod静态方法

将类中的函数作为类静态方法，这样就可以直接使用该类调用该函数，而无需生成实例后，再调用该函数。

```python
class test_staticmethod(object):
    @staticmethod
    def a(x, y):	# 注意该函数的第一个参数无需是self
        print(x, y)


test_staticmethod.a(1, 2)	# 可以直接使用类调用方法，无需实例化
```

### 3. classmethod类方法

类方法是给类使用的，类在使用该方法的时候，会把==该类本身==作为参数传递给类方法的第一个参数，就像类中的函数的第一个参数为self类似，只不过后者的第一个参数为该类自身，而前者的第一个参数可以是其他类。

类方法也可以直接由该类调用，无需生成实例后，再调用该函数。

```python
class test_classmethod(object):
    @classmethod
    def a(cls, a, b):	# 注意该函数的第一个参数是cls，表示调用该方法的类
        print (a, b)

    
test_classmethod.a(1, 2)	# 可以直接使用类调用方法，无需实例化
```

看下面这个例子，体会该方法的使用环境：

```python
import time
class Date:
    def __init__(self,year,month,day):
        self.year=year
        self.month=month
        self.day=day
    # @staticmethod
    # def now():
    #     t=time.localtime()
    #     return Date(t.tm_year,t.tm_mon,t.tm_mday)

    @classmethod #改成类方法
    def now(cls):
        t=time.localtime()
        return cls(t.tm_year,t.tm_mon,t.tm_mday) #哪个类来调用,即用哪个类cls来实例化

class EuroDate(Date):
    def __str__(self):
        return 'year:%s month:%s day:%s' %(self.year,self.month,self.day)

e=EuroDate.now() # 此时传入now()的参数及EuroData这个类
print(e) #我们的意图是想触发EuroDate.__str__,此时e就是由EuroDate产生的,所以会如我们所愿
```

> 输出结果为：
>
> ```
> year:2017 month:3 day:3
> ```

> 注意：`__str__`是用于打印类产生的实例时，会触发的方法