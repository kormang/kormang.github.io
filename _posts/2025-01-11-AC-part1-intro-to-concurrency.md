---
layout: post
author: kormang
---

Asynchronicity and concurrency are difficult subjects, and this post is as well. This post is first (and the most boring) post in a series of posts that tend to explain how asynchronous programming and concurrency work. After reading this series of posts, asynchronicity and concurrency should no longer be a mystery.

Most examples will be in one of the three languages: Python, JavaScript and C. Most people should be able to read and understand these languages. C will be used only when it doesn't make sense to use Python or JavaScript. Later we will see how to build event loop and use async functions on top of it, in C++ as well. We will have examples in few other languages as well, when it makes sense, with the goal of broadening our perspective.

_Study each code example carefully, copy it and run it. Try to change something, and run it again; try to find bugs, try to find answers to remaining questions. That is the process of learning. If you're not familiar with some of the languages used in the examples, feel free to skip them, but it's better to try to understand them._

The first post will introduce basic concepts.

## Natural phenomenon

Asynchronicity is a programming approach for handling concurrent operations - activities that happen simultaneously but independently. We experience concurrency in our daily lives: while downloading a file, we might step away to make coffee, and during that time, multiple things happen in parallel: the download continues, the coffee machine brews our coffee, and we might chat with a colleague. Each activity proceeds independently of the others.
Computers handle concurrency in a similar way. Just as we can multitask in real life, a computer manages multiple independent operations simultaneously. For example, when we use a computer, it might be:

* Downloading a file from a remote server
* Processing user input from the keyboard
* Handling network communications
* Running background processes

These operations are analogous to our real-world examples: the remote server is like our colleague (an independent actor), the computer's processing units are like us (managing multiple tasks), and input/output devices like the network interface and keyboard are like the coffee machine (working independently while other operations continue).

![Simple computer model](/assets/images/simple_computer_model.png "Simple computer model")

The CPU executes instructions that are laid out sequentially in memory. These instructions can: store values into memory, read values from memory, perform operations (like addition or multiplication) on values stored in registers, and read from or write to output pins for I/O operations. Registers are small pieces of memory within the CPU itself that serve as its working memory.

Devices like GPUs, Network Interfaces, Disks, Timers, Keyboards, and Mice operate concurrently while the CPU performs computations. Communications with these devices are called Input/Output (I/O) operations. There are two main approaches to implementing I/O interactions with these devices.

Consider a real-life example of communication: When we send a message to someone, we have two options for handling their response. We can continue with other activities while waiting for their reply, checking it only when we receive a notification. Alternatively, we can focus entirely on waiting for their response, reading and acting on it immediately when it arrives. The second approach makes sense when we expect a quick reply and need to react to it immediately.

![Impatiently waiting for message](/assets/images/waiting_for_message.webp "Impatiently waiting for message")


In computer systems, the first approach typically relies on interrupts, events, or signals (different names for similar concepts). These mechanisms require special routines to process them - known as interrupt service routines, event handlers, signal handlers, or slots (again, varying terminology for essentially the same concept).

The second approach is usually called polling.

Polling and events both have their advantages and disadvantages as we already said. We will not go into details, just be aware of that.

There is also a third approach - we sleep and do nothing until the event we are interested in happens. Sometimes, it might seem that the third approach is actually the first approach, just a specific version of it, or just from a specific perspective. Similarly, it might sometimes seem like it is actually the second approach. It might not be obvious now, but soon, hopefully, it will be clear. The truth is, it depends on the layer of abstraction we are looking from.


## A bit of OS theory

### Short introduction to OS theory

When an operating system loads a program into memory and starts executing it, that program becomes what is called a process inside the operating system. Instructions that the process executes run with lower privileges - instructions that attempt to write directly to I/O pins will not be executed and will usually terminate the process instead. Additionally, processes are isolated from each other and from the OS kernel; they cannot read or write to memory that is not allocated to that process. This is done for security reasons, because we don't want a process that belongs to, e.g., a screenshot application, to be able to access memory and variables belonging to the browser, or for the browser to be able to read directly from anywhere on a disk.

To support hardware and memory isolation from processes, the OS needs hardware support, namely a Memory Management Unit, that provides support for virtual memory (inside of which processes are isolated) and CPU protection rings that provide restrictions on I/O instructions. Users' running programs - processes - run under Ring 3, which is called "user space," while the kernel runs in what is called "kernel space" (usually Ring 0, 1, or 2). When user space code (a process, e.g., a browser) wants to perform an I/O operation (e.g., make a network request), it needs to call an Application Programming Interface (API) provided by the OS kernel to do that. Kernel APIs not only provide a security mechanism but also a layer of abstraction over hardware, which makes programs portable across different hardware.

### What happens when we wait on 'getc'

In C programming language we can read single character that is pressed on the keyboard. To do this, we can simple call `getc` function.

```c
char c = getc();
```

Process that calls `getc` will stop execution until key is pressed. This is not asynchronous API, it is blocking API.

When a process calls `getc` (which is part of the C standard library), it will eventually call an OS kernel API, which will switch from user space to kernel space (this requires help from hardware). Once we're inside the kernel, we have much less restrictions and have direct access to hardware. Since the process that just called `getc` wants to wait until a key is pressed, the kernel knows that it might take a while until a key is pressed, so to utilize the CPU, the kernel can take another process and start executing its instructions (so another process runs on that CPU, while the one that called `getc` is put aside, waiting for the keyboard). Once a key is pressed, the CPU receives the keyboard interrupt request, which triggers the keyboard interrupt service routine. As we already said, such an event stops execution of the current code (usually user space process), and simply jumps to execution of the keyboard ISR. The ISR is inside kernel space and is able to save the key it got from the keyboard, find the process that was waiting for the key, and continue executing that process, while another process that was interrupted by the keyboard will have to wait its turn now.

Similar things happen with with other types of IO, like network traffic, not just key presses.

When data is ready (e.g., key from keyboard or response from a network request, or from reading a file from disk) the sleeping process is woken up, and provided with the desired data before continuing execution. A process does not have to wait for IO operations, it can also call `sleep` to wait for some time to pass, or simply call API functions like `yield`. That way, process just say, "I could wait now (for a specific time interval or simply take a break if there is somebody else waiting for the CPU)". This is called **cooperative multitasking**, where processes implicitly or explicitly allow other processes to take the CPU while they wait.

The OS provides synchronous blocking APIs to perform IO operations, although all IO is naturally performed in a highly concurrent and asynchronous manner.

### Preemptive multitasking

A similar mechanism enables OS kernels to run multiple processes on a single-core CPU. For example, we might start a video rendering application and, at the same time, play a video game. If we have a single CPU core, and these processes don't perform too many I/O operations but are instead CPU-intensive, the only way to interrupt one process is to set a timer and wait for a timer interrupt. That is what the OS kernel does. The kernel sets a timer for a short period, and if the current process doesn't perform any I/O or doesn't stop by the end of that period, the timer's interrupt will interrupt the process. Once in the timer ISR, the kernel can give another process a chance, but it will first set another timer to prevent that process from taking too long, and so on. That way, using timer interrupts, the kernel can run multiple processes on the same CPU core, each process will be running for short period of time, and processes will be taking their turns one after the other, giving the impression that they are running simultaneously. This mechanism is called **time-sharing**. With multiple CPU cores, processes can indeed run at the same time, but even then, there are usually more processes started than there are CPU cores. In practice, processes usually wait for I/O operations, which gives other processes a chance to run on the CPU.

### Threads

If a single process needs to perform two tasks simultaneously or concurrently, such as waiting for data to arrive over the network and waiting for keyboard input, it can start another thread of execution. Each process has a main thread of execution but can start additional threads. Threads can be blocked waiting for I/O operations and share CPU time with other threads from the same process or with other processes through time-sharing. Multiple threads belonging to the same process share the process's memory, but each has its own stack and set of instructions to execute. This enables one thread to wait for keyboard input while another makes a network request.

Here is an example of C program that starts two threads. One thread reads keyboard input (only single character using getc) and prints it back out 3 times. The other thread prints "Hello from other thread" indefinitely with one second pause between prints.

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

// Function for the thread that reads keyboard input and prints it 3 times.
void *keyboard_thread(void *arg) {
    char c;
    while (1) {
        c = getc(stdin); // Read a single character from input
        if (c == '\n') {
          continue; // Ignore newline characters
        }
        for (int i = 0; i < 3; i++) {
           printf("%c\n", c); // Print the character 3 times
        }
    }
    return NULL;
}

// Function for the thread that prints a message indefinitely.
void *message_thread(void *arg) {
    while (1) {
        printf("Hello from other thread\n");
        sleep(1); // Pause for 1 second to avoid flooding the output
    }
    return NULL;
}

int main() {
    pthread_t thread1, thread2;
    // Create and start the threads.
    pthread_create(&thread1, NULL, keyboard_thread, NULL);
    pthread_create(&thread2, NULL, message_thread, NULL);
    return 0;
}

```

If this is example input from user after starting the program:

```
a
b
```

then this could be possible output:

```
Hello from other thread
Hello from other thread
Hello from other thread
a
a
a
Hello from other thread
b
b
b
Hello from other thread
Hello from other thread
...
```

The exact output depends on many circumstances and might differ each time we run the program, because the two functions, `keyboard_thread` and `message_thread`, are running simultaneously (literally if there are two CPUs, otherwise with time-sharing).

Threads can also be used for parallel computing. A typical example is finding the sum of numbers in a very large array. For each CPU core, we start a new thread, and each thread sums a portion of the array. If there are 8 cores, each core will sum 1/8th of the whole array, and at the end, these partial sums will be combined. This speeds up processing by dividing the work among CPU cores, with each core running one thread. However, our current focus is on concurrent programming.

### Concurrency and parallelism

To illustrate the difference between parallel and concurrent programming, let's examine a few examples with graphical illustrations. Let's say we're downloading two files in parallel. Most of the time, the CPU is idle, waiting for data to be received over the network. Data arrives chunk by chunk. Whenever we get a chunk of data, there is a short period of processing before waiting for another chunk.

Let's suppose there is only one CPU core in the system. Those two files could be downloaded in two separate threads that run in short intervals and spend most of the time waiting.

Here is a simple timeline diagram representing the simultaneous downloading of two files from the network. The blue sections represent the periods of processing the first file, and the green sections represent the processing of the second file. The white or blank spaces on the timeline indicate the CPU is idle, waiting for chunks of files. The processing times are short and randomly scattered along the timeline, while the idle times are generally longer.

![download timeline 1 cpu](/assets/images/download_timeline_1cpu.png "download timeline 1 cpu")

If we would have two CPUs, it would look like this.

![download timeline 2 cpu](/assets/images/download_timeline_2cpu.png "download timeline 2 cpu")

It is obvious that we don't really need two CPU cores for that, as most of the time the CPU is waiting for chunks of data to arrive over the network. In case processing those chunks requires much more time, we would have something like this.

![download timeline 2 cpu overlap](/assets/images/download_timeline_2cpu_overlap.png "download timeline 2 cpu overlap")

It is now obvious that having two CPU cores helps, as data from both files could be processed at the same time.

![download timeline 1 cpu overlap](/assets/images/download_timeline_1cpu_overlap.png "download timeline 1 cpu overlap")

Now it is obvious that in some cases, processing one chunk of a file must wait until processing of another chunk from a different file is complete.

Parallel processing requires real OS threads to put multiple CPUs to work, while downloading multiple files with minimal CPU usage can be done in a single OS thread using an event loop and callbacks.

Matrix multiplication, finding the maximum number in an array, image processing, finding shortest paths in a graph, and other "number crunching" tasks are examples where multiple OS threads are needed to utilize multiple CPUs for parallelization. Some of these tasks are trivially parallelizable, while others require special algorithmic modifications.

## Up next

Next we will see how concurrency work on different layers on abstraction, from hardware, to OS, to application. We will also see different approaches to concurrency - blocking, polling, event handling, and how one turns into another.