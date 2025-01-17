---
layout: post
author: kormang
---

This post will introduce you to the following observation:

**Just as any program written using iteration can be rewritten using recursion, any program that uses event handlers (callbacks) can be rewritten using polling or "async functions" (coroutines). There is no better approach, there is only more convenient approach for specific use case and situation.**

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

Simply put, when the CPU receives a signal from an I/O device, it stops whatever it was doing and starts executing an Interrupt Service Routine (ISR). The ISR acts as an event handler for events such as mouse movements, screen touches, or new data received over the network.

Just out of curiosity, let's briefly examine how this works in a toy operating system for processors with x86 architecture.

_In this OS, "task" is common name for both process and thread (in Linux threads are called light weight processes, because there is not much difference between the two when it comes to scheduling)._

Now let's look at the `getc` function, how it works actually. Let's remind ourselves what it is.

In C programming language we can read single character that is pressed on the keyboard. To do this, we can simple call `getc` function. It is similar to python's `c = input()` but just for single character.

```c
char c = getc(); // We we press key on the keyboard getc will return pressed character.
```

We mentioned `getc` in the [previous post](/2025/01/11/AC-part1-intro-to-concurrency.html#what-happens-when-we-wait-on-getc), with more details.


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

Next, let's look at how the keyboard ISR looks like. Additional assembly code is needed to register this function as keyboard interrupt handler, but eventually CPU will call this function when key is pressed on the keyboard, and corresponding signal is sent from keyboard to CPU.

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
    // If we put it in eax register calling convention says that the task will
    // observe this value as return value of the function call.
    t->registers.eax = c;
    // Also, this means that this task is no longer waiting for keyboard,
    // so clear that tag.
    t->waiting_input &= ~WAITING_KEYBOARD;
  }
}

```

Finally, when a newly scheduled task starts waiting for an I/O operation or the time for preemptive multitasking runs out, the newly scheduled task will be "kicked out," and the task that received the keyboard input might get a chance to run again.

It will then simply wake up and return from the `getc` function with the pressed character as the result (this requires some assembly magic, so we will skip that).

The OS can also offer **non-blocking APIs**.

Non-blocking APIs are typically implemented using **polling**—the application does not sleep while waiting; instead, it constantly asks the OS if any I/O operation has been completed.

At least in theory, the OS can also offer asynchronous APIs that call event handlers in the application whenever the desired I/O operation is completed. In practice, polling is commonly used, but there are user-space libraries that turn polling into an event-handling API for applications to use. These libraries use **event loops**, which will be the subject of the next post in this series.

### Application using event handlers

Now, imagine how it would look to process those characters in application code via event handlers.

Application code runs in what is called user space. Unlike kernel space, user space code cannot access hardware, or write and read to/from IO pins directly. These instructions are not available to user space code; the CPU will not execute them.

The kernel can call an event handler for a new character when a keyboard key is pressed, and user space code can accept this new character in this way.

Now, let's see how it would look to write an application that accepts new keys and performs desired actions depending on which key is pressed. We will write a simple application in C (but simple C that should be understandable to anyone). The application has a primitive user interface and limited functionality.

First, the user is supposed to enter key '1' to work with customer records, '2' to generate a report. When working with customer records, we can select sub-options - again '1' to add a new customer, '2' to edit an existing customer, and '3' to delete a customer. When generating a report, we can select '1' to generate a yearly report, or '2' for a monthly report.

#### Explicit state machine approach

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

#### Less explicit state machine

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

### Application using blocking IO

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

* Regular timer interrupts to prevent a single thread from taking too long. This can interrupt the execution of the thread at any time, at any instruction. This is called preemptive multitasking.
* Calls to blocking kernel APIs, during which the kernel can put the caller thread to sleep until the data it is waiting for is ready, and put another thread to work during that time.

IO APIs can be either blocking or non-blocking. Blocking APIs are traditional APIs, like `getc`, which put application to sleep until IO operation is ready.

Non-blocking APIs use a polling approach.

There are also asynchronous APIs that call event handlers. These are typically used at the hardware level and by applications that utilize an event loop.

We briefly examined how the event-handling approach can be translated into a blocking approach by the OS. For example, when a key is pressed, an event handler is triggered in the OS kernel, but the OS presents this as a blocking API to applications.

In fact, all these approaches to I/O can be converted from one to another by adding additional layers of abstraction. The OS can convert event handling to a blocking or polling approach. Similarly, a library can transform a polling approach back into event handling using an event loop (which will be the subject of the next post) or into a blocking approach (to be discussed in a future post in this series).
