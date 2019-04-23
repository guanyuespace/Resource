---
title: 多层解释器的执行顺序
date: 2019-04-23 10:57:23
---
<!-- TOC -->

- [python装饰器执行顺序](#python%E8%A3%85%E9%A5%B0%E5%99%A8%E6%89%A7%E8%A1%8C%E9%A1%BA%E5%BA%8F)

<!-- /TOC -->
# python装饰器执行顺序
> [原文地址](https://blog.csdn.net/u010472499/article/details/72233571)

探究多个装饰器执行顺序
**疑问**
大部分涉及多个装饰器装饰的函数调用顺序时都会说明它们是自上而下的，比如下面这个例子:

```python
def decorator_a(func):
    print 'Get in decorator_a'
    def inner_a(*args, **kwargs):
        print 'Get in inner_a'
        return func(*args, **kwargs)
    return inner_a

def decorator_b(func):
    print 'Get in decorator_b'
    def inner_b(*args, **kwargs):
        print 'Get in inner_b'
        return func(*args, **kwargs)
    return inner_b

@decorator_b
@decorator_a
def f(x):
    print 'Get in f'
    return x * 2
```

调用：f(1)
上面代码先定义里两个函数: `decotator_a`, `decotator_b`, 这两个函数实现的功能是，接收一个函数作为参数然后返回创建的另一个函数，在这个创建的函数里调用接收的函数 (文字比代码绕人)。最后定义的函数 f 采用上面定义的 `decotator_a`, `decotator_b` 作为装饰函数。在当我们以 1 为参数调用装饰后的函数 `f` 后， `decotator_a`, `decotator_b` 的顺序是什么呢（这里为了表示函数执行的先后顺序，采用打印输出的方式来查看函数的执行顺序）？

如果不假思索根据自下而上的原则来判断地话，先执行 `decorator_a` 再执行 `decorator_b` , 那么会先输出 Get in decotator_a, Get in inner_a 再输出 Get in decotator_b , Get in inner_b 。然而事实并非如此。

实际上运行的结果如下:

```
Get in decorator_a
Get in decorator_b
Get in inner_b
Get in inner_a
Get in f
```

**函数和函数调用的区别**
为什么是先执行 `inner_b` 再执行 `inner_a` 呢？为了彻底看清上面的问题，得先分清两个概念: 函数和函数调用。上面的例子中 `f` 称之为函数， `f(1)` 称之为函数调用，后者是对前者传入参数进行求值的结果。    
在 `Python` 中函数也是一个对象，所以 `f` 是指代一个函数对象，它的值是函数本身， `f(1)` 是对函数的调用，它的值是调用的结果，这里的定义下 `f(1)` 的值 2。    
同样地，拿上面的 `decorator_a` 函数来说，它返回的是个函数对象 `inner_a` ，这个函数对象是它内部定义的。在 `inner_a` 里调用了函数 `func` ，将 `func` 的调用结果作为值返回。    

**装饰器函数在被装饰函数定义好后立即执行**
其次得理清的一个问题是，当装饰器装饰一个函数时，究竟发生了什么。现在简化我们的例子，假设是下面这样的:

```python
def decorator_a(func):
    print 'Get in decorator_a'
    def inner_a(*args, **kwargs):
        print 'Get in inner_a'
        return func(*args, **kwargs)
    return inner_a

@decorator_a
def f(x):
    print 'Get in f'
    return x * 2
# f=decorater_a(f)
```

当解释器执行这段代码时， `decorator_a` 已经调用了，它以函数 `f` 作为参数， 返回它内部生成的一个函数，所以此后 `f` 指代的是 `decorater_a` 里面返回的 `inner_a` 。所以当以后调用 `f` 时，实际上相当于调用 `inner_a` , 传给 `f` 的参数会传给 `inner_a` , 在调用 `inner_a` 时会把接收到的参数传给 `inner_a` 里的 `func` 即 `f` , 最后返回的是 `f` 调用的值，所以在最外面看起来就像直接再调用 `f` 一样。

疑问的解释
当理清上面两方面概念时，就可以清楚地看清最原始的例子中发生了什么。     
**当解释器执行下面这段代码时，实际上按照从下到上的顺序已经依次调用了 decorator_a 和 decorator_b ，这是会输出对应的 Get in decorator_a 和 Get in decorator_b 。**  这时候 `f` 已经相当于 `decorator_b` 里的 `inner_b` 。但因为 `f` 并没有被调用，所以 `inner_b` 并没有调用，依次类推 `inner_b` 内部的 `inner_a` 也没有调用，所以 Get in inner_a 和 Get in inner_b 也不会被输出。
<!-- 相当于多层封装,ok!!! -->

```python
@decorator_b
@decorator_a
def f(x):
    print 'Get in f'
    return x * 2
```

<p style="background-color:cyan;filter:brightness(0.8);">然后最后一行当我们对 `f` 传入参数 1 进行调用时， `inner_b` 被调用了，它会先打印 Get in inner_b ，然后在 inner_b 内部调用了 `inner_a` 所以会再打印 Get in inner_a, 然后再 `inner_a` 内部调用的原来的 `f`, 并且将结果作为最终的返回。这时候你该知道为什么输出结果会是那样，以及对装饰器执行顺序实际发生了什么有一定了解了吧。</p><br/>
当我们在上面的例子最后一行 f 的调用去掉，放到 repl 里演示，也能很自然地看出顺序问题:

```shell
➜  test git:(master) ✗ python
Python 2.7.11 (default, Jan 22 2016, 08:29:18)
[GCC 4.2.1 Compatible Apple LLVM 7.0.2 (clang-700.1.81)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import test13
Get in decorator_a
Get in decorator_b
>>> test13.f(1)
Get in inner_b
Get in inner_a
Get in f
2
>>> test13.f(2)
Get in inner_b
Get in inner_a
Get in f
4
>>>
```

在实际应用的场景中，当我们采用上面的方式写了两个装饰方法比如先验证有没有登录 @login_required ， 再验证权限够不够时 @permision_allowed 时，我们采用下面的顺序来装饰函数:

```py
@login_required
@permision_allowed
def f()
  # Do something
  what="result"
  return what
```

引自:[Python 装饰器执行顺序迷思 - Python 提高班 - SegmentFault 思否](https://segmentfault.com/a/1190000007837364)
