---
layout: post
author: kormang
---

Why is evolution of code and programming languages actually evolution of constraints? Why is it sometimes more desirable to have more constraints than more freedom in software development? In this article we'll try to shed some light on that.

Software developers often want more freedom, more ability to hack their way through the problems, without many obstacles from tooling and programming languages.

This often makes sense. Especially if you want to validate some idea, to quickly try something out in a REPL, or even build a demo or prototype (depends on the situation). In such cases, you don't want to bother writing type annotations; you don't want your compiler or interpreter to tell you that a variable can be `null` when you know it isn't in this particular case. You don't care about naming variables, and generally, you're exploring and finding the shortest path to working code.

However, history has shown us that to build maintainable software, we need constraints. For some things, we want freedom, and for some, we want constraints.

Example of changes in constraints is dynamic memory allocation in languages that have garbage collector. This constraint made it easier to build software, not harder.

### Structured programming

Another historical example that confirms this is the emergence of structured programming. Chances are you have never heard about it before, because now almost everybody programs using structured programming, so there is no need for a special word for it; it is just programming. However, that was not always the case.

In the early days of programming, using `goto` statements was a common thing. It was directly inherited from assembly programming, which does not have constructs like loops.

Consider this program that finds the first even number greater than 10 from a list.

```javascript
const numbers = [3, 7, 12, 5, 18, 9, 20, 15];

for (let i = 0; i < numbers.length; i++) {
    const num = numbers[i];
    if (num <= 10) {
        continue;  // Skip numbers less than or equal to 10
    }

    if (num % 2 === 0) {
        console.log(`The first even number greater than 10 is: ${num}`);
        break;  // Exit the loop once the first even number is found
    }
}

```

Now consider same algorithm written in early version of BASIC programming language, that did not have loops and instead relied on `goto` statement to jump to specific line labeled with numbers. (That was just a illustration, you don't have to understand it.)

```basic
10 REM Initialize variables
20 LET X = 1
30 LET LIMIT = 8
40 LET FOUND = 0

50 REM Start of the loop
60 LET NUM = VAL("3 7 12 5 18 9 20 15")
70 IF X > LIMIT THEN GOTO 180

80 REM Check if NUM(X) is even and greater than 10
90 LET TEMP = NUM(X)
100 IF TEMP > 10 THEN GOTO 110
110 IF TEMP % 2 = 0 THEN GOTO 150

120 REM Increment index and continue loop
130 LET X = X + 1
140 GOTO 70

150 REM Output the first even number greater than 10
160 LET FOUND = TEMP
170 PRINT "The first even number greater than 10 is: "; FOUND
180 END

```

From the beginning, `goto` statements found their way even into C. In C language one could declare a label and then use `goto label_name;` to jump to that label from anywhere within the function.

Here is example of the same algorithm written in C, but without using loops, instead labels and goto statements are used.

```c
#include <stdio.h>

int main() {
    int numbers[] = {3, 7, 12, 5, 18, 9, 20, 15};
    int length = sizeof(numbers) / sizeof(numbers[0]);
    int i = 0;

loop_start:
    if (i >= length)
        goto loop_end;

    int num = numbers[i];
    if (num <= 10)
        goto next_iteration;

    if (num % 2 == 0) {
        printf("The first even number greater than 10 is: %d\n", num);
        goto loop_end;
    }

next_iteration:
    i++;
    goto loop_start;

loop_end:
    return 0;
}

```

Here is the "normal" C version of the same program.
Here, we've also added 5 more conditions, so the number has to satisfy 7 conditions in total.

```c
#include <stdio.h>

int main() {
    int numbers[] = {3, 7, 12, 5, 18, 9, 20, 15};
    int length = sizeof(numbers) / sizeof(numbers[0]);

    for (int i = 0; i < length; i++) {
        int num = numbers[i];
        if (num <= 10) {
            continue;
        }

        if (!satisfiesA(num)) {
          continue;
        }

        if (!satisfiesB(num)) {
          continue;
        }

        if (!satisfiesC(num)) {
          continue;
        }

        if (!satisfiesD(num)) {
          continue;
        }

        if (!satisfiesE(num)) {
          continue;
        }

        if (num % 2 == 0) {
            printf("%d satisfies all conditions\n", num);
            break;
        }
    }

    return 0;
}

```

Imagine that we got that code in that form, and it doesn't matter how it became like that. Imagine also that we were asked to change it - new requirement is to print the statement if the number `numbers[i] / 2` also satisfies all the conditions. So, if `numbers[i]` meets the criteria, the statement should be printed, and if `numbers[i] / 2` also satisfies all the conditions, then the statement for that number should also be printed.

If you feel like this is getting too complicated, it is because it is, but we have to deal with such complexities every day, and simpler examples often fail to illustrate the point. So stick with me, please.

We might be lazy, and the first thing that comes to our mind might be to reuse all those if statements with a quick trick:

```c
#include <stdio.h>

int main() {
    int numbers[] = {3, 7, 12, 5, 18, 9, 20, 15};
    int length = sizeof(numbers) / sizeof(numbers[0]);

    for (int i = 0; i < length; i++) {
        int num = numbers[i];
check:
        if (num <= 10) {
            continue;
        }

        if (!satisfiesA(num)) {
          continue;
        }

        if (!satisfiesB(num)) {
          continue;
        }

        if (!satisfiesC(num)) {
          continue;
        }

        if (!satisfiesD(num)) {
          continue;
        }

        if (!satisfiesE(num)) {
          continue;
        }

        if (num % 2 == 0) {
            printf("%d satisfies all conditions\n", num);
            i = length; // Don't continue the loop in any case.
            num = num / 2;
            goto check;
        }
    }

    return 0;
}

```

Now we have a bug there. If the new `num` satisfies the conditions, we will divide it by 2 and try again, which is not what our specification requires. So, to address this, we don't want to think about refactoring. When we're already here, we might just as well add a little `if` statement before `i = length;` to prevent it.

```c
#include <stdio.h>

int main() {
    int numbers[] = {3, 7, 12, 5, 18, 9, 20, 15};
    int length = sizeof(numbers) / sizeof(numbers[0]);

    for (int i = 0; i < length; i++) {
        int num = numbers[i];
check:
        if (num <= 10) {
            continue;
        }

        if (!satisfiesA(num)) {
          continue;
        }

        if (!satisfiesB(num)) {
          continue;
        }

        if (!satisfiesC(num)) {
          continue;
        }

        if (!satisfiesD(num)) {
          continue;
        }

        if (!satisfiesE(num)) {
          continue;
        }

        if (num % 2 == 0) {
            printf("%d satisfies all conditions\n", num);
            if (i == length) break; // Prevent doing next lines again.
            i = length; // Don't continue the loop in any case.
            num = num / 2;
            goto check;
        }
    }

    return 0;
}

```

This is the shortest path to implementing new requirements. However, it's also how we start cooking some spaghetti pasta out of our code. Can you suggest a refactoring?

In the late 1960s, Edsger W. Dijkstra wrote a paper titled "Go to statement considered harmful," critiquing the freedom the `goto` statement provides. He also authored "Notes on structured programming" as a case for programming in a structured manner, utilizing loops.

> Note that `goto` is still widely used in C, even today. One of the common uses cases is to put resource clean up code at the end of the function, and then instead of using early return when ever we want to exit the function we `goto cleanup`, to clenup the resource and then exit the function. This pattern is obsolite in other languages. For example in C++ compiler will do that for us, it will call destructors at the end of the function and jump to that part of the function before returning. When dealing with low level optimizations, for example in operating system where every instruction counts, `goto` is still often used.

Here is refactored code without `goto`:

```c
int satisfiesAllConditions(int num) {
  if (num <= 10) {
      return 0;
  }

  if (!satisfiesA(num)) {
    return 0;
  }

  if (!satisfiesB(num)) {
    return 0;
  }

  if (!satisfiesC(num)) {
    return 0;
  }

  if (!satisfiesD(num)) {
    return 0;
  }

  if (!satisfiesE(num)) {
    return 0;
  }

  if (num % 2 != 0) {
    return 0;
  }

  return 1;
}

int main() {
    int numbers[] = {3, 7, 12, 5, 18, 9, 20, 15};
    int length = sizeof(numbers) / sizeof(numbers[0]);

    for (int i = 0; i < length; i++) {
        int num = numbers[i];

        if (satisfiesAllConditions(num)) {
          printf("%d satisfies all conditions\n", num);
          if (satisfiesAllConditions(num / 2)) {
            printf("%d satisfies all conditions\n", num);
          }
          break;
        }
    }

    return 0;
}

```

Notice that structured programming is limiting our freedom and forces us to do some refactoring to meet new demands while maintaining the structure of the program. In particular, we can't use some quick and tricky "if". For example, upon entering a loop, the program cannot go back to an earlier point in the code, before the loop, so to reuse all the conditions we need to extract them into a separate function.

This structure imposes constraints on our code, forces us to do more work, but also makes it easier to reason about the code.

### Other types of constraints

A more modern problem with the lack of structure arises when dealing with concurrent programming. That's why [structured concurrency](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) emerged as a way to solve the concurrent spaghetti code.

Even with structured programming and structured concurrency, larger programs can get complicated. In such cases, we should look for constraints that should be applied to our code to keep it organized and maintainable.

Frameworks often provide such constraints or frames; they limit what and where we can write certain code, providing structure and organization for the code.

Another important constraint mechanism is placing constraints on the dependency graph (which doesn't have to depend on our own discipline; there are tools that can check if we have imported modules that are not allowed to be imported from certain other modules).

In next article we will see more example of how code turns into spaghetti when we take shortest path to meet new requirements. We will also see at alternative approaches and general principles we should fallow.
