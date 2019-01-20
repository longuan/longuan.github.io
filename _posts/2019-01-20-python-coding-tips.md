---
title: "python编程技巧积累"
---


- [Python 新特性](#python-新特性)
    - [使用f-Strings来格式化字符串](#使用f-strings来格式化字符串)
    - [用Type Hinting来代替Doc String](#用type-hinting来代替doc-string)
- [不使用可变的数据类型作为函数参数](#不使用可变的数据类型作为函数参数)
- [使用property来处理getter和setter函数](#使用property来处理getter和setter函数)
- [使用with处理加锁](#使用with处理加锁)
- [容器类型使用技巧](#容器类型使用技巧)
    - [写扩展性更好的代码](#写扩展性更好的代码)
    - [避免频繁扩充列表/创建新列表](#避免频繁扩充列表/创建新列表)
    - [在列表头部操作多的场景用 deque 模块](#在列表头部操作多的场景用-deque-模块)
    - [使用集合字典来判断成员是否存在](#使用集合字典来判断成员是否存在)
    - [最好不用“获取许可”，也无需“要求原谅”](#最好不用获取许可也无需要求原谅)
- [对于函数return的技巧](#)
    - [单个函数不要返回多种数据类型](#单个函数不要返回多种数据类型)
    - [使用partial构造新函数](#使用partial构造新函数)
    - [尽量使用异常而不是返回结果处理错误](#尽量使用异常而不是返回结果处理错误)
    - [合理使用空对象模式](#合理使用空对象模式)
    - [尽量不使用递归](#尽量不使用递归)
- [参考](#参考)


--------------------------------------


## Python 新特性

### 使用f-Strings来格式化字符串

在Python3.6版本之前，主要有两种方式来格式化字符串：`%`和`str.format()`方法。


百分号类型的格式化字符串方法是从c语言继承过来的。由百分号和一个代表类型的字符组成占位符嵌入字符串，之后执行格式化。这里有几个问题：首先就是可读性，当字符串里百分号占位符多的时候，我们很可能会搞不清这个位置的占位符代表什么含义。其次是类型的问题，众所周知，Python的变量在使用的时候不用声明类型，而且**在对一个变量操作的时候，更应该关注其抽象属性，而不是其类型本身**。而且[官方文档](https://docs.python.org/3/library/stdtypes.html#printf-style-string-formatting)也不推荐这种做法。

format()方法是比较好的，定制化比较强，也更容易写出Pythonic的代码。但是仍然有可读性的问题。

*[f-Strings](https://www.python.org/dev/peps/pep-0498/)*:

f-Strings的想法很棒，就是直接用变量名来作为占位符，在运行时格式化，这样就可以做很多东西，包括直接调用函数。而且f-Strings默认调用对象的`__str__()`方法。并且可以使用在文档字符串。

```py
>>> name = "john"
>>> gender = True
>>> age = 23
>>> person = f"{name}'s gender is {gender}, and his age is {age}"
>>> person
"john's gender is True, and his age is 23"
>>> person = f"{name.upper()}'s gender is {gender}, and his age is {age}"
>>> person
"JOHN's gender is True, and his age is 23"
>>>
```

### 用Type Hinting来代替Doc String

[Type Hinting](https://www.python.org/dev/peps/pep-0484/)是Python3.5版本及之后的，为了兼容性还是老老实实使用Doc String吧。

```py
>>>def foo(x: int, y: str) -> bool:
...    return x < y

>>> foo.__annotations__()
{'y': <class 'str'>, 'return': <class 'bool'>, 'x': <class 'int'>}
```

这个特性有一个专门的库：typing 。

## 不使用可变的数据类型作为函数参数

关于[可变数据类型](https://medium.com/@meghamohan/mutable-and-immutable-side-of-python-c2145cf72747)

```python
def add_to(num, target=[]):
    target.append(num)
    return target

add_to(1)
# Output: [1]

add_to(2)
# Output: [1, 2]

add_to(3)
# Output: [1, 2, 3]
```

## 使用property来处理getter和setter函数


1. 使用property装饰器

```py
class Clock(object):
    def __init__(self):
        self.__hour = 1

    @property
    def hour(self):
        print("getting value ")
        return self.__hour

    @hour.setter
    def hour(self, value):
        print("setting value ")
        self.__hour = value

    @hour.deleter
    def hour(self):
      del self.__hour

c = Clock()
print(c.hour)
c.hour = 12
```

2. 使用property函数

```py
class Clock(object):
    def __init__(self):
        self.__hour = 10
        print(self.__hour is self.hour)

    def __setHour(self, hour):
        print("setting value")
        if 25 > hour > 0: 
            self.__hour = hour
        else: 
            raise BadHourException

    def __getHour(self):
        print("Getting value")
        return self.__hour

    hour = property(__getHour, __setHour)

c = Clock()
print(c.hour)
c.hour = 12
```

需要注意的是，上面一段代码中`print(self.__hour is self.hour)`的输出是`True`。`self.hour`是property的一个实例，它操作的还是`self.__hour`

上面一行`hour = property(__getHour, __setHour)`也可以分开写：
```py
# make empty property
hour = property()
# assign fget
hour = hour.getter(__getHour)
# assign fset
hour = hour.setter(__setHour)
```

## 使用with处理加锁

```py
##不推荐
import threading
lock = threading.Lock()

lock.acquire()
try:
    # 互斥操作...
finally:
    lock.release()


##推荐
import threading
lock = threading.Lock()

with lock:
    # 互斥操作...
```

## 容器类型使用技巧

来自： [Python工匠：解析容器类型的门道](https://mp.weixin.qq.com/s?__biz=MzUyOTk2MTcwNg==&mid=2247483943&idx=1&sn=cdbaf74444e7097497309898e5084825&chksm=fa5845a2cd2fccb46c9478e9c8ccd1fce94bb8f72b5cedc6c64537add564646c0d40f79a0052&mpshare=1&scene=24&srcid=0118gWkB7S8NqAG9BVKyHavm#rd)

> 从高层来看，什么定义了容器？
> 
> 答案是：各个容器类型实现的接口协议定义了容器。不同的容器类型在我们的眼里，应该是 是否可以迭代、是否可以修改、有没有长度 等各种特性的组合。我们需要在编写相关代码时，更多的关注容器的抽象属性，而非容器类型本身，这样可以帮助我们写出更优雅、扩展性更好的代码。

### 写扩展性更好的代码

让函数依赖“可迭代对象”这个抽象概念，而非实体列表类型。

### 避免频繁扩充列表/创建新列表

- 更多的使用 yield 关键字，返回生成器对象
- 尽量使用生成器表达式替代列表推导表达式
  - 生成器：`(i for in range(100))`
  - 列表推导式：`[i for in range(100)]`
- 尽量使用模块提供的懒惰对象
  - 使用 `re.finditer` 替代 `re.findall`
  - 直接使用可迭代的文件对象：`for line in fp`，而不是`for line in fp.readlines()`

### 在列表头部操作多的场景用 deque 模块

列表是基于数组结构（Array）实现的，当你在列表的头部插入新成员（list.insert(0, item)）时，它后面的所有其他成员都需要被移动，操作的时间复杂度是 O(n)。这导致在列表的头部插入成员远比在尾部追加（list.append(item) 时间复杂度为 O(1)）要慢。

如果你的代码需要执行很多次这类操作，请考虑使用 collections.deque 类型来替代列表。因为 deque 是基于双端队列实现的，无论是在头部还是尾部追加元素，时间复杂度都是 O(1)。

### 使用集合/字典来判断成员是否存在

当你需要判断成员是否存在于某个容器时，用集合比列表更合适。因为 item in [...] 操作的时间复杂度是 O(n)，而 item in {...} 的时间复杂度是 O(1)。这是因为字典与集合都是基于哈希表（Hash Table）数据结构实现的。

### 最好不用“获取许可”，也无需“要求原谅”

如果用一个经典的需求：“计算列表内各个元素出现的次数” 来作为例子，两种不同风格的代码会是这样：

```py
# AF: Ask for Forgiveness
# 要做就做，如果抛出异常了，再处理异常
def counter_af(l):
    result = {}
    for key in l:
        try:
            result[key] += 1
        except KeyError:
            result[key] = 1
    return result


# AP: Ask for Permission
# 做之前，先问问能不能做，可以做再做
def counter_ap(l):
    result = {}
    for key in l:
        if key in result:
            result[key] += 1
        else:
            result[key] = 1
    return result
```

整个 Python 社区对第一种 Ask for Forgiveness 的异常捕获式编程风格有着明显的偏爱。这其中有很多原因，首先，在 Python 中抛出异常是一个很轻量的操作。其次，第一种做法在性能上也要优于第二种，因为它不用在每次循环的时候都做一次额外的成员检查。

不过，示例里的两段代码在现实世界中都非常少见。为什么？因为如果你想统计次数的话，直接用collections.defaultdict就可以：
```py
from collections import defaultdict


def counter_by_collections(l):
    result = defaultdict(int)
    for key in l:
        result[key] += 1
    return result
```

一些小提示：
- 操作字典成员时：使用collections.defaultdict类型，或者使用dict[key] = dict.setdefault(key, 0) + 1 内建函数
- 如果移除字典成员，不关心是否存在：调用 pop 函数时设置默认值，比如 dict.pop(key, None)
- 在字典获取成员时指定默认值：dict.get(key, default_value)
- 对列表进行不存在的切片访问不会抛出 IndexError 异常：["foo"][100:200]

## 对于函数return的技巧

来自[Python 工匠：让函数返回结果的技巧](https://github.com/piglei/one-python-craftsman/blob/master/zh_CN/5-function-returning-tips.md)


### 单个函数不要返回多种数据类型

### 使用partial构造新函数

`partial(func, *args, **kwargs)`

### 尽量使用异常而不是返回结果处理错误

### 合理使用空对象模式

当我们使用返回结果或者异常处理错误，上层函数调用的时候会频繁的使用if/else或者try/except，为了减少这种情况的出现，可以使用合理的“空对象”来代替判断。

### 尽量不使用递归

python对递归的支持非常有限，而且debug的时候非常痛苦，我之前就遇到过一个递归的问题，折磨了我很久，后来发现是因为——递归函数的每个逻辑分支都要有返回值，不能返回None。

还有几点：

- python不支持“尾递归优化”
- 递归最大层数有限制`sys.getrecursionlimit()`
- 尝试使用`functools.lru_cache`等缓存工具函数来降低递归层数

## 参考

- https://github.com/piglei/one-python-craftsman
- https://realpython.com/python-f-strings/#old-school-string-formatting-in-python
- https://medium.com/@meghamohan/mutable-and-immutable-side-of-python-c2145cf72747