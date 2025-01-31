
---
layout: post
author: kormang
---


Coroutines were introduced in C++20 as a language-level construct. These coroutines are more similar to Python coroutines than the boost ones. They use special syntax (any function that uses the `co_await`, `co_yield`, or `co_return` keywords is a coroutine) and are stackless. They are also based on coroutine handles (similar to generator objects) and other notions such as promises and awaitables. There are a huge number of rules on how coroutines should behave in certain situations and what the code the compiler generates will call and do, in what order, and so on. This is a huge topic, and it is partially covered in the documentation on [cppreference](https://en.cppreference.com/w/cpp/language/coroutines).

While in Python, generator functions return generator objects, in C++, coroutines return a custom object. Those can be any object; the programmer can, and in fact, must define that return type. That type has to fulfill some requirements as described in short terms [here in the cppreference docs](https://en.cppreference.com/w/cpp/language/coroutines). The most important requirement is that it has to have a member `promise_type` that represents the promise, a type through which the coroutine communicates with the outside world, the caller (yields values or returns values). Callers use return objects, which can use handles to communicate with the coroutine (suspend, resume, extract yielded or returned values). If the returned object is also awaitable, it can be used in a `co_await` expression. Awaitables can, but don't have to have `operator co_await` defined (similar to how in Python, a type has to have the magic method `__await__`), which produces an "awaiter" object, or fulfills some of the other few conditions, like being an awaiter object itself, which requires it to have functions `await_ready`, `await_suspend`, and `await_resume`.

There are many more rules regarding coroutines and their implementations, and it is much more complicated than in Python. All those rules and complexity serve the purpose of allowing programmers more freedom, and by picking a particular `handle` and corresponding `promise_type`, the programmer can choose how the coroutine should behave and how it can and should be used. It is possible to define how it should suspend, what happens when it resumes, how it yields and returns values, and so on. That enables programmers to transparently suspend a coroutine on one thread and resume it on another, and so on. It is accomplished by defining particular functions on those types, like `get_return_object`, `initial_suspend`, `return_value`, `yield_value` (for example, `yield expr` compiles to `co_await promise.yield_value(expr)`), and so on. One can reuse certain return types for multiple coroutines. By using different return types, we produce coroutines with different behavior. For example, to make generator-like coroutines that yield an infinite sequence of values, we can use `std::generator` or write our own return type (actually a handle-wrapper) like the one shown in the cppreference example.

Since we're here to understand concepts of async programming, and not the details of a particular language, we will not go into all those details. There are documentation, standards, and other sources of more detailed information, so soon we will just look at an example of how coroutines can be used to perform asynchronous IO.

An important difference between stackful coroutines (e.g., Boost coroutine2) and stackless coroutines (introduced in C++20) is how they preserve state between suspend and resume operations. Stackful coroutines are suspended by a library function that saves the return address as the current suspension point of the coroutine and the CPU's stack pointer register value as a pointer to the coroutine's stack, holding all local variables. Stackless coroutines save the state of the coroutine after `awaiter.await_ready()` returns `false` and before a call to `awaiter.await_suspend(handle)`. The state is saved to the dynamically allocated storage that contains the suspension point (it can be the address of the next instruction to execute, so far very similar to stackful coroutines), and local variables are copied from the stack. Only variables local to that particular coroutine, while stackful coroutines preserve the whole stack, but actually just a pointer to it.

```c++
#include <coroutine>
#include <iostream>
#include <stdexcept>
#include <thread>
#include <vector>
#include <chrono>
#include <sys/epoll.h>
#include <fcntl.h>
#include <unistd.h>
#include <queue>

struct Scheduler;

// Helper function to make file descriptor use non-blocking IO.
// In such case if we request 100 bytes from file descriptor,
// and there are none available it will return 0 bytes immediately,
// instead of waiting to fill requested 100 bytes.
// Linux system call `fcntl` is used to accomplish that.
void setnonblocking(int fd) {
    int flags;

    // Get current flags for the file descriptor
    flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) {
        perror("fcntl");
        return;
    }

    // Set the non-blocking flag
    flags |= O_NONBLOCK;
    if (fcntl(fd, F_SETFL, flags) == -1) {
        perror("fcntl");
        return;
    }
}

// This is helper function that uses std::chrono
// to get current time since epoch in milliseconds.
int64_t epoch_time_ms() {
    auto now = std::chrono::system_clock::now();
    auto epoch = now.time_since_epoch();
    auto value = std::chrono::duration_cast<std::chrono::milliseconds>(epoch);
    return value.count();
}


// This is awaitable returned by `Promise::final_suspend` below.
// Standard says that `Promise::final_suspend` is called by
// compiler-generated code when current coroutine that the promise
// belongs to is done.
// Then compiler-generated code calls `await_suspend`
// on it (actually after calling `await_ready`),
// and if `await_suspend` returns another coroutine handle,
// that coroutine will be resumed by compiler-generated code.
// This is the way we decide what should run next after
// certain coroutine finishes and returns.
// Usually you don't want to think about all of this,
// so it is job of library or a framework to implement
// classes like this one.
template<typename PromiseType>
struct FinalSuspendAwaitable {
    bool await_ready() noexcept { return false; }

    std::coroutine_handle<> await_suspend(std::coroutine_handle<PromiseType> h) noexcept {
        if (!h.promise().continuation) {
            return std::noop_coroutine();
        }
        return h.promise().continuation;
    }

    void await_resume() noexcept {}
};

// This represents a promise associated with a coroutine.
// This template is supposed to work for
// coroutines that return values.
// There is specialization below that has some other
// functions in order to work for coroutines that return nothing (void).
// First three members are the same for both.
template<typename Task, typename T>
struct Promise {
    // These first three members are same for
    // this generic promise that returns value,
    // and they are also present in specialization for void below.
    std::coroutine_handle<> continuation;
    std::suspend_always initial_suspend() noexcept {
        return {};
    }
    void unhandled_exception() {}

    // This is specific for value-returing coroutine.
    // Although it looks like the same function in void specialization,
    // it is not since it uses different `Promise`
    // template parameter for `FinalSuspendAwaitable`.
    auto final_suspend() noexcept {
        // This is the rule:
        // This function is called when coroutine is finished,
        // and it is returning.
        // It may return an awaitable.
        // If the returned awaitable has function called 'await_suspend'
        // and that function returns another coroutine,
        // that coroutine will be resumed.
        // To resume that coroutine,
        // the compiler generates code which calls 'resume'.
        // So this is analogous to normal return in normal functions,
        // we decide where we return to, but since we've saved the caller,
        // we will return to our caller.

        return FinalSuspendAwaitable<Promise>{};
    }

    // This is also specific for value-returning promise
    // only because it is passing
    // different `Promise` to `std::coroutine_handle`.
    Task get_return_object() {
        return Task{
            std::coroutine_handle<Promise>::from_promise(*this)
        };
    }

    // These too members are real difference between this template class,
    // and the one below. Those are needed for coroutine to return value.
    // Specialization for void does not have these members.

    T result;

    void return_value(T t) {
        result = std::move(t);
    }
};


// This is specialization for void coroutines, that do not return values.
template<typename Task>
struct Promise<Task, void> {
    std::coroutine_handle<> continuation;
    std::suspend_always initial_suspend() noexcept {
        return {};
    }
    void unhandled_exception() {}

    auto final_suspend() noexcept {
        return FinalSuspendAwaitable<Promise>{};
    }

    Task get_return_object() {
        return Task{
            std::coroutine_handle<Promise>::from_promise(*this)
        };
    }

    void return_void() {}
};

// Task is class that wraps coroutine.
// In order to be scheduled to our scheduler
// coroutine has to return instance of
// this class with T = void.
// This class uses the Promises from above to remember
// what coroutine (if any) had called it,
// and when it is done it will resume the caller.
// Like the classes above, this type of class should
// also be implemented by a library or a framework,
// to allow application developers to focus on business logic.
// Application developer then only need to return object
// of this class from their functions.
template<typename T = void>
struct Task {
    Task(const Task&) = default;

    using promise_type = Promise<Task, T>;

    explicit Task(const std::coroutine_handle<Task<T>::promise_type> coro) :
        coroutine_(coro)
    {}

    // In order to be used in `co_await task` expressions
    // we need to implement these three functions
    // required to be awaitable.
    // So Task is awaitable (in structural subtyping sense).

    bool await_ready() {
        return coroutine_.done();
    }

    void await_suspend(std::coroutine_handle<> h) {
        // We will save our caller, we will later return to that caller.
        // This is equivalent to saving return address of
        // the caller function on the stack
        // when normal functions are called.
        coroutine_.promise().continuation = std::move(h);

        // After saving the caller, we resume the called coroutine.
        coroutine_.resume();
    }

    auto await_resume() const noexcept {
        // What this function returns will be returned from `co_await task`.
        // We use modern compile time C++ `if constexpr` statement to decide
        // weather we return nothing (void) or value from our promise based
        // on type of template parameter.
        if constexpr (std::is_same_v<T, void>) {
            return;
        } else {
            return std::move(coroutine_.promise().result);
        }
    }

    std::coroutine_handle<promise_type> coroutine_;
};

// This is our scheduler,
// which is analogous to EventLoop from Python example.
// It does not work with callbacks,
// it only works with tasks,
// but otherwise works in similar way.
// It has priority queue for timers and
// uses polling system calls to check
// which file descriptor has available data to
// be read (write operation is not implemented
// in this example but works in similar way).
struct Scheduler {
    std::vector<Task<>> tasks_;

    using timer_entry = std::pair<uint64_t, std::coroutine_handle<>>;

    struct timeGreater {
        bool operator()(const timer_entry& l,
                        const timer_entry& r) const {
            return l.first > r.first;
        }
    };
    std::priority_queue<timer_entry, std::vector<timer_entry>, timeGreater> timers_;
    std::vector<struct epoll_event> events_;
    int epoll_fd_;

    Scheduler() {
        epoll_fd_ = epoll_create1(0);
    }

    void add_task(Task<> task) {
        tasks_.push_back(task);
    }

    // Adds coroutine into timers priority queue. When time elapses
    // it will be resumed by this scheduler.
    void suspend_till_timeout_(int64_t ms, std::coroutine_handle<> h) {
        timers_.push(
                std::make_pair(epoch_time_ms() + ms, h));
    }

    // Makes coroutine wait for file descriptor to
    // have bytes available to read,
    // after which corresponding coroutine will
    // be resumed by this scheduler.
    // Here, unlike Python example, we don't use `select` system call
    // for polling but more modern `epoll`.
    void suspend_for_read_(std::coroutine_handle<> h, int fd) {
        struct epoll_event ev;
        ev.events = EPOLLIN | EPOLLET;
        setnonblocking(fd);
        ev.data.fd = fd;
        // We will save the coroutine address to user data of epoll event,
        // and will later when read is ready, we will restore
        // coroutine handle from that address.
        ev.data.ptr = h.address();
        epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, 0, &ev);
        events_.reserve(events_.capacity() + 1);
    }

    void run_forever() {
        std::vector<Task<>> tasks = tasks_;
        tasks_.clear();

        // Start initial tasks.
        for (auto t : tasks) {
            t.coroutine_.resume();
        }

        // Run the loop.
        while (true) {
            // First check if there are timers that are ready, and resume corresponding
            // coroutines.
            while (timers_.size() && timers_.top().first < epoch_time_ms()) {
                timers_.top().second.resume();
                timers_.pop();
            }

            // Now check if there are file descriptors that are ready for reading, and
            // resume corresponding coroutines.
            int nfds = epoll_wait(epoll_fd_, events_.data(), events_.capacity(), 100);
            for (int i = 0; i < nfds; ++i) {
                void* ptr = events_.data()[i].data.ptr;
                auto h = std::coroutine_handle<>::from_address(ptr);
                h.resume();
            }
        }
    }

    // This is the way we access our scheduler, we just use one single global
    // instance. There are better methods, but this one is simple.
    static Scheduler& instance() {
        static Scheduler scheduler;
        return scheduler;
    }
};

// This function returns awaitable,
// that when its `await_suspend` function is called,
// asks the scheduler to resume the suspended coroutine after some time.
// Functions like this one, and few of them below should also
// be implemented by a library or a framework.
auto coro_sleep(int64_t ms) {
    struct SleepAwaitable {
        int64_t ms_;
        bool await_ready() {return false; }
        void await_suspend(std::coroutine_handle<> h) {
            Scheduler::instance().suspend_till_timeout_(ms_, h);
        }
        void await_resume() {}
    };

    return SleepAwaitable{ms};
}

// This function returns awaitable,
// that when its `await_suspend` function is called,
// asks the scheduler to resume the suspended coroutine when
// corresponding file descriptor is ready for reading.
auto coro_wait_fd_read_ready(int fd) {
    struct ReadAwaitable {
        int fd_;
        bool await_ready() {return false; }
        void await_suspend(std::coroutine_handle<> h) {
            auto& scheduler = Scheduler::instance();
            scheduler.suspend_for_read_(h, fd_);
        }
        void await_resume() {}
    };

    return ReadAwaitable{fd};
}

/*
 * Application code:
 */

Task<int64_t> fetch_ms() {
    // This function simulates some IO operation.
    // It returns milliseconds that should be used to later call sleep,
    // in our simulated business logic below.
    // It simulates call to some remote server to fetch number of ms,
    // but actually just waits for key press on keyboard,
    // which is also IO operation, so the point is the same.
    // On Linux it is all just file descriptor, be it keyboard,
    // file, or network socket.


    // We could have used any file descriptor,
    // for regular file, or network connection,
    // or any other, but for simplicity we will use stdin (e.g. keyboard).
    int stdin_fd = 0; // 0 is fd of stdin.

    // We will wait until there are data to be read from stdin.
    // During this time other code can be executed by the scheduler.
    co_await coro_wait_fd_read_ready(stdin_fd);
    // At this point there is definitely something to read from fd,
    // and we will not block.

    // We will read all the data, but discard it, because we're only
    // interested in the fact that user has entered something in the
    // terminal. So we wait for user, instead of waiting on network socket
    // file descriptor to be read ready, to simulate response from
    // external server.
    const size_t count = 1024;
    char buf[count];
    while (read(stdin_fd, buf, count) > 0);

    co_return 3500;
}


Task<> func() {
    int ms = co_await fetch_ms();
    std::cout << "Will sleep now for " << ms << "ms\n";
    co_await coro_sleep(ms);
    std::cout << "Good morning\n";
}

Task<> print_hello_later() {
    co_await coro_sleep(5000);
    std::cout << "Hello\n";
}

int main()
{
    auto hello_handle = print_hello_later();
    Scheduler::instance().add_task(hello_handle);
    auto f_handle = func();
    Scheduler::instance().add_task(f_handle);
    std::cout << "Starting scheduler loop\n";
    Scheduler::instance().run_forever();
}

```

The code above produces the following output:

```
Starting scheduler loop

Will sleep now for 3500ms
Good morning
Hello
```

It can also produce the following output if the user presses the Enter key a bit later:

```
Starting scheduler loop

Will sleep now for 3500ms
Hello
Good morning
```

This code is not near optimal, probably it has number of bugs, certainly it has memory or other resource leaks. All that is left aside to make it shorter, and easier to notice the concept.

In reality, as C++ is performance oriented language, schedulers tend to run on multiple threads, each processing one coroutine in parallel. Usually each thread has its own queue of tasks (coroutines) to execute, when one gets suspended it takes another one, when task is ready to run again it is enqueued into one of the queues. Also, usually work stealing is employed, which means that when thread runs out of tasks to run, it tries to steal task from queues that belong to other threads.
