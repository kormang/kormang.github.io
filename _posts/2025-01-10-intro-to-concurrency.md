---
layout: post
author: kormang
---

Asynchronicity and concurrency are difficult subjects, and this post is as well. This post is first (and the most boring) post in a series of posts that tend to explain how asynchronous programming and concurrency work. After reading this series of posts, asynchronicity and concurrency should no longer be a mystery.

_Study each code example carefully, copy it and run it. Try to change something, and run it again; try to find bugs, try to find answers to remaining questions. That is the process of learning. If you're not familiar with some of the languages used in the examples, feel free to skip them, but it's better to try to understand them._

The first post will introduce basic concepts, as well as introduce you to the following observation:

**Just as any program written using iteration can be rewritten using recursion, any program that uses event handlers (callbacks) can be rewritten using polling or "async functions" (coroutines). There is no better approach, there is only more convenient approach for specific use case and situation.**

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

For example, when a keyboard is connected to the CPU (usually indirectly, but simplified here), it might use 8 pins for communication. If the keyboard puts low voltage on the first 5 pins and high voltage on the last 3, the CPU reads this as the binary number 00000111, which equals 7 in decimal. Based on the established protocol, this value carries meaning that the CPU can act upon.

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

Exact output depends on a lot of circumstances, and might be different each time we run the program, because those two functions, `keyboard_thread` and `message_thread` are running at the same time (if there are two CPUs then literally, otherwise with time-sharing).

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


## Layers of abstraction

Important thing to understand is that on different layers of abstraction implementation can be based on polling or events.

Devices connected to the CPU will usually send signals to CPU pins when they have something to transmit or have already transmitted data (from disk to memory, for example). On the level of transistors in the CPU, there could be different implementations. The CPU might check if there are any new signals after each instruction executed, or if it receives some signal (for example, voltage on a certain pin increases), transistors could be set up in such a way that those parts, those transistors that execute instructions, are literally cut off and redirected to processing the new signal. So even on this level, we have either checking for the signal ourselves or being somehow interrupted and notified about the signal.

When CPU is connected to independent devices like keyboard, they communicate asynchronously via message passing (actually electrical signals). CPU executes instructions synchronized to the CPU clock cycles, but the keyboard may change the voltage level on the CPU pin asynchronously to the CPU clock. This leads to the [problem](https://stackoverflow.com/questions/21688952/vhdl-logic-produces-wrong-result-when-using-higher-frequencies) called [Metastability](https://en.wikipedia.org/wiki/Metastability_(electronics)) in electronics. This makes asynchronous communication fundamental problem even on the transistor level.

![Clock signal, sync signal, and async signal](/assets/images/clock_sync_async_signals.png "Clock signal, sync signal, and async signal")

### On the OS Level

Regardless of the low-level implementation, the CPU provides an interface that allows a program (typically the operating system kernel, except in microcontroller systems) to interact with signals in two ways. It can either poll the CPU by explicitly executing instructions to check for new signals, or the CPU can interrupt program execution when signals arrive.

In the polling approach, the program executes an instruction that either proceeds to the next instruction if a new signal is present, or jumps to a different instruction if no signals are detected. Let's examine how this would look in pseudo C-like code:

```c
while (we_need_to_continue_polling) {
  if (signal_has_arrived) {
    process_signal();
  } else {
    do_something_when_no_signal();
  }
}
```

That C(-like) code is compiled into machine instructions and executed by the CPU.

In the second case, when the CPU is interrupting the program, for example, the CPU is executing instructions that belong to a specific function (let's say the function belongs to the OS kernel), and suddenly it receives an interrupt signal. Instead of continuing the execution of the next instruction, the CPU itself jumps to an instruction that is located in a predefined memory location for that kind of signal. At that memory location, there is a function which is usually called an Interrupt Service Routine (ISR). So, if we want to process a certain signal in a specific way, we need to put instructions that do the processing at that location (which is analogous to registering an event listener in high-level languages). Most of the CPUs behave this way.

Simply put, when CPU receives signal from IO device, it will stop doing what ever it was doing, and start executing Interrupt Service Routine (ISR), which is equivalent of event handler for events like mouse move, screen touch, new data received over network, etc.

Just out of curiosity, let's briefly examine how this works in a toy operating system for processors with x86 architecture.

_In this OS, "task" is common name for both process and thread (in Linux threads are called light weight processes, because there is not much difference between the two when it comes to scheduling)._

The function below is what gets called in the kernel when program calls `getc`.

```c
void sc_getc(registers_t* regs) {
  // Tag the task as "waiting for keyboard".
  current_task->waiting_input |= WAITING_KEYBOARD;
  // While this task waits for keyboard, switch to some other task.
  switch_task_from_isr(regs);
}
```

It will mark current task as waiting for keyboard input, and then it will switch to some other task while current task is waiting for keyboard. **This is the essence of concurrency at this level.**

_The function `switch_task_from_isr` is called that way because `sc_getc` also runs in a ISR, but it has to do with how system calls (switching from user space program to kernel) work on x86 CPUs._

Next is how the keyboard ISR looks like. Additional assembly code is needed to register this function as keyboard interrupt handler, but eventually CPU will call this function when key is pressed on the keyboard, and corresponding signal is sent from keyboard to CPU.

```c
static void keyboard_handler(registers_t* regs) {
    uint8_t scancode;

    /* Read from the keyboard's data buffer */
    scancode = inb(0x60);

    /* If the top bit of the byte we read from the keyboard is
    *  set, that means that a key has just been released */
    if (scancode & 0x80) {
        /* This one can be used to see if the user released the
        *  shift, alt, or control keys... */
    } else {
        /* Here, a key was just pressed. Please note that if you
        *  hold a key down, you will get repeated key press
        *  interrupts.
        */
        on_key_pressed(kbdus[scancode]);
    }
}
```

```c
void on_key_pressed(char c) {
  // When key is pressed, first we need to find what task was waiting for it.
  task_t* t = find_task_waiting(WAITING_KEYBOARD);
  if (t) {
    // If there is such task, then put the character in the register that will
    // hold the result of `getc` function call, when the task wakes up again.
    t->registers.eax = c;
    // Also, this means that this task is no longer waiting for keyboard,
    // so clear that tag.
    t->waiting_input &= ~WAITING_KEYBOARD;
  }
}

```

Finally when newly scheduled task starts waiting for some IO operation, or the time for preemptive multitasking ticks out, newly scheduled task will be "kicked out", and the task that got the keyboard input might get its chance to run again.

It will then simply wake up and return from `getc` function with pressed character as result (this requires some assembly magic, so we will skip that).

OS can also offer **non-blocking APIs**.

Non-blocking APIs are typically implemented using **polling** - application does not sleep while waiting, instead it is constantly asking OS if there is some IO operation that is completed.

At least in theory OS can also offer asynchronous API, which calls event handlers in application, when ever desired IO operation is completed. In practice polling is used, but there are user-space libraries that turn polling into event handling API for application to use. These libraries use **event loops**, which will be the subject for the next post in this series.

### Application using event handlers

Now, imagine how it would look to process those characters in application code via event handlers.

Application code runs in what is called user space. Unlike kernel space, user space code cannot access hardware, or write and read to/from IO pins directly. These instructions are not available to user space code; the CPU will not execute them.

The kernel can call an event handler for a new character when a keyboard key is pressed, and user space code can accept this new character in this way.

Now, let's see how it would look to write an application that accepts new keys and performs desired actions depending on which key is pressed. We will write a simple application in C (but simple C that should be **understandable to anyone**). The application has a primitive user interface and limited functionality.

First, the user is supposed to enter key '1' to work with customer records, '2' to generate a report. When working with customer records, we can select sub-options - again '1' to add a new customer, '2' to edit an existing customer, and '3' to delete a customer. When generating a report, we can select '1' to generate a yearly report, or '2' for a monthly report.

Let's see what the implementation would look like.

```c
typedef enum {
  NOT_CHOSEN,
  CUSTOMER_RECORDS,
  REPORT_GENERATION
} initial_option_t;

typedef enum {
  CUSTOMER_RECORDS_NOT_CHOSEN,
  CUSTOMER_RECORDS_ADD,
  CUSTOMER_RECORDS_EDIT,
  CUSTOMER_RECORDS_DELETE
} customer_records_option_t;

typedef enum {
  REPORTS_NOT_CHOSEN,
  REPORTS_YEARLY,
  REPORTS_MONTHLY
} reports_option_t;

initial_option_t initial_option = NOT_CHOSEN;
customer_records_option_t customer_records_option = CUSTOMER_RECORDS_NOT_CHOSEN;
reports_option_t reports_option = REPORTS_NOT_CHOSEN;

void on_character(char c) {
  switch (initial_option) {
    case NOT_CHOSEN:
      if (c == '1') {
        initial_option = CUSTOMER_RECORDS;
        printf("Press 1 to add new customer, 2 to edit existing customer, and 3 to delete existing customer:");
      } else if (c == '2') {
        initial_option = REPORT_GENERATION;
        printf("Press 1 to generate yearly report, press 2 to generate monthly report:");
      }
      break;
    case CUSTOMER_RECORDS:
      switch (customer_records_option) {
        case CUSTOMER_RECORDS_NOT_CHOSEN:
          switch (c) {
            case '1':
              customer_records_option = CUSTOMER_RECORDS_ADD;
            case '2':
              customer_records_option = CUSTOMER_RECORDS_EDIT;
            case '3':
              customer_records_option = CUSTOMER_RECORDS_DELETE;
          }
          break;
        case CUSTOMER_RECORDS_ADD:
          // TODO
          break;
        case CUSTOMER_RECORDS_EDIT:
          // TODO
          break;
        case CUSTOMER_RECORDS_DELETE:
          // TODO
          break;
      }
      break;
    case REPORT_GENERATION:
      switch (reports_option) {
        case REPORTS_NOT_CHOSEN:
          switch (c) {
            case '1':
              reports_option = REPORTS_YEARLY;
            case '2':
              reports_option = REPORTS_MONTHLY;
          }
          break;
        case REPORTS_YEARLY:
          // TODO
          break;
        case REPORTS_MONTHLY:
          // TODO
          break;
      }
      break;
  }
}
```

This is not easy to follow, right? The reason is that we don't have a linear flow of execution, and we manage state manually. We have an explicit state machine. There is, for sure, a better way to implement it. For a start, we don't have to put everything into one function, but as long as we have to manage state explicitly, the code will be hard to understand and maintain.

One way to make the state machine less explicit is to have the state as a function, and switch function pointers instead of enum values.

```c

typedef void (*char_handler_t)(char);

void customer_records(char c);
void report_generation(char c)

void not_chosen(char c) {
  if (c == '1') {
    initial_option = customer_records;
    printf("Press 1 to add new customer, 2 to edit existing customer, and 3 to delete existing customer:");
  } else if (c == '2') {
    initial_option = report_generation;
    printf("Press 1 to generate yearly report, press 2 to generate monthly report:");
  }
}

void customer_records_not_chosen(char c);
// Implementation omitted.
void customer_records_add(char c);
void customer_records_edit(char c);
void customer_records_delete(char c);

char_handler_t customer_records_handler = customer_records_not_chosen;

void customer_records(char c) {
  customer_records_handler(c);
}

void reports_not_chosen(char c);
// Implementation omitted.
void reports_yearly(char c);
void reports_monthly(char c);

char_handler_t reports_handler = reports_not_chosen;

void report_generation(char c) {
  reports_handler(c);
}

void customer_records_not_chosen(char c) {
  switch (c) {
    case '1':
      customer_records_handler = customer_records_add;
    case '2':
      customer_records_handler = customer_records_edit;
    case '3':
      customer_records_handler = customer_records_delete;
  }
}

void reports_not_chosen(char c) {
  switch (c) {
    case '1':
      reports_handler = reports_yearly;
    case '2':
      reports_handler = reports_monthly;
  }
}

char_handler_t initial_handler = not_chosen;

void on_character(char c) {
  initial_handler(c);
}
```

Maybe it adds a bit of readability, but still not enough, fundamentally it is the same.

### Alternative to callbacks and state machines

What we want is a code that does not use char handler, but that calls `getc`.

```c

customer_records_add();
customer_records_edit();
customer_records_delete();
reports_yearly();
reports_monthly();

int main() {
  chat c = getc(stdin);

  if (c == '1') {
    printf("Press 1 to add new customer, 2 to edit existing customer, and 3 to delete existing customer:");

    c = getc(stdin);

    switch (c) {
      case '1':
        customer_records_add();
      case '2':
        customer_records_edit();
      case '3':
        customer_records_delete();
    }

  } else if (c == '2') {
    printf("Press 1 to generate yearly report, press 2 to generate monthly report:");

    c = getc(stdin);

    switch (c) {
      case '1':
        reports_yearly();
      case '2':
        reports_monthly();
    }
  }

  return 0;
}
```

Although parts could be extracted into separate functions, it is still much more concise and readable. Just compare it to the previous two versions.

## Where we're at and what's next

We know that multiple processes can run on the computer, each doing something different, at the same time, concurrently. Each of these processes can also do multiple things at once by starting multiple threads. To give each process and thread a chance to run, the OS relies on either:

* Regular timer interrupts to prevent a single thread from taking too long. This can interrupt the execution of the thread at any time, at any instruction. This is called **preemptive multitasking**.
* Calls to blocking kernel APIs, during which the kernel can put the caller thread to sleep until the data it is waiting for is ready, and put another thread to work during that time.

IO APIs can be either blocking or non-blocking. Blocking APIs are traditional APIs, like `getc`, which put application to sleep until IO operation is ready.

Non-blocking APIs use polling approach.

There are also asynchronous APIs, that calls event handlers. This is usually used on the hardware level and by applications that use event loop.

We took a glimpse at how event handling approach can be translated to blocking approach by the OS (when key is pressed event handler is called in OS kernel, but OS presents it as blocking API to applications).

In fact all these approaches to IO can be turned from one to another by adding additional layers of indirection. OS can turn event handling to blocking, or polling approach. A library can turn polling approach back to event handling using event loop (which will be subject of the next post), or to blocking (which will be subject of one of the future posts in this series).
