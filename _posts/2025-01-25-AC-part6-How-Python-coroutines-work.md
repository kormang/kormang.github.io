
## Coroutines in Python

Similar to JavaScript, Python has concept of iterable and iterator, that are basically the same but are implemented a bit differently.

### Iterators and iterables

First, we will take a look at iterators and iterables, but this iterators are a bit special, as they have additional `send` method.

```python
class Iterator:
    def __init__(self):
        self.state = 0

    def send(self, input):
        if self.state == 0:
            self.state += 1
            return "one"
        elif self.state == 1:
            self.state += 1
            if input:
                return "two"
            else:
                # Go to state 2.
                return self._step(None)
        elif self.state == 2:
            self.state += 1
            return "three"
        elif self.state == 3:
            self.state += 1
            raise StopIteration("four")
        elif self.state == 4:
            raise StopIteration

    def __next__(self):
        # This special kind of iterator has `send` method,
        # and the next method is just alias for `send(None)`.
        return self.send(None)

    def __iter__(self):
        return self


class Iterable:
    def __iter__(self):
        return Iterator()


iterator = Iterator()

print(next(iterator))
print(iterator.send(True))
print(next(iterator))
try:
    print(next(iterator))
except StopIteration as si:
    print(si)

print("Now, let's use it through iterable and with for loop.")

iterable = Iterable()

for j in iterable:
    print(j)

```

This code produces the following output:

```
one
two
three
four
Now, let's use it through iterable and with for loop.
one
three
```

### Generators

Now, let's see the same code using generator.

```python

# This is generator.
# More precisely, this is generator function.
# Generator functions, return generator objects, we call
# iterator below. But since they also have `send` method
# they are more than iterators, which have only __next__.
def create_iterator():
    input = yield "one"
    if input:
        yield "two"
    yield "three"
    return "four"


iterator = create_iterator()

print(next(iterator))
print(iterator.send(True))
print(next(iterator))
try:
    print(next(iterator))
except StopIteration as si:
    print(si)

print("Now, let's use it through iterable and with for loop.")

iterable = create_iterator()

for j in iterable:
    print(j)

```

This code produces exactly the same output as the above code.

It would be fair to say that our iterator is not just iterator. It also has the `send` method. This is not really about iterating over some collection of values, it is more about producing and consuming data dynamically, and it is about state of execution, and control flow.

Likewise, our generator is not just generator, it does not just generate values. It also consumes values. When ever it produces value, it suspends, the caller gains control back, and when the caller calls `send` it resumes the generator. This makes the generator a coroutine.

We would like our coroutines to call other coroutines, so we can factor out some parts of the code to sub-coroutines.

Let's start with our current coroutine.

```python
# This function creates coroutine object.
# The coroutine object has `__next__` and `send` methods.
def create_coroutine():
    input = yield "one"
    if input:
        yield "two"
    yield "three"
    return "four"


coro = create_coroutine()

print(next(coro))
print(coro.send(True))
print(next(coro))
try:
    print(next(coro))
except StopIteration as si:
    print(si)

```

Now we can extract first part of the coroutine into another subroutine.

```python
def subroutine():
    input = yield "one"
    if input:
        yield "two"


def create_coroutine():
    subcoro = subroutine()
    output = next(subcoro) # Equivalent to subcoro.send(None)
    input = yield output
    yield subcoro.send(input)
    yield "three"
    return "four"

```

### Generator based coroutines

This takes a lot of manual work, and it will not work if we add more yield statements in `subroutine`. To make it a bit more generalized, we can write it in the following way.

```python
def subroutine():
    input = yield "one"
    if input:
        yield "two"
    yield "two.one"


def create_coroutine():
    subcoro = subroutine()
    output = subcoro.send(None) # next(subcoro) does the same.
    input = yield output
    while True:
        try:
            input = yield subcoro.send(input)
        except StopIteration:
            break
    yield "three"
    return "four"
```

We will refer to the example as 'manual sub-coroutine calling' later in this text.

This method involves a significant amount of manual work and is prone to errors. In fact, we have omitted many important details. Fortunately, there is a shortcut for writing code in this manner: the `yield from` statement.

```python
def subroutine():
    input = yield "one"
    if input:
        yield "two"
    yield "two.one"


def create_coroutine():
    yield from subroutine()
    yield "three"
    return "four"

```

This code is not exactly equivalent; it takes care of many more things, such as error propagation and handling, among others. The interpreter can even optimize this code to avoid going through the `create_coroutine` coroutine when calling methods of the `subroutine`. This is similar to how async coroutines in JavaScript work; they are actually generators that return a Promise object. After the Promise is fulfilled, the outer system will call back into the generator. Similarly, here the interpreter can do the same thing under the hood: it can communicate directly with the `subroutine` object until it finishes, and then call back into the `create_coroutine` object. It can also pass the final result of the `subroutine` to the outer coroutine.

We have already seen that functions are objects with a `__call__` method. We also know that when functions call other functions, the CPU or interpreter has to push a new stack frame onto the call stack. When the called function returns, its stack frame is popped from the stack, and control is returned to the caller function along with the return value. A similar thing can be done with generator functions, but instead of `__call__`, they have `send`, and it can be called multiple times.

```python
def subroutine():
    input = yield "one"
    if input:
        yield "two"
    yield "two.one"
    return "result of the subroutine"


def create_coroutine():
    subresult = yield from subroutine()
    print("subresult=", subresult)
    yield "three"
    return "four"


coro = create_coroutine()

print(next(coro))
print(coro.send(True))
print(next(coro))
print(next(coro))
try:
    print(next(coro))
except StopIteration as si:
    print(si)

```

This code produces the following output:

```
one
two
two.one
subresult= result of the subroutine
three
four
```


Let's take a look at another, more complex example.
We will have coroutine `d` that `yields from` coroutine `c`, that yields from coroutine `b`, that yields from coroutines `a`, and then `a1`.

```python
def a():
    sum = 0
    while sum < 100:
        sum += yield sum
        print("Sum in a is", sum)

    return sum

def a1():
  yield -1
  yield -2
  return -3

def b():
    # yield from will forward all `send` calls, from outside to `a`.
    # But StopIteration will be caught by `b`
    # to get the final result of `a`.
    a_result = yield from a()
    print('After yield from a()')
    a1_result = yield from a1()
    print('After yield from a1()')
    return a_result + a1_result


def c():
    result = yield from b()
    print("After yield from b()")
    return result

def d():
    return (yield from c())


coro = d()

coro.send(None)
print(coro.send(1))
print(coro.send(1))
print(coro.send(99))
print(coro.send(1))
try:
    coro.send(1)
except StopIteration as si:
    print("Result:", si.value)

```

In this example, we don't really want all calls to `coro.send` to be sent first to `d`, then from `d` to `c`, then from `c` to `b`, and finally from `b` to `a`. Instead, behind the scene, for the `yield from` syntax, the compiler produces special instructions (bytecode for CPython, which is not limited by the same limitation real hardware is) that send messages directly to `a`. It also pushes `c` and `b` to the stack similar to regular function calls, so that it is possible to resume them later in correct order, for example to execute `print('After yield from a()')`, similar to how regular functions resume their execution, only after callee function is completely done. Regular functions return after being called, similarly generators (coroutines) get exhausted and return. So when the outside caller calls `coro.send`, it works as if all those messages are passed down the chain, but the implementation is optimized, messages are sent directly to `a` until it is exhausted, after that, `a` is popped from stack and we continue sending messages directly to `b`, which puts `a1` to the stack, after that messages are being sent to `a1` directly.

The best way to understand what is going on is to look at the output of the above code:

```
Sum in a is 1
1
Sum in a is 2
2
Sum in a is 101
After yield from a()
-1
-2
After yield from a1()
After yield from b()
Result: 98
```

So we can think of it as syntactic sugar for "manual sub-coroutine calling" (because it really does automate that process, literally). But depending on the internal implementation, we can think of it as putting coroutine objects on the stack instead of function objects and calling `send` repeatedly instead of calling `__call__` once. Think of `yield from` as making function call, and exhausting generator as equivalent to return from function.


Let's **summarize** what we have so far. We have seen that generator functions return generator objects, which are similar to iterators, but with additional `send` method to support to way data flow. Such generators are essentially coroutines. Coroutines can pass control flow into another coroutine, and to avoid tedious, error prone manual passing, we can use `yield from`, a much more robust and optimized way, and also shorter and easier to use.

Next, we will see how to create even loop that can run such generators to do async IO.
Then we will see what are async functions actually.
