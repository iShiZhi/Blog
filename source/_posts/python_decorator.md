---
title: Python 装饰器
categories: 开发语言
tags: [Python]
date: 2017-06-26
---

### 模式动机
通常情况下，在不改变类或者对象功能的基础上，我们有两种方式可以实现给一个类或对象增加行为：
- 继承机制：通过继承一个现有类可以使得子类在拥有自身方法的同时还拥有父类的方法。但是这种方法是静态的，用户不能控制增加行为的方式和时机。
- 关联机制：将一个类的对象嵌入另一个对象中，由另一个对象来决定是否调用嵌入对象的行为以便扩展自己的行为，我们称这个嵌入的对象为装饰器 (Decorator)

装饰器本质上是在不对现有函数做任何代码变动的前提下，增加额外的功能。它经常用于有切面需求的场景，比如：插入日志、性能测试、事务处理、缓存、权限校验等场景。

### 基础：函数是对象
在 Python 中，函数也是对象，它可以：
- 赋值给变量
- 作为形参传递给其它的函数
- 作为函数的返回值
- 在一个函数的内部定义另外一个函数

而这些就是 Python 装饰器的基础。

### 简单装饰器
假设我们当前有一个处理业务逻辑的函数，如下：
```python
def business_logic():
    print('deal with business logic')
```
现在有一个需求，希望可以记录下函数调用的日志，于是在函数中添加日志代码：
```python
def business_logic():
    logging.info('business_logic is invoked!')
    print('deal with business logic')
```
而在正常的开发过程中，会有很多函数有类似的需求。为了减少重复写代码，我们可以重新定义一个函数：专门处理日志 ，日志处理完之后再执行真正的业务代码：
```python
def api_logging(func):
    logging.info("%s is invoked!" % func.__name__)
    func()

def business_logic():
    print('deal with business logic')
```
但是这样的话，我们每次都需要将包含业务逻辑的函数作为参数传递给 api_logging 函数。这样的话，就破坏了原有的调用逻辑。
使用装饰器模式，我们可以把上述的逻辑改为下面的实现：
```python
def api_logging(func):
    def wrapper(*args, **kwargs):
        logging.info("%s is invoked!" % func.__name__)
        return func(*args, **kwargs)
    return wrapper

def business_logic():
    print('deal with business logic')

business_logic = api_logging(business_logic)
business_logic()
```
通过这样的方式，直接调用 business_logic 就可以按照期望地打印调用日志，并且执行业务逻辑了。但是这样每次定义新函数的时候，需要多加一个函数赋值语句。
Python 提供了一个语法糖，能省去这个函数赋值语句，就是 @ 符号。我们可以把上述的代码改为如下形式：
```python
def api_logging(func):
    def wrapper(*args, **kwargs):
        logging.info("%s is invoked!" % func.__name__)
        return func(*args, **kwargs)
    return wrapper

@api_logging
def business_logic():
    print('deal with business logic')

business_logic()
```
这样就创建了一个简单的装饰器。

### functions.wraps
使用装饰器有一个缺点，就是原函数的原信息不见了，比如 __name__、__doc__ 等，具体例子如下：
```python
def api_logging(func):
    def wrapper(*args, **kwargs):
        print func.__name__
        print func.__doc__
        logging.info("%s is invoked!" % func.__name__)
        return func(*args, **kwargs)
    return wrapper

@api_logging
def business_logic():
    print('deal with business logic')

api_logging(business_logic)()
```
不难发现，函数 business_logic 的信息被 wrapper 取代了。Python 提供了一个装饰器 functools.wraps，它能把原函数的元信息拷贝到装饰器里面的 func 函数中，使得装饰器里面的 func 函数拥有和原函数一样的元信息。
```python
from functools import wraps
def api_logging(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print func.__name__
        print func.__doc__
        logging.info("%s is invoked!" % func.__name__)
        return func(*args, **kwargs)
    return wrapper
```

### 带参数的装饰器
装饰器还有更大的灵活性，例如上述的例子中，我们希望针对不同的业务接口的调用使用不同日志级别，那么需要给装饰器传入日志级别作为参数，具体需要的改动如下：
```python
def api_logging(level):
    def decorator(func):
        def wrapper(*args, **kwargs):
            if level == "warn":
                logging.warn("%s is invoked!" % func.__name__)
            elif level == "info":
                logging.info("%s is invoked!" % func.__name__)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@api_logging('warn')
def business_logic():
    print('deal with business logic')

business_logic()
```
上述处理方法的核心思想很简单。最外层的函数 api_logging() 接受参数并将它们作用在内部的装饰器函数上面。内层的函数 decorate() 接受一个函数作为参数，然后在函数上面放置一个包装器。这里的关键点是包装器是可以使用传递给 api_logging() 的参数。

### 类装饰器
相比函数装饰器，类装饰器更灵活，封装性也更好。当使用 @ 符号将类装饰器附加到函数上时，会调用类的 __call__ 方法：
```python
class Logging(object):
    def __init__(self, func):
        self._func = func

    def __call__(self):
        logging.info("%s is invoked!" % self._func.__name__)
        self._func()

@Logging
def business_logic():
    print('deal with business logic')

business_logic()
```
通过这种方式，我们可以在日志处理的逻辑发生多样化时，还可以继承 Logging 进行实现。

### 装饰器顺序
一个函数可以同时定义多个装饰器，比如：
```python
@decorator1
@decorator2
def func():
    pass
```
它的执行顺序是从里到外的，先执行最里面的装饰器，然后依次向外，直到最外层的装饰器，等效于：
```python
func = decorator1(decorator2(func))
```
