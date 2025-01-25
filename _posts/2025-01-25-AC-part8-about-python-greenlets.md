---
layout: post
author: kormang
---


## Greenlets

We have seen how cooperative multitasking and coroutines could be implemented using event loop and generator based coroutines. However they have significant limitations. They are hard to use for IO operations inside GUI event loops, particularly if those are run on top of C/C++ event loops. That was the [motivation](https://greenlet.readthedocs.io/en/latest/gui_example.html#gui-example) to implement [greenlets](https://greenlet.readthedocs.io/en/latest/greenlet.html).

Here is one example from the official web site.

```python
from greenlet import greenlet

def test1():
    print("[gr1] main  -> test1")
    gr2.switch()
    print("[gr1] test1 <- test2")
    return 'test1 done'

def test2():
    print("[gr2] test1 -> test2")
    gr1.switch()
    print("This is never printed.")

gr1 = greenlet(test1)
gr2 = greenlet(test2)
print(gr1.switch())
print(gr1.dead)
print(gr2.dead)
```

That code will produce following output.

```
[gr1] main  -> test1
[gr2] test1 -> test2
[gr1] test1 <- test2
'test1 done'
True
False
```

As we can see, greenlets to not have any event loop based, or other kind of scheduler that would resume certain greenlets when current one suspends. Instead, with greenlet we explicitly choose to which greenlets to transfer the control to. This is similar to `yield from` or calling `generator.send`, but with one important difference. Greenlets can suspend from any function, not just generator/coroutine, whatever the call stack may look like at that moment, even if there are C-functions on the stack.

That makes greenlets suitable for running on top of C/C++ GUI event loops, and can also be used to bridge the gap between sync and async functions. [The article explaining the motivation](https://greenlet.readthedocs.io/en/latest/gui_example.html#gui-example) behind it, also serves as a good complementary material to this material, and is **recommended read** to get better understanding of asynchronous programming.

As we know, async functions can only be called from other async functions. That makes it harder to create libraries reusable from both blocking/threaded and asynchronous applications. One such example, is the Alchemy, library for high level SQL database interaction. Basically what it does is this user calls something like this:

```python
user = users.findById(1)
```

What it would to is, create SQL query like `SELECT * FROM users WHERE id = 1`, and they call the DB driver to execute the query.

```
+------------------------------------+
| func from DB driver                |
|------------------------------------|
| func from core that prepares query |
|------------------------------------|
| func the user calls (e.g findById) |
+------------------------------------+

```

But if we want to use it in an async manner, then driver is async, and the function the user calls should be async. However, we would like to keep the core of the library that prepares the query sync, as to not duplicate it. We want this:

```
+------------------------------------------+
| async func from DB driver                |
|------------------------------------------|
| SYNC func from core that prepares query  |
|------------------------------------------|
| async func the user calls (e.g findById) |
+------------------------------------------+
```

Things like that are also possible to accomplish using greenlets, [see](https://habr.com/ru/articles/767532/).

To understand what greenlets do it is useful to read [the corresponding answers](https://stackoverflow.com/questions/3349048/how-do-greenlets-work) on stack overflow, and if more details are needed, the linked resources, namely the [source code full of comments](https://github.com/python-greenlet/greenlet/blob/master/src/greenlet/greenlet.cpp#L107), or [the author's letter](http://www.stackless.com/pipermail/stackless-dev/2004-March/000022.html).

We will not go into details about how it works, as it changes over time, but the idea is that with access to lower level primitives (which is in case of Python, the C-API, and in the case of C, the assembly), we can do manipulations otherwise inaccessible from a higher level languages. For example, we can save the state of current call stack on suspend, and restore it later on resume.
