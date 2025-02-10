---
layout: post
author: kormang
---

For a long time coroutines in C++ did not exist as a language-level construct, but thanks to low level nature of C++, it was possible to implement them using some tricks and assembly magic. Later, in C++20, couroutines similar to those in other languages where added.

### Stackful coroutines in C++

The most famous coroutine implementation is from [Boost](https://www.boost.org/) libraries. There is the [older](https://www.boost.org/doc/libs/1_84_0/libs/coroutine/doc/html/index.html), pre C++11 version, and the newer, [coroutine2](https://www.boost.org/doc/libs/1_84_0/libs/coroutine2/doc/html/coroutine2/overview.html) version, that requires C++11 or newer.

Same as the [article](https://greenlet.readthedocs.io/en/latest/gui_example.html#gui-example) about greenlet motivation, [the article on Boost coroutine motivation](https://www.boost.org/doc/libs/1_84_0/libs/coroutine2/doc/html/coroutine2/motivation.html) also serves as a good complementary material to this material, and is **recommended read** to get better understanding about asynchronous programming.

When a function calls another function, the other function's stack frame is pushed onto the call stack. A stack frame contains local variables and the return address (where the function should return). When a function returns, its stack frame is popped from the call stack. These coroutines are **stackful**, which means that the stack is preserved when a coroutine suspends. This enables coroutines to suspend not just from their own stack frame but also from nested stack frames of functions called by the coroutine. You can find plenty of resources about why and how these coroutines work in the [official docs](https://www.boost.org/doc/libs/1_84_0/libs/coroutine2/doc/html/index.html).

As official documentation contains all the necessary details to understand the concepts and provides many examples, we will see just one simple example of coroutine usage, and describe the way it works in concise manner.

We will use example from the official docs, but a bit simplified.

We're going to solve the "same fringe" problem. We say that the two trees have same fringe, if by iterating over the nodes of the two trees in the same way, we visit their leaves in the the same order, as show on the picture.

![Same fringe trees](./images/same_fringe.png "Same fringe trees")

We will make program using coroutines to compare the fringes of the two trees, without producing intermediate data structures, and by reusing generalized `traverse_inorder` function.

```c++
#include <memory>
#include <algorithm>
#include <iostream>
#include <boost/coroutine2/coroutine.hpp>

struct node{
    typedef std::shared_ptr<node> ptr_t;

    ptr_t left;
    ptr_t right;
    char value;

    // Constructor for leaf nodes.
    node(char v):
        left(),right(),value(v)
    {}
    // Constructor for  non-leaf nodes.
    node(char v, ptr_t l, ptr_t r):
        left(l), right(r), value(v)
    {}
};

typedef boost::coroutines2::coroutine<char> coro_t;

// recursively walk the tree, delivering values in order
void traverse_inorder(node::ptr_t n,
              coro_t::push_type& yield){
    if(n->left) traverse_inorder(n->left, yield);
    yield(n->value);
    if(n->right) traverse_inorder(n->right, yield);
}

int main() {
    // Create two tree as on the picture above.
    auto tree1 = std::make_shared<node>(
        'd',
        std::make_shared<node>(
            'b',
            std::make_shared<node>('a'),
            std::make_shared<node>('c')
        ),
        std::make_shared<node>('e')
    );

    auto tree2 = std::make_shared<node>(
        'b',
        std::make_shared<node>('a'),
        std::make_shared<node>(
            'd',
            std::make_shared<node>('c'),
            std::make_shared<node>('e')
        )
    );

    auto lambda_func_1 = [tree1](coro_t::push_type& yield) {
        std::cout << "Starting first traversal...\n";
        traverse_inorder(tree1, yield);
    };
    coro_t::pull_type g1(std::move(lambda_func_1));

    coro_t::pull_type g2([tree2](coro_t::push_type& yield) {
        std::cout << "Starting second traversal...\n";
        traverse_inorder(tree2, yield);
    });

    std::cout << "Comparing tree leaves...\n";
    while (g1 && g2) {
        std::cout << g1.get() << " == " << g2.get() << std::endl;
        if (g1.get() != g2.get()) {
            break;
        }
        // Advance both.
        g1();
        g2();
    }

    // Did they both got to the end?
    bool result = !g1 && !g2;

    std::cout << (result ? "they have the same fringe" : "they don't have the same fringe") << std::endl;

    return 0;
}

```

This code produces the following output.

```
Starting first traversal...
Starting second traversal...
Comparing tree leaves...
a == a
b == b
c == c
d == d
e == e
they have the same fringe
```

Coroutines can also be used with iterators so we could have written code like this.

```c++
    for (char c : g1) {
        std::cout << c;
    }
    std::cout << std::endl;

    for (auto it = begin(g2), eit = end(g2); it != eit; ++it) {
        std::cout << *it;
    }
    std::cout << std::endl;
```

Which would produce following output.

```
abcde
abcde
```

As we can see boost coroutines have a lot in common with Python generators, but are stackful, and can yield from nested functions.

The way it works is this, when the `yield` functor in our example (of type `coro_t::push_type`) is invoked it saves the context of the current coroutine, similar to how OS saves the context of the current thread when suspending thread. It is done by using [callcc](https://www.boost.org/doc/libs/1_84_0/libs/context/doc/html/context/cc.html#implementation) function of the [context](https://www.boost.org/doc/libs/1_84_0/libs/context/doc/html/index.html) library from boost. It has to save the return address (probably by inspecting call stack), and stack pointer. Then it has to switch to another stack (on amd64 architecture it requires saving and changing value of the RSP register). It probably saves the state of few other registers like RBP, but that depends on the calling convention used by the compiler, and of course the CPU architecture.

### Used for web frameworks in C++, Java, Golang, etc

It should not be too difficult to imagine how this can be used to implement, for example web framework, that serves each request in its own coroutine, similar to how traditional web servers used to server each request in its own thread. When coroutine eventually calls some low level function to perform IO (those functions should belong to the framework), those function could yield control to the framework's scheduler, which would leave coroutine suspended until IO operation completes, and by that time it will execute other coroutines.

Example of one such server that uses boost coroutine is [userver](https://github.com/userver-framework/userver/tree/develop/third_party/uboost_coro). Here is one [example](https://userver.tech/d7/d08/md_en_2userver_2intro__io__bound__coro.html) of how one request handler that is executed inside coroutine, looks like.

```c++
Response View::Handle(Request&& request, const Dependencies& dependencies) {
  auto cluster = dependencies.pg->GetCluster();                             // 🚀
  auto trx = cluster->Begin(storages::postgres::ClusterHostType::kMaster);  // 🚀

  const char* statement = "SELECT ok, baz FROM some WHERE id = $1 LIMIT 1";
  auto row = psql::Execute(trx, statement, request.id)[0];                  // 🚀
  if (!row["ok"].As<bool>()) {
    LOG_DEBUG() << request.id << " is not OK of "
                << GetSomeInfoFromDb();                                     // 🚀
    return Response400();
  }

  psql::Execute(trx, queries::kUpdateRules, request.foo, request.bar);      // 🚀
  trx.Commit();                                                             // 🚀

  return Response200{row["baz"].As<std::string>()};
}

```

The programmer doesn't even have to know about coroutines, it looks like regular code with threads, but in lines marked with 🚀, coroutine can be suspended, and another coroutine could run in its place. This is similar to what would OS do, but this is done more efficiently in user space. This model fits nicely in web servers, and is often chosen for that purpose. For example, `goroutines` in programming language Go, and `Virtual Threads` in Java, work in similar way, without any `async` or `await` keywords, using user space cooperative multitasking and scheduling.

### Simple implementation example

To make it clearer how this works, let's take a look at an example. We will create a simple stackful coroutine system with two coroutines. When one gets suspended, the other one continues execution, and so on. This should help illustrate how we can use a similar mechanism to suspend a coroutine until a timer expires (for a sleep operation) or until an I/O operation is ready.

To make it work, we will limit ourselves to the amd64 architecture, as we need a bit of assembly. We will use GASM (AT&T) assembly for the amd64 ISA. We need one function that will accept a pointer to a structure that holds the context (state of the CPU) that we want to store and another pointer to the context that we want to switch to. This function will store the current state of the CPU and switch to the new one.

We have following C code (here we use pure C, not C++).

```c
// stackful.c

#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

// This will be our structure that holds the register values.
// In other words this is the state of a CPU core with amd64 architecture.
typedef struct {
    uint64_t rbx;
    uint64_t rsp;
    uint64_t rbp;
    uint64_t r12;
    uint64_t r13;
    uint64_t r14;
    uint64_t r15;
    uint64_t rip;
} context_t;

// Declaration of function which is implemented in assembly, so that we can call
// it from C. Take a look at assembly implementation below.
extern void _switch_context(context_t *src, context_t* dst);


// This function stores the current context to src,
// and switches to context that is already stored in dst.
// Real work is done in _switch_context, which is implemented in assembly.
void switch_context(context_t *src, context_t* dst) {
    _switch_context(src, dst);
}

// We will create to stacks for two stackful coroutines.
void* stack0[1000];
void* stack0_base = &stack0[999];
void* stack1[1000];
void* stack1_base = &stack1[999];

// Here we will keep the context of our coroutines.
context_t coro0_context;
context_t coro1_context;
context_t* volatile current_context;

// This function is suspending current coroutine and picks next
// coroutine to continue execution.
// Since in our simple case we only have two coroutines it will
// always pick non-current to continue executing next.
// Hopefully, you can imagine that this mechanism can used to build
// more complex schedulers that suspend coroutines until
// timer expires, or until IO operation is ready.
// To make things simple we will just switch between the two coroutines.
void coro_suspend_and_yield_to_other() {
    // Very advanced scheduling algorithm :)
    context_t* new_context = current_context == &coro0_context
        ? &coro1_context
        : &coro0_context;

    context_t* prev_conext = current_context;
    current_context = new_context;
    switch_context(prev_conext, new_context);
}


// Here we have two functions to illustrate that at any point in the
// call stack stackful coroutine can suspend, and its stack will be
// preserved.
void func1() {
    printf("Enter func1\n");
    coro_suspend_and_yield_to_other();
    printf("Exit func1\n");
}

void func2() {
    printf("Enter func2\n");
    coro_suspend_and_yield_to_other();
    func1();
    printf("Exit func2\n");
}


// Here are the two main functions for the two coroutines.
void coro0() {
    int i = 0;
    while (1) {
        ++i;
        if (i == 3) {
            // Terminate the whole program after 2 iterations.
            exit(0);
        }

        printf("coro0 before func1\n");
        func1();
        printf("coro0 after func1\n");
        coro_suspend_and_yield_to_other();
        printf("coro0 after suspend\n");
    }
}

void coro1() {
    while (1) {
        printf("coro1 before func\n");
        func2();
        printf("coro1 after func\n");
        coro_suspend_and_yield_to_other();
        printf("coro1 after suspend\n");
    }
}

int main() {
    // This is needed just as a parameter to switch_context.
    context_t dummy_context;

    // Here, we prepare initial state of the coroutines,
    // initial stack and addresses of the starting instruction that coroutines
    // should execute (which are equal to the beginning of their
    // main functions, which is equal to the address of the function).
    context_t context0 = {
        .rbx = 0,
        .rsp = (uint64_t)stack0_base,
        .rbp = (uint64_t)stack0_base,
        .r12 = 0,
        .r13 = 0,
        .r14 = 0,
        .r15 = 0,
        .rip = (uint64_t)&coro0
    };
    coro0_context = context0;

    context_t context1 = {
        .rbx = 0,
        .rsp = (uint64_t)stack1_base,
        .rbp = (uint64_t)stack1_base,
        .r12 = 0,
        .r13 = 0,
        .r14 = 0,
        .r15 = 0,
        .rip = (uint64_t)&coro1
    };
    coro1_context = context1;

    current_context = &coro0_context;

    // Jump straight into coro0.
    switch_context(&dummy_context, &coro0_context);

    // This will never be reached!
    return 0;
}

```

And here is short and quite simple assembly implementation of _switch_context.

```
# asm.S

.global _switch_context
_switch_context:
# Save register values to structure pointed to by first argument.
# First argument in C is stored in rdi register.
movq %rbx, 0(%rdi)
movq %rsp, 8(%rdi)
movq %rbp, 16(%rdi)
movq %r12, 24(%rdi)
movq %r13, 32(%rdi)
movq %r14, 40(%rdi)
movq %r15, 48(%rdi)

# Now replace register values by values store in structure pointer to by second argument.
# Second argument in C is stored in rsi register.
movq 0(%rsi), %rbx
movq 8(%rsi), %rsp
movq 16(%rsi), %rbp
movq 24(%rsi), %r12
movq 32(%rsi), %r13
movq 40(%rsi), %r14
movq 48(%rsi), %r15

lea __ret_switch_context(%rip), %rcx
movq %rcx, 56(%rdi)

jmp *56(%rsi)

__ret_switch_context:
ret
```


If we compile it with `gcc -g stackful.c asm.S -o a.out` and run it with `./a.out`, we get the following output.

```
coro0 before func1
Enter func1
coro1 before func
Enter func2
Exit func1
coro0 after func1
Enter func1
coro0 after suspend
coro0 before func1
Enter func1
Exit func1
Exit func2
coro1 after func
Exit func1
coro0 after func1
coro1 after suspend
coro1 before func
Enter func2
coro0 after suspend

```

As we can see, two coroutines were constantly switching between one another.

### How it works in practice

What we saw here is similar to what OS is doing with threads and processes, but here we do that in user space.

In practice functions like `coro_suspend_and_yield_to_other` are called from hypothetical functions like `read_from_network_socket` when there are no data to be read and coroutine has to wait for data to be received. By that time, another coroutine can be scheduled to run in its place. Scheduler can use polling, and non-blocking OS API to implement it. More info about polling and non-blocking APIs can be found here [here](/2025/01/16/AC-part3-how-event-loops-work.html).

To give you an idea how that works with non-blocking IO, using event loop and stackless coroutines in Python, take a look at this post - [How Python coroutines work](/2025/01/25/AC-part7-how-to-write-your-own-event-loop-for-python-async-funcs.html).

To give you an idea how OS handles blocking IO with threads take a look at this post - [Layers of Abstraction](/2025/01/12/AC-part2-layers-of-abstraction.html).

In reality, coroutines in high performance environments are ran by multiple threads, each thread executes single coroutine at a time, but as multiple threads run simultaneously, multiple coroutines can run in parallel. When one thread has no more coroutines to run (all of them might be suspended waiting for IO operation or sleeping), that thread can try and take task/coroutine from task queue that belongs to another thread (work stealing).

Next, we will see how stackless coroutines work in C++, more precisely async functions of C++20.

After that we will take a quick look at coroutines in other languages like Go, Kotlin and Java.
