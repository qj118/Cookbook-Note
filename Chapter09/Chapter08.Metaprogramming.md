# Chapter09. Metaprogramming

![mind map of Metaprogramming](Metaprogramming.gif)

- In a nutshell, metaprogramming is about creating functions and classes whose main goal is to manipulate code(e.g., modifying, generating, or wrapping existing code).

## 9.1 Putting a Wrapper Around a Function

- If you ever need to wrap a function with extra code, define a decorator function.
```python
import time
from functools import wraps

def timethis(func):
    '''
    Decorator that reports the execution time.
    '''
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(func.__name__, end - start)
        return result
    return wrapper

@timethis
def countdown(n):
    '''
    Counts down
    '''
    while n > 0:
        n -= 1

countdown(100000)   # countdown 0.009999990463256836
countdown(10000000) # countdown 1.0709998607635498
```

- A decorator is a function that accepts a function as input and returns a new function as output. The following two code fragments is equivalent.
```python
@timethis
def countdown(n):
    ...
```
```python
def countdown(n):
    ...
countdown = timethis(countdown)
```

- As an aside, built-in decorators such as *@staticmethod*, *@classmethod*, and *@property* work in the same way.
- The return value of a decorator is almost always the result of calling `fun(*args, **kwargs)`, where *func* is the original unwrapped function.

## 9.2 Preserving Function Metadata When Writing Decorators

- Whenever you define a decorator, you should always remember to apply the *@wraps* decorator from the *functools* library to the underlying wrapper function.
```python
>>> countdown.__name__
'countdown'
>>> countdown.__doc__
'\n\tCounts down\n\t'
```

- If you forget to use *@wraps*, you'll find that the decorated function loses all sorts of useful information.
- An important feature of the *@wraps* decorator is that it makes the wrapped function available to you in the `__wrapped__` attribute.
- The presence of the `__wrapped__` attribute also makes decorated functions properly expose the underlying signature of the wrapped function.

## 9.3 Unwrapping a Decorator

- Assuming that the decorator has been implemented properly using *@wraps*, you can usually gain access to original function by accessing the `__wrapped__` attribute.
- However, this recipe only works if the implementation of a decorator properly copies metadata using *@wraps* from the *functools* module or sets the `__wrapped__` attribute directly.
```python
>>> @somedecorator
>>> def add(x, y):
    return x + y

>>> orig_add = add.__wrapped__
>>> orig_add(3, 4)
7
```

## 9.4 Defining a Decorator That Takes Arguments

- Let's illustrate teh process of accepting arguments with an example.
```python
from functools import wraps
import logging

def logged(level, name = None, message = None):
    '''
    Add logging to a function. level is the logging level,
    name is the logger name, and message is the log message.
    If name and message aren't specified, they default to the
    function's module and name.
    '''
    def decorate(func):
        logname = name if name else func.__module__
        log = logging.getLogger(logname)
        logmsg = message if message else func.__name__

        @wraps(func)
        def wrapper(*args, **kwargs):
            log.log(level, logmsg)
            return func(*args, **kwargs)
        return wrapper
    return decorate

@logged(logging.DEBUG)
def add(x, y):
    return x + y

@logged(logging.CRITICAL, 'example')
def spam():
    print('Spam!')
```

- The outermost function *logged()* accepts the desired arguments and simply makes them available to the inner functions of decorator. 
- The inner function *decorate()* accepts a function and puts a wrapper around it as normal.
- The key part is that wrapper is allowed to use the arguments passed to *logged()*.

## 9.5 Defining a Decorator with User Adjustable Attributes

- Here is a solution that expands on the last recipe by intorducing accessor functions that change internal variables through the use of *nonlocal* variable declarations. The accessor functions are then attached to the wrapper function as function attributes.
```python
from functools import wraps, partial
import logging

def attach_wrapper(obj, func = None):
    if func is None:
        return partial(attach_wrapper, obj)
    setattr(obj, func.__name__, func)
    return func

def logged(level, name = None, message = None):
    '''
    Add logging to a function. level is the logging level,
    name is the logger name, and message is the log message.
    If name and message aren't specified, they default to the
    function's module and name.
    '''
    def decorate(func):
        logname = name if name else func.__module__
        log = logging.getLogger(logname)
        logmsg = message if message else func.__name__

        @wraps(func)
        def wrapper(*args, **kwargs):
            log.log(level, logmsg)
            return func(*args, **kwargs)

        @attach_wrapper(wrapper)
        def set_level(newlevel):
            nonlocal level
            level = newlevel

        @attach_wrapper(wrapper)
        def set_message(newmsg):
            nonlocal logmsg
            logmsg = newmsg

        return wrapper
    return decorate

@logged(logging.DEBUG)
def add(x, y):
    return x + y

# ===============================================
# >>> import logging
# >>> logging.basicConfig(level = logging.DEBUG)
# >>> add(2, 3)
# DEBUG:__main__:add
# 5
# >>> add.set_message('Add called')
# >>> add(2, 3)
# DEBUG:__main__:Add called
# 5
# >>> add.set_level(logging.WARNING)
# >>> add(2, 3)
# WARNING:__main__:Add called
# 5
```

- The key to this recipe lies in the accessor functions that get attached to the wrapper as attributes.
- Each of these accessors allows internal parameters to be adjusted through the use of *nonlocal* assignment.

## 9.6 Defining a Decorator That Takes an Optional

- To find a straightforward way to do it due to differences in calling conventions between simpe decorators and decorators taking arguments.
```python
from functools import wraps, partial
import logging

def logged(func = None, *, level = logging.DEBUG, name = None, message = None):
    if func is None:
        return partial(logged, level = level, name = name, message = message)

    logname = name if name else func.__module__
    log = logging.getLogger(logname)
    logmsg = message if message else func.__name__

    @wraps(func)
    def wrapper(*args, **kwargs):
        log.log(level, logmsg)
        return func(*args, **kwargs)
    return wrapper

@logged
def add(x, y):
    return x + y

@logged(level = logging.CRITICAL, name = 'example')
def spam():
    print('Spam!')
```

- When arguments are passed, a decorator is supposed to return a function that accepts the function and wraps it.

## 9.7 Enforcing Type Checking on a Function Using a Decorator

- The aim of the recipe is to have a means of enforcing type contracts on the input arguments to a function.
```python
from inspect import signature
from functools import wraps

def typeassert(*ty_args, **ty_kwargs):
    def decorate(func):
        if not __debug__:
            return func

        sig = signature(func)
        bound_types = sig.bind_partial(*ty_args, **ty_kwargs).arguments

        @wraps(func)
        def wrapper(*args, **kwargs):
            bound_values = sig.bind(*args, **kwargs)
            for name, value in bound_values.arguments.items():
                if name in bound_types:
                    if not isinstance(value, bound_types[name]):
                        raise TypeError('Argument {} must be {}'.format(name, bound_types[name]))
            return func(*args, **kwargs)
        return wrapper
    return decorate

@typeassert(int, z = int)
def spam(x, y, z = 42):
    print(x, y, z)

spam(1, 2, 3)       # 1 2 3
spam(1, 'hello', 3) # 1 hello 3
spam(1, 'hello', 'world')

# Traceback (most recent call last):
#   File "F:\Learning\Python\Cookbook\test.py", line 29, in <module>
#     spam(1, 'hello', 'world')
#   File "F:\Learning\Python\Cookbook\test.py", line 18, in wrapper
#     raise TypeError('Argument {} must be {}'.format(name, bound_types[name]))
# TypeError: Argument z must be <class 'int'>
```

- One aspect of decorators is that they only get applied once, at the time of function definition.
 + In the solution, the following code fragment returns the function unmodified if the value of the global `__debug__` variable is set to *False*(as is case when Python executes in optimized mode with the *-0* or *-00* options to the interpreter).
- A triky part of writing this decorator is that it involves examining and working with the argument signature of the function being wrapped.
 + The *inspect.signature()* function allows you to extract signature information from a callable.
 + In the first part of our decorator, we use the *bind_partial()* method of signatures to perform a **partial binding** of the supplied types to argument names.
 + In the actual wrapper function made by the decorator, the *sig.bind()* method is used. *bind()* is like *bind_partial()* except that it does not allow for missing arguments.
```python
>>> from inspect import signature
>>> def spam(x, y, z = 42):
    pass

>>> sig = signature(spam)
>>> print(sig)
(x, y, z=42)
>>> sig.parameters
mappingproxy(OrderedDict([('x', <Parameter "x">), ('y', <Parameter "y">), ('z', <Parameter "z=42">)]))
>>> sig.parameters['z'].name
'z'
>>> sig.parameters['z'].default
42
>>> sig.parameters['z'].kind
<_ParameterKind.POSITIONAL_OR_KEYWORD: 1>
>>> bound_types = sig.bind_partial(int, z = int)
>>> bound_types
<BoundArguments (x=<class 'int'>, z=<class 'int'>)>
>>> bound_types.arguments
OrderedDict([('x', <class 'int'>), ('z', <class 'int'>)])
>>> bound_value = sig.bind(1, 2, 3)
>>> bound_value.arguments
OrderedDict([('x', 1), ('y', 2), ('z', 3)])
```

- A somewhat subtle aspect of the solution is that the assertions do not get applied to unsupplied arguments with default values.
- By using decorator arguments, as shown in the solution, the decorator becomes a lot more general purpose and can be used with any function whatsoever -- even functions that use annotations.

## 9.8 Defining Decorators As Part of a Class

- You want to define a decorator inside a class definition and apply it to other functions or methods.
- You first need to sort out the manner in which the decorator will be applied. Specifically, whether it is applied as an instance or a class method.
```python
from functools import wraps

class A:
    def decorator1(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            print('Decorator 1')
            return func(*args, **kwargs)
        return wrapper

    @classmethod
    def decorator2(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            print('Decorator 2')
            return func(*args, **kwargs)
        return wrapper

a = A()

@a.decorator1
def spam():
    pass

@A.decorator2
def grok():
    pass
```

- Suppose you want to supply one of the decorators defined in class A to methods defined in a subclass B. The decorator has to be defined as a class method and you have to explicitly use the name of the superclass A when applying.
```python
class B(A):
    @A.decorater2
    def bar(self):
        pass
```

## 9.9 Defining Decorators As Classes

- To define a decorator as an instance, you need to make sure it implements the `__call__()` and `__get__()` methods.
```python
import types
from functools import wraps

class Profiled:
    def __init__(self, func):
        wraps(func)(self)
        self.ncalls = 0

    def __call__(self, *args, **kwargs):
        self.ncalls += 1
        return self.__wrapped__(*args, **kwargs)

    def __get__(self, instance, cls):
        if instance is None:
            return self
        else:
            return types.MethodType(self, instance)

@Profiled
def add(x, y):
    return x + y

class Spam:
    @Profiled
    def bar(self, x):
        print(self, x)

# ======================================
# >>> add(2, 3)
# 5
# >>> add(4, 5)
# 9
# >>> add.ncalls
# 2
# >>> s = Spam()
# >>> s.bar(1)
# <__main__.Spam object at 0x022CFD10> 1
# >>> s.bar(2)
# <__main__.Spam object at 0x022CFD10> 2
# >>> s.bar(3)
# <__main__.Spam object at 0x022CFD10> 3
# >>> Spam.bar.ncalls
# 3
```

- There are some rather subtle details deserve more explanation, especially if you plan to apply the decorator to instance methods.
 1. The use of the *functools.wraps()* function serves the same purpose here as it does in normal decorators -- namely to copy important metadata from the wrapped function the callable instance.
 2. If you omit the `__get__()` and keep all of the other code the same, you'll find that bizarre things happen when you try to invoke decorated instance methods. The reason it breaks is that whenever functions implementing methods are looked up in a class, their `__get__()` method is invoked as part of the descriptor protocol.
 3. The purpose of `__get__()` is to create a bound method object. Bound methods only get created if an instance is being used. Here is an example that illustrates the underlying mechanics.
```python
>>> s = Spam()
>>> def grok(self, x):
    pass

>>> grok.__get__(s, Spam)
<bound method grok of <__main__.Spam object at 0x022CFD10>>
```

- You might consider an alternative formulation of the decorator using closures and *nonlocal* variables.
```python
import types
from functools import wraps

def profiled(func):
    ncalls = 0
    @wraps(func)
    def wrapper(*args, **kwargs):
        nonlocal ncalls
        ncalls += 1
        return func(*args, **kwargs)
    return wrapper

@profiled
def add(x, y):
    return x + y
```

## 9.10 Applying Decorators to Class and Static Methods

- Applying decorators to class and static methods is straightforward, but make sure that your decorators are applied before *@classmethod* or *@staticmethod*.
```python
import time
from functools import wraps

def timethis(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        r = func(*args, **kwargs)
        end = time.time()
        print(end - start)
        return r
    return wrapper

class Spam():
    @timethis
    def instance_method(self, n):
        print(self, n)
        while n > 0:
            n -= 1

    @classmethod
    @timethis
    def class_method(cls, n):
        print(cls, n)
        while n > 0:
            n -= 1

    @staticmethod
    @timethis
    def static_method(n):
        print(n)
        while n > 0:
            n -= 1

# ===========================================
# >>> s = Spam()
# >>> s.instance_method(100000)
# <__main__.Spam object at 0x022F0ED0> 100000
# 0.06400656700134277
# >>> Spam.class_method(100000)
# <class '__main__.Spam'> 100000
# 0.029002904891967773
# >>> Spam.static_method(100000)
# 100000
# 0.05400538444519043
```

- If you get the order of decorators wrong, you'll get error. The problem here is that *@classmethod* and *@staticmethod* don't actually create objects that directly callable. Instead, they create special descriptor objects.

## 9.11 Writing Decorators That Add Arguments to Wrapped Functions

- Extra arguments can be injected into the calling signature using keyword-only arguments.
```python
from functools import wraps

def optional_debug(func):
    @wraps(func)
    def wrapper(*args, debug = False, **kwargs):
        if debug:
            print('Calling', func.__name__)
        return func(*args, **kwargs)
    return wrapper

@optional_debug
def spam(a, b, c):
    print(a, b, c)

spam(1, 2, 3)
spam(1, 2, 3, debug = True)

# 1 2 3
# Calling spam
# 1 2 3
```

- By using a keyword-only argument, it gets singled out as a special case and removed from subsequent calls that only use the remaining positional and keyword arguments.
- If the *@optional_debug* decorator was applied to a function that already had a *debug* argument, then it would break. If that's concern, an extra check could be added.
```python
from functools import wraps
import inspect

def optional_debug(func):
    if 'debug' in inspect.getargspec(func).args:
        raise TypeError('debug argument already defined')

    @wraps(func)
    def wrapper(*args, debug = False, **kwargs):
        if debug:
            print('Calling', func.__name__)
        return func(*args, **kwargs)
    return wrapper
```

- The signature of wrapped functions is wrong.`print(inspect.signature(spam)) # (a, b, c)`
```python
from functools import wraps
import inspect

def optional_debug(func):
    if 'debug' in inspect.getargspec(func).args:
        raise TypeError('debug argument already defined')

    @wraps(func)
    def wrapper(*args, debug = False, **kwargs):
        if debug:
            print('Calling', func.__name__)
        return func(*args, **kwargs)
    sig = inspect.signature(func)
    parms = list(sig.parameters.values())
    parms.append(inspect.Parameter('debug', inspect.Parameter.KEYWORD_ONLY, default = False))
    wrapper.__signature__ = sig.replace(parameters = parms)
    return wrapper

@optional_debug
def spam(a, b, c):
    print(a, b, c)

print(inspect.signature(spam))

# (a, b, c, *, debug=False)
```

## 9.12 Using Decorators to Patch Class Definitions

- Here is a class decorator that rewrites the `__getattribute__` special method to perform logging.
```python
def log_getattribute(cls):
    # Get the original implementation
    orig_getattribute = cls.__getattribute__

    # Make a new definition
    def new_attribute(self, name):
        print('getting:', name)
        return orig_getattribute(self, name)

    cls.__getattribute__ = new_attribute
    return cls

@log_getattribute
class A:
    def __init__(self, x):
        self.x = x
    def spam(self):
        pass

# =====================
# >>> a = A(42)
# >>> a.x
# getting: x
# 42
# >>> a.spam()
# getting: spam
```

- In some sense, the class decorator solution is much more direct in how it operates, and it doesn't introduce new dependencies into the inheritance hierarchy.

## 9.13 Using a Metaclass to Control Instance Creation

- If you define a class, you call it like a function to create instances.
- If you want to customize this step, you can do it by defining a metaclass and reimplementing its `__call__()` method in some way.
- Now, suppose you want to implement this singleton pattern.
```python
class Singleton(type):
    def __init__(self, *args, **kwargs):
        self.__instance = None
        super().__init__(*args, **kwargs)

    def __call__(self, *args, **kwargs):
        if self.__instance is None:
            self.__instance = super().__call__(*args, **kwargs)
            return self.__instance
        else:
            return self.__instance

class Spam(metaclass = Singleton):
    def __init__(self):
        print('Creating Spam')

a = Spam() # Creating Spam
b = Spam()
print(a is b) # True
```

## 9.14 Capturing Class Attribute Definition Order 

- Capturing information about the body of class definition is easily accomplished through the use of a metaclass.
```python
from collections import OrderedDict

class Typed:
    _expected_type = type(None)
    def __init__(self, name = None):
        self.name = name

    def __set__(self, instance, value):
        if not isinstance(value, self._expected_type):
            raise TypeError(self._name + ' expects ' + str(self._expected_type))
        instance.__dict__[self._name] = value

class Integer(Typed):
    _expected_type = int

class Float(Typed):
    _expected_type = float

class String(Typed):
    _expected_type = str

class OrderedMeta(type):
    def __new__(cls, clsname, bases, clsdict):
        d = dict(clsdict)
        order = []
        for name, value in clsdict.items():
            if isinstance(value, Typed):
                value._name = name
                order.append(name)
        d['_order'] = order
        return type.__new__(cls, clsname, bases, d)

    @classmethod
    def __prepare__(cls, clsname, bases):
        return OrderedDict()
```

- This can then be used by methods of the class in various ways.
```python
class OrderedMeta(type):
    def __new__(cls, clsname, bases, clsdict):
        d = dict(clsdict)
        order = []
        for name, value in clsdict.items():
            if isinstance(value, Typed):
                value._name = name
                order.append(name)
        d['_order'] = order
        return type.__new__(cls, clsname, bases, d)

    @classmethod
    def __prepare__(cls, clsname, bases):
        return OrderedDict()

class Structure(metaclass = OrderedMeta):
    def as_csv(self):
        return ','.join(str(getattr(self, name)) for name in self._order)

class Stock(Structure):
    name = String()
    shares = Integer()
    price = Float()
    def __init__(self, name, shares, price):
        self.name = name
        self.shares = shares
        self.price = price

s = Stock('GOOG', 100, 490.1)
print(s.name)
print(s.as_csv())
t = Stock('AAPL', 'a lot', 610.23)
# GOOG
# GOOG,100,490.1
# Traceback (most recent call last):
#   File "F:\Learning\Python\Cookbook\test.py", line 53, in <module>
#     t = Stock('AAPL', 'a lot', 610.23)
#   File "F:\Learning\Python\Cookbook\test.py", line 47, in __init__
#     self.shares = shares
#   File "F:\Learning\Python\Cookbook\test.py", line 10, in __set__
#     raise TypeError(self._name + ' expects ' + str(self._expected_type))
# TypeError: shares expects <class 'int'>
```

- The entire key to this recipe is the `__prepare__()` method, which is defined in the *OrderedMeta* metaclass. This method is invoked immediately at the start of a class definition with the class name and base classes. It must then return a mapping object ot use when processing the class body.
- It's possible to extend this functionality even further if you are willing to make your own dictionary-like objects. For example, consider this variant of the solution that rejects duplicate definitions.
```python
class NoDupOrderedDict(OrderedDict):
    def __init__(self, clsname):
        self.clsname = clsname
        super().__init__()
    def __setitem__(self, name, value):
        if name in self:
            raise TypeError('{} already defined in {}'.format(name, self.clsname))
        super().__setitem__(name, value)

class OrderedMeta(type):
    def __new__(cls, clsname, bases, clsdict):
        d = dict(clsdict)
        d['_order'] = [name for name in clsdict if name[0] != '_']
        return type.__new__(cls, clsname, bases, d)

    @classmethod
    def __prepare__(cls, clsname, bases):
        return NoDupOrderedDict(clsname)

class A(metaclass = OrderedMeta):
    def spam(self):
        pass

    def spam(self):
        pass

# Traceback (most recent call last):
#   File "F:\Learning\Python\Cookbook\test.py", line 41, in <module>
#     class A(metaclass = OrderedMeta):
#   File "F:\Learning\Python\Cookbook\test.py", line 45, in A
#     def spam(self):
#   File "F:\Learning\Python\Cookbook\test.py", line 28, in __setitem__
#     raise TypeError('{} already defined in {}'.format(name, self.clsname))
# TypeError: spam already defined in A
```

## 9.15 Defining a Metaclass That Takes Optional Arguments

- When defining classes, Python allows a metaclass to be specified using the *metaclass* keyword argument in the *class* statement.
```python
class NoDupOrderedDict(OrderedDict):
    def __init__(self, clsname):
        self.clsname = clsname
        super().__init__()
    def __setitem__(self, name, value):
        if name in self:
            raise TypeError('{} already defined in {}'.format(name, self.clsname))
        super().__setitem__(name, value)

class OrderedMeta(type):
    def __new__(cls, clsname, bases, clsdict):
        d = dict(clsdict)
        d['_order'] = [name for name in clsdict if name[0] != '_']
        return type.__new__(cls, clsname, bases, d)

    @classmethod
    def __prepare__(cls, clsname, bases):
        return NoDupOrderedDict(clsname)

class A(metaclass = OrderedMeta):
    def spam(self):
        pass

    def spam(self):
        pass

# Traceback (most recent call last):
#   File "F:\Learning\Python\Cookbook\test.py", line 41, in <module>
#     class A(metaclass = OrderedMeta):
#   File "F:\Learning\Python\Cookbook\test.py", line 45, in A
#     def spam(self):
#   File "F:\Learning\Python\Cookbook\test.py", line 28, in __setitem__
#     raise TypeError('{} already defined in {}'.format(name, self.clsname))
# TypeError: spam already defined in A
```

- However, in custom metaclasses, additional keyword arguments can be supplied, like this
```python
class Spam(metaclass = MyMeta, debug = True, synchronize = True):
    ...
```

- To support such keyword arguments in a metaclass, make sure you define them on the `__prepare__()`, `__new__()`, and `__init__()` methods using keyword-only arguments.
```python
class MyMeta(type):
    @classmethod
    def __prepare__(cls, name, bases, *, debug = False, synchronize = False):
        ...
        return super.__prepare__(name, bases)

    def __new__(cls, name, bases, ns, *, debug = False, synchronize = False):
        ...
        return super.__new__(cls, name, bases, ns)

    def __init__(self, name, bases, ns, *, debug = False, synchronize = False):
        ...
        super.__init__(name, bases, ns)
```

- The `__prepare__()` method is called first and used to create the class namespace prior to the body of any class definition being processed.
- The `__new__()` method is used to instantiate the resulting type object. It is called after the class body has been fully executed.
- The `__init__()` method is called last and used to perform any additional initialization steps.
- When writing metaclasses, it is somewhat common to only define a `__new__()` or `__init__()` method, but not both. However, if extra keyword arguments are going to be accepted, then both methods must be provided and given compatible signatures.

## 9.16 Enforcing an Argument Signature on `*arg` and `*kwargs`

- For any problem where you want to manipulate function calling signatures, you should use the signature features found in the *inspect* module. Two classes, *Signature* and *Parameter*, are of particular interest here.
```python
>>> from inspect import Signature, Parameter
>>> parms = [Parameter('x', Parameter.POSITIONAL_OR_KEYWORD),
          Parameter('y', Parameter.POSITIONAL_OR_KEYWORD, default = 42),
          Parameter('z', Parameter.KEYWORD_ONLY, default = None)]
>>> sig = Signature(parms)
>>> print(sig)
(x, y=42, *, z=None)
```

- Once you have a signature object, you can easily bind it to `*args` and `**kwargs` using the signature's *bind()* method.
```python
>>> def func(*args, **kwargs):
    bound_values = sig.bind(*args, **kwargs)
    for name, value in bound_values.arguments.items():
        print(name, value)

        
>>> func(1, 2, z = 3)
x 1
y 2
z 3
>>> func(1)
x 1
>>> func(1, z = 3)
x 1
z 3
>>> func(y = 2, x = 1)
x 1
y 2
>>> func(1, 2, 3, 4)
Traceback (most recent call last):
  ...
TypeError: too many positional arguments
>>> func(y = 2)
Traceback (most recent call last):
  ...
TypeError: missing a required argument: 'x'
>>> func(1, y = 2, x = 3)
Traceback (most recent call last):
  ...
TypeError: multiple values for argument 'x'
```

- Here is a more concrete example of enforcing function signatures.
```python
from inspect import Signature, Parameter

def make_sig(*names):
    parms = [Parameter(name, Parameter.POSITIONAL_OR_KEYWORD)
             for name in names]
    return Signature(parms)

class Structure:
    __signature__ = make_sig()
    def __init__(self, *args, **kwargs):
        bound_values = self.__signature__.bind(*args, **kwargs)
        for name, value in bound_values.argument.items():
            setattr(self, name, value)

class Stock(Structure):
    __signature__ = make_sig('name', 'shares', 'price')

class Point(Structure):
    __signature__ = make_sig('x', 'y')
```

## 9.17 Enforcing Coding Conventions in Classes

- If you want to monitor the definition of classes, you can often do it by defining a metaclass. A basic metaclass is usually defined by inheriting from *type* and redefining its `__new__()` method or `__init__()` method.
- To use metaclass, you would generally incorporate it into a top-level base class from which other objects inherit.
- A key feature of a metaclass is that it allows you to examine the contents of a class at the time of definition.
- Once a metaclass has been specified for a class, it gets inherited by all of the subclasses.
- Here is a metaclass that checks the definition of redefined methods to make sure they have the same calling signature as the original method in the superclass.
```python
from inspect import signature
import logging

class MatchSignatureMeta(type):
    def __init__(self, clsname, bases, clsdict):
        super().__init__(clsname, bases, clsdict)
        sup = super(self, self)
        for name, value in clsdict.items():
            if name.startswith('_') or not callable(value):
                continue
            prev_dfn = getattr(sup, name, None)
            if prev_dfn:
                prev_sig = signature(prev_dfn)
                val_sig = signature(value)
                if prev_sig != val_sig:
                    logging.warning('Signature mismatch in %s. %s != %s', value.__qualname__, prev_sig, val_sig)

class Root(metaclass = MatchSignatureMeta):
    pass

class A(Root):
    def foo(self, x, y):
        pass
    def spam(self, x, *, z):
        pass

class B(A):
    def foo(self, a, b):
        pass
    def spam(self, x, z):
        pass

# WARNING:root:Signature mismatch in B.foo. (self, x, y) != (self, a, b)
# WARNING:root:Signature mismatch in B.spam. (self, x, *, z) != (self, x, z)
```

- The choice of redefining `__new__()` or `__init__()` in a metaclass depends on how you want to work with the resulting class.
 + `__new__()` is invoked prior to class creation and is typically used when a metaclass wants to alter the class definition in some way (by changing the contents of the class dictionary).
 + The `__init__()` method is invoked after a class has been created, and is useful if you want to write code that works with the fully formed class object.

## 9.18 Defining Classes Programmatically

- You can use the function *types.new_class()* to instantiate new class objects. All you need to do is provide the name of the class, tuple of parent classes, keyword arguments, and callback that populates the class dictionary with members.
```python
def __init__(self, name, shares, price):
    self.name = name
    self.shares = shares
    self.price = price

def cost(self):
     return self.shares * self.price

cls_dict = {
    '__init__': __init__,
    'cost': cost,
}

import types

Stock = types.new_class('Stock', (), {}, lambda ns: ns.update(cls_dict))
Stock.__module__ = __name__

# ======================================
# >>> s = Stock('ACME', 50, 91.1)
# >>> s
# <__main__.Stock object at 0x022E0EB0>
# >>> s.cost()
# 4555.0
```

- Whenever a class is defined, its `__module__` attribute contains the name of the module in which it was defined. This name is used to produce the output made by methods such as `__repr__()`. It's also used by various libraries, such as *pickle*.
- If the class you want to create involves a different metaclass, it would be specified in the thid argument to *types.new_class()*.
```python
>>> import abc
>>> Stock = types.new_class('Stock', (),  {'metaclass': abc.ABCMeta},
            lambda ns: ns.update(cls_dict))
>>> Stock.__module__ = __name__
>>> Stock
<class '__main__.Stock'>
>>> type(Stock)
<class 'abc.ABCMeta'>
```

- The third argument may also contain other keyword arguments.
```python
class Spam(Base, debug = True, typecheck = False):
    ...

Spam = types.new_class('Spam', (Base,),
                        {'debug': True, 'typecheck': False},
                        lambda ns: ns.update(cls_dict))
```

- The fourth argument to *new_class()* is the most mysterious, but it is a function that receives the mapping object being used for the class namespace as input. This is normally a dictionary, but it's actually whatever object gets returned by the `__prepare__()` method.
- Being able to manufacture new class objects can be useful in certain contexts. One of the more familiar examples involves the *collections.namedtuple()* function. *namedtuple()* uses *exec()* instead of the technique shown here.
```python
>>> Stock = collections.namedtuple('Stock', ['name', 'shares', 'price'])
>>> Stock
<class '__main__.Stock'>
```

- If you only want to carry out the preparation step, use *types.prepare_class()*.
```python
import types

metaclass, kwargs, ns = types.prepare_class('Stock', (), {'metaclass': type})
```
