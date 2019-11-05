# Python函数闭包及用途

> 定义：闭包是指延展了作用域的函数，可访问定义体之外的**非全局变量**

tag: 闭包，作用域，装饰器

目录

* 函数作用域
  * global声明
* 创造一个闭包
  * 函数内的lambda函数也是闭包
* 使函数在调用间保存状态
  * nonlocal声明
* 为装饰器加上参数
* 处理被装饰函数的参数
* 练习

## 全局变量的作用域

请看下面这段代码：

```python
prev1, prev2 = 0, 1
def fib():
    # global prev1, prev2
    print("before: prev1: %d, prev2: %d" % (prev1, prev2))
    prev1, prev2 = prev2, prev1+prev2 # if global variable been modified even in the middle of function body, it will been interpreted as a local variable all over function body
    print("after: prev1: %d, prev2: %d" % (prev1, prev2))

fib()
print("outside function: prev1: %d, prev2: %d" % (prev1, prev2))
```

你可能注意到里代码中有一行被注释掉了的```global prev1, prev2```，我们将分别讨论加上这一行与否的情况

依照一般的逻辑，在第一次print的时候，prev1与prev2应该是全局变量，因为我们还没有在定义它。

下面一行的```prev1, prev2 = prev2, prev1+prev2```，“=”右边应当也是全局变量，而等号右边而“=”左边的prev1与prev2应该是局部变量，因为我们重新给它们重新赋值了，而Python正常情况下是不能修改全局变量的

以下是运行结果

```python
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 3, in fib
UnboundLocalError: local variable 'prev1' referenced before assignment
```

这个报错信息显示，在第一次print中使用局部变量prev1的时候，我们还没有定义它。
为什么此时prev1会是局部变量呢？这是因为由于我们在这个函数体的后面更改了它的值，解释器会将这个变量在整个作用域内解释为一个局部变量。

这可能有一些反直觉，我们在某行代码之后的行为，居然会影响到这行代码是如何被解释的！

如果要正确地在函数体中更改全局变量的值，使用global声明这prev1, prev2为全局变量，就像我们在注释的那一行做的一样：```global prev1, prev2```，以下是运行结果

```python
before: prev1: 0, prev2: 1
after: prev1: 1, prev2: 1
outside function: prev1: 1, prev2: 1
```

prev1, prev2的值确实被正确的更改了，这说明```global```关键字能将函数作用域内对某个变量的控制能力扩展到作用域外

## 创造一个闭包

要实现一个闭包很简单，在函数内再定义一个函数，内部的那个函数就成为了一个闭包，这个内部函数可以访问定义它的函数的作用域

这下我们就回到了闭包的定义：闭包是指延展了作用域的函数，可访问定义体之外的**非全局变量**

```python
def call(caller):
    print("Hi, this is %s calling!" % caller)
    def answer(answerer): # this function is a closure
        print("Hi %s, this is %s answering!" % (caller, answerer))
    return answer

phoneCall = call("Ming") # phoneCall: <function call.<locals>.answer at 0x000001FDD2E23E18>
"""Hi, this is Ming calling!"""
phoneCall("Fan")
"""Hi Ming, this is Fan answering!"""
```

我们在call函数中定义了一个函数answer，这个answer函数就像一个普通变量一样，作为call函数的返回值
这个例子中answer函数可以访问call的局部变量caller

事情还没完，我们在调用完call函数之后，由于没有任何变量名指向它，这个函数的作用域和其中的变量（caller）应该被销毁了的，为何我们还能在下一行的answer函数中使用域中的值呢？由于phoneCall这个变量指向了call中的闭包answer，所以**函数的作用域必须等待其闭包销毁之后才会被销毁**。

这个特性我们可以在多次函数调用间用于保存某些变量的值

### 函数内的lambda函数也是一个闭包

我们也可以使用lambda函数定义匿名函数来实现上面的功能，lambda函数的作用域也是一个闭包

```python
def call(caller):
    print("Hi, this is %s calling!" % caller)
    return lambda answerer: print("Hi %s, this is %s answering!" % (caller, answerer))

phoneCall = call("Ming") # phoneCall: <function call.<locals>.<lambda> at 0x000001FDD314A400>
"""Hi, this is Ming calling!"""
phoneCall("Fan")
"""Hi Ming, this is Fan answering!"""
```

## 闭包用途一：使函数在调用间保存状态

设想一个函数，我们每次调用他的时候，它会返回斐波那契数列的下一个值，并保存当前状态以便下次调用时使用（有些像迭代器），我们就可以用闭包可以访问定义它的函数的作用域，且能使其延迟销毁的特性，来实现这个功能

来看下面这个例子

```python
def fibGenerator():
    a, b = 0, 1
    storedList = [a, b]
    def fibNext():
        nonlocal a, b  # nonlocal声明a, b不是本地变量，以便更改
        a, b = b, a+b
        return b
    return fibNext

FibGenerator = fibGenerator()
print([FibGenerator() for i in range(5)]) # [1, 2, 3, 5, 8]
print(FibGenerator()) # 13
```

nonlocal的用法类似于global，即使我们在闭包内能访问外层函数的变量，但只有我们使用nonlocal声明他为非本地变量后，才能更改它的值

我们连续调用了函数六次，每次这个返回数列的下一个值，在这个过程中，每次调用FibGenerator()是互不相关的，但数列的状态仍保存在闭包所处的作用域之中

## 闭包用途二：为装饰器加上参数

装饰器的本质是一个接收函数作为参数，并返回函数的特殊函数，并且只具有一个参数func，普通的装饰器，如@decorator这种形式，无法传入参数。

而将含有闭包的函数作为装饰器，可多一层函数嵌套结构为变为```@decorator(args)```（实际上decorator函数返回的内部函数才是装饰器），此时括号内可传入所需参数供闭包内使用

```python
funcs = {}

def functionRegister(key):
    def regist(func):
        funcs[key] = func
        return func
    return regist

@functionRegister("ADD")
def add(a,b): return a+b

@functionRegister("MINUS")
def add(a,b): return a+b

print(funcs)
"""
{'ADD': <function add at 0x0000023B04B2B158>,
'MINUS': <function add at 0x0000023B04B2B1E0>}
"""
```

## 闭包用途三：处理被装饰函数的参数

普通的装饰器只能接收一个参数：被装饰器所装饰的函数。无法获取被装饰函数所传递的参数
  
而由于装饰器的本质是使用一个函数代替原有函数，故而我们可以使返回的函数的参数和原有函数的参数一致，这样我们就在内部函数中获取到所有传入的参数

```python
def IntCheck(func):
    def innerFunc(*args):
        if False in set(type(arg)==int for arg in args): # check type of values inside args
            raise Exception("Parameters of %s can only be integer" % func.__name__)
        return func(*args)
    return innerFunc

@IntCheck
def add(a,b): return a+b

@IntCheck
def multiple(a, b): return a*b

multiple(2, 3) # 6
add("Hello", "World") # Exception: Parameters of add can only be integer
```

在这个例子中，装饰器IntCheck返回它的内部函数innerFunc以代替原本的函数，innerFunc接受了本应add和multiple函数接受所有的参数，并逐一检查

我们希望add和multiple这两个函数的参数限制为int，诸如```"Hello"+"World"```这种虽然使用了“+”，但不是我们所期待的数字加法，我们可以通过这种方式使string类型参数失效，而不改动函数本身

## 练习
一个实现一个initMonitor装饰器，使其能够装饰一个类的```__init__()```函数，每次```__init__```被调用前后提示，并显示其是如何被调用的（类、函数名、参数）

```python
import functools

def initMonitor(_type): # take parameter
    def funcWrapper(func): # actual decorator
        @functools.wraps(func)
        def targetFunc(*args, **kwargs): # this function will replace the function which been decorated
            # functioncall is a string that shows how this function been called
            functionCall = "%s.%s(%s%s)" % (_type,
                                            func.__name__,
                                            ", ".join(map(str,args[1:])),
                                            (", "+str(kwargs)[1:-1].replace("'", "").replace(": ","=")) if kwargs else "")
            print("enter %s" % functionCall)
            func(*args, **kwargs)
            print("exit %s" % functionCall)
        return targetFunc
    return funcWrapper

if __name__ == "__main__":
    class A:
        @initMonitor('A') # this parameter (parsed as _type) can be replaced by self.__class__ or type(self) inside closure targetFunc(*args, **kwargs)
        def __init__(self, a, b=2):
            print("start init A")

    A(1,b=2)
    """
    enter A.__init__(1, b=2)
    start init A
    exit A.__init__(1, b=2)
    """
    A(1)
    """
    enter A.__init__(1)
    start init A
    exit A.__init__(1)
    """
```
