---
layout: post
author: kormang
---

# Thoughts on Software Development

## Introduction

The goal is to explain some of the most important things I have learned from my experience and the experience of others (contained in books, articles, in-person talks, etc.), and hopefully help people around me write better software, and make their, and my own too, lives easier.

Examples will be predominantly in JavaScript and TypeScript to be accessible to as large an audience as possible. Sometimes other languages will be used to better illustrate the concepts. Hopefully, examples in those other languages will be useful for better understanding of the concepts. Knowledge of basic JavaScript is absolutely required to follow the material. Knowledge of Java is not required as examples should be simple and intuitive enough to allow anyone to understand them. Examples in C/C++ can be skipped, but they do present additional details to better understand the subject. Also, this material is initially and primarily meant for my colleagues, and in the company I work for at the moment of starting this journey.

It is hard to explain anything without examples, and it is hard to make examples that are short, yet illustrate the point perfectly. Reading examples will require focus; it is not expected that the reader can skim through the examples.

I'll be using the term "unit of code" to represent a function, class, file, component, module, basically any level of code organization.

Some concepts might seem too obvious, and you will have many "Well, I know this already" moments. That might be true, but it can also be the "Illusion of Competence" (a synonym for the Dunning-Kruger effect).

> _"All the advice that has come out of our revolution does not help much until it becomes second nature"_ - Apprenticeship Patterns, Guidance for the Aspiring Software Craftsman, by David H. Hoover and Adewale Oshineye.

If you don't practice until some of the advice presented here becomes natural to you, it will be painful to follow these pieces of advice. We can compare it to swimming. Let's say you know how to swim, but you get tired easily or swim too slowly. You could learn a new technique, you could learn to submerge your head underwater. But to do that, you need to let go of the old way of swimming, and for some time, until you get good at it, you need to accept that you'll swim worse (slower, more awkwardly) than you did previously by swimming the old way. The same can be said about skiing. You might have learned to ski in some suboptimal way. Maybe you can swing your hips during turns, and you're able to ski all the slopes you want. However, you might be struggling and losing too much energy. To overcome that problem, you need to learn to carve, and to do that you need to let go of the old way of skiing, to reduce your speed, and to pick easier slopes. Be it swimming, skiing, or some other activity, switching to a new technique will feel awkward, strange, and unnatural at the beginning. However, after you successfully master the new technique, it will definitely pay off. That period of decreased performance is the price of growth.

In the process of learning new things, you will probably overuse some of the advice, trying to apply it even when it is not applicable. That is OK. That is part of learning, of exploration. You will then understand the limitations of each piece of advice, each pattern, each technique, and each approach to solving software engineering problems. To make that process easier and to create fewer frustrations among your colleagues, remember that each piece of advice presented here is not always applicable, and sometimes you should do exactly the opposite. Use judgment each time you try to apply it - can this advice be applied here, depending on the goals and circumstances? Also note that at the beginning, you will be more likely to conclude that it is not applicable, but that is because you haven't mastered these pieces of advice yet.

I don't claim that everything I write here is absolute truth, and I would definitely like to get some feedback and grow from it.

### Precondition

There are two conditions you need to meet before continuing reading.

Solve these two tasks in no more than 5 minutes each, using only pen and paper:

First task:

You have following code:

```typescript
type Result = {
  odd: number[],
  even: number[],
}

function separate(values: number[]): Result {

}
```

Implement function `separate`, without changing function declaration or any code outside the function. The function should separate odd and even numbers from its input argument, in form of `Result` object.

If some other language is more familiar to you, then write analogous function in that language.

In Python, for example it would look like this:

```python
@dataclass
Result:
  odd: int[]
  even: int[]

def separate(values: int[]) -> Result:
  ...
```

In C++:

```c++
struct Result {
  std::vector<int> odd;
  std::vector<int> even;
}

Result separate(const std::vector<int>& values) {

}
```

Second task:

```typescript
function mostFrequent(values: number[]): number {

}
```

Implement function `mostFrequent`, without changing function declaration or any code outside the function. Function should return the number that is repeated the most in the input array `values`.

```python
def most_frequent(values: int[]) -> int:
  ...
```

```c++
int most_frequent(const std::vector<int>& values) {

}
```


**Why is this important?**
While some of you find this easy and solvable in less than one minute, many people are not able to solve it in 10 minutes. Those people build software, relatively successfully, much more complicated software than this function. Yet, the inability to solve this task using pen and paper indicates the inability to evaluate multiple solutions, multiple steps upfront in your head, which leads to every solution being made in the software - being the first solution authors came up with, and sometimes these solutions are local optima, far from global optima, far from thought-through solutions, just the first things that worked.

This work is about the absolute opposite. It is about picking the best solutions as possible, if not even the optimal solution, among many, for each problem.

Thus, it is incomprehensible for people who cannot solve this task.

The good news is that it is a trainable skill. In a few weeks or months, it is possible to train yourself to solve much more complicated tasks than this one. Then, after obtaining this new skill, a few more months working on real-life software development while having this skill, and your way of thinking will change, and you'll be ready to read the rest of this work, as well as read books on similar topics.

### Evolution of code

Software developers often want more freedom, more ability to hack their way through the problems, without many obstacles from tooling and programming languages.

This often makes sense. Especially if you want to validate some idea, to quickly try something out in a REPL, or even build a demo or prototype sometimes (depends on the situation). In such cases, you don't want to bother writing type annotations; you don't want your compiler or interpreter to tell you that a variable can be `null` when you know it isn't in this particular case. You don't care about naming variables, and generally, you're exploring and finding the shortest path to working code.

However, history has shown us that to build maintainable software, we need constraints. For some things, we want freedom, and for some, we want constraints.

One historical example that confirms this is the emergence of structured programming. Chances are you have never heard about it before. That is because now almost everybody programs using structured programming, so there is no need for a special word for it; it is just programming. However, that was not always the case.

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

From the beginning, `goto` statements found their way even to `C`. In C language one could declare a label and then use `goto label_name;` to jump to that label from anywhere within the function.

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

Imagine that we got that code in that form, and it doesn't matter how it became like that. Imagine also that we were asked to change it - we also need to print the statement if the number `numbers[i] / 2` also satisfies all the conditions. So, if `numbers[i]` meets the criteria, the statement should be printed, and if `numbers[i] / 2` also satisfies all the conditions, then the statement for that number should also be printed.

If you feel like this is getting too complicated, it is because it is, but we have to deal with such complexities everyday, and simpler examples often fail to illustrate the point. So stick with me, please.

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

Notice that structured programming is limiting our freedom and forces us to do some refactoring to meet new demands while maintaining the structure of the program. In particular, we can't use some quick trick "if", after that, our program will not satisfy conditions. For example, upon entering a loop, the program cannot go back to an earlier point in the code, before the loop.

This structure imposes constraints on our code but also makes it easier to reason about the code.

A more modern problem with the lack of structure arises when dealing with concurrent programming. That's why [structured concurrency](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) emerged as a way to solve the concurrent spaghetti code.

Even with structured programming and structured concurrency, larger programs can get complicated. In such cases, we should look for constraints that should be applied to our code to keep it organized and maintainable.

Frameworks often provide such constraints or frames; they limit what and where we can write certain code, providing structure and organization for the code.

Another important constraint mechanism is placing constraints on the dependency graph (which doesn't have to depend on our own discipline; there are tools that can check if we have imported modules that are not allowed to be imported from certain other modules).

### Price of a change

One of the advantages of de-duplicated code is the ability to make changes in only one place.

However, that is not always true, nor is it always cheaper to make changes in only one place or a few places rather than many places.

#### Mess example 1

Consider this case: we have a function, and for some reason, we either don't want or our language doesn't allow us to add another argument with a default value. Instead, we need to locate each call site and add another argument. This is relatively easy to do, even if the number of call sites is large. On the other hand, if we frequently have to make changes like this, then even if the changes are simple, the cost is high.

The opposite example would be having a few simple lines like this:

```typescript
  if (someVar <= 3) {
    result = processData(someVar, data)
    saveResult(result)
  }
```

We might have code like this in only three places, and it seems really simple. We see no point in extracting it to some function.

However, after a few months, we need to change it because the declaration and behavior of the function `processData` have changed; we need to call two different functions `processData2` and `processData3` depending on some conditions. The first place where we wrote that code after these changes looks like this (and there were no other changes to the code in that first place in the meantime):

```typescript
  if (someVar <= 2) {
    {result} = processData2(someVar, data)
    saveResult2(result.data)
  } else if (someVar == 3) {
    {result} = processData3(someVar, data)
    saveResult3(result.data)
  }
```

Seems simple enough. Now we need to do it in two more places. The change is not that trivial and is already starting to make us a bit nervous, but objectively it is still a simple thing to do. Right? Well, it doesn't have to be.

Imagine that the code in the second location changed in the meantime, and now it looks like this:

```typescript
  if (someVar <= 3 && eligibleToProcess) {
    result = processData(someVar, data)
  } else {
    result = null
  }

  if (result != null) {
    if (0.0 <= result.val1 && result.val1 <= 1.0) {
      result = processValidProb(result)
    }
  } else {
    someVar++;
    result = ensureResult(someVar)
  }

  result.val2 = correctValue2(someVar, result.val2)

  saveResult(result)

  ...
```

Now, it is not so trivial. Imagine that the third place where our code was duplicated was also changed in a similar way. Now we need to completely understand all changes made to this code. These changes were probably made by other people, and some of them may not even work on the project anymore. These changes likely relate to at least one more, and possibly a few other aspects of the project that we are not familiar with. The code in those calling functions mixes a few aspects and concerns, interleaving lines and even subexpressions related to aspect A of the code with behavior B and concern C of the project. Finally, it's intertwined with the responsibility for which we originally wrote this code.

Now, we have to comprehend it in its entirety, and our nervousness is justified for a good reason.

**How do we get into this situation?**

First, we were probably delving into possible solutions to implement a certain feature or trying to find the cause of the bug that we were attempting to get rid of, and it took us quite some time. After this exhausting search for a solution or for the cause, we arrived at a solution and added a few lines of code. We realized that we needed to apply it in three places, and we copied the code because we didn't have the energy or time to extract it and separate it from the rest of the system. Additionally, we wanted to avoid the challenging task of coming up with a name for the new function.

Then, a colleague joined us and found themselves in a similar situation. After contemplating the issue, digging into the code, and trying to understand the concepts, this colleague concluded that the easiest solution would be to process his task immediately after our code execution. He decided to modify the 'result' slightly, and that was it – just a few lines in one place, making it easy to implement.

Subsequently, another colleague arrived, and her task was easily solvable by modifying our existing condition. She added another term to it.

By not encapsulating our logic within a separate module and not segregating our responsibilities from the rest of the code, we inadvertently allowed our colleagues to **tightly couple** different concerns and blend code with different responsibilities into one unit of code.

This behavior exhibited by developers is caused by numerous factors:

- A focus on short-term gains (just getting my task done)
- A lack of responsibility and critical thinking to re-evaluate solutions after implementation
- Lack of time
- A tendency to simply do what everybody else does
- An tendency to make minimal alterations to code we don't own
- An inability to assess various solutions without actually implementing them (resulting in settling for the first solution, as described in the previous section with programming assignments)
- Fear of breaking things if we refactor them to a better state (often due to the absence of automated testing)

This is the way things unfold. Gradually, complexity grows, and even if the code was initially simple, just three lines, and was replicated in only three places. Eventually, it evolved into a nightmare to modify.

Not only did this code become more challenging to alter, but it also became harder to understand, which wasn't our original intention. The increased difficulty in understanding didn't come from a higher volume of code; rather, it emerged because the boundaries between different aspects, concerns, and responsibilities had become blurred.

#### Mess example 2

Let's see one more example of code getting complicated.

Imagine that we're building chat bot to entertain users. We already have one chat bot that you can talk to about certain topics, it has its name and other parameters, like main topic of the conversation, basic instruction on how to behave, and so on. User can configure those parameters depending on what user finds interesting. Each user can configure their own bot.

We find bot config by username. using function `getBotConfig`. That function can search the database to find the config for the bot by username, or any other way. The implementation of this function doesn't matter right now. When we find the config, we instantiate the bot that we are going to use for chat session. If there is no config for the username (user hasn't configured bot yet), we instantiate generic bot.

Imagine that existing code that runs chat session looks like this.

```typescript
function getBot(username: string): ChatBot {
  const botConfig = await getBotConfig(username)

  if (botConfig) {
    return new CustomBot(botConfig)
  }

  return new GenericBot()
}

function runBot(username: string) {
  const bot = getBot(username)
  while (!bot.hasFinished()) {
    const input = await getUserInput()
    const response = await bot.replyToMessage(input)
    await sendResponseToUser(response)
  }
}
```

After some time it is decided that we want to enable configuration of custom bot through another bot. We want "config bot" that would ask user some questions in order to create config for user's custom bot. For example, config bot might ask the user "How would you like to name your new bot?", and save the name user has selected to the bot config. After user has answered all the questions, bot should say to user something like this "Thank you for answering all the questions. You can now continue conversation with the bot you have just configured". After that user can simply continue conversation with the configured bot.

We might modify code the following way.


```typescript
const SWITCH_MSG = "Thank you for answering all the questions. You can now continue conversation with the bot you have just configured"

function getBot(botConfig?: BotConfig): ChatBot {
  if (botConfig) {
    return new CustomBot(botConfig)
  }
  return new ConfigurationBot()
}

function runBot(username: string) {
  let botConfig = await getBotConfig(username)

  let bot = getBot(botConfig)

  while (true) {
    const input = await getUserInput()
    const response = await bot.replyToMessage(input)
    await sendResponseToUser(response)

    if (bot.hasFinished()) {
      if (botConfig) {
        break;
      }
      await sendResponseToUser(SWITCH_MSG)
      botConfig = await getBotConfig(username)
      bot = CustomBot(botConfig)
    }
  }
}
```

We have slightly increased complexity of `runBot` functions that runs the chat session. Hopefully it is possible to understand how it works, although we aim to present at the end much clearer version, which is in fact the point of this example. This example illustrates some bad practices that makes code harder to understand. For example, additional explanation is probably needed to clarify the we use `break` when there is no `botConfig`, because we there is no `botConfig`, it must have been `ConfigurationBot` rather then `CustomBot`, that just finished the conversation with the user, and then we need to immediately connect user with newly configured `CustomBot`.

Now, imagine that we have new features to implement. For user named `admin` we need  `AdminHelperBot` that helps admin accomplish some of its tasks, it does not have `BotConfig`.

This could be the code that we write in similar fashion as the one before.

```typescript

function getBot(botConfig?: BotConfig, username: string): ChatBot {
  if (username === 'admin') {
      return new AdminHelperBot()
  }
  if (botConfig) {
    return new CustomBot(botConfig)
  }
  return new ConfigurationBot()
}

function runBot(username: string) {
  let botConfig = await getBotConfig(username)

  let bot = getBot(botConfig, username)

  while (true) {
    const input = await getUserInput()
    const response = await bot.replyToMessage(input)
    await sendResponseToUser(response)

    if (bot.hasFinished()) {
      if (botConfig || username === 'admin') {
        break;
      }
      await sendResponseToUser(SWITCH_MSG)
      botConfig = await getBotConfig(username)
      bot = new CustomBot(botConfig)
    }
  }
}
```

Now, imagine that for `guest` user we instantiate simply `GuestBot` which does not have config.


```typescript

function getBot(botConfig?: BotConfig, username: string): ChatBot {
  if (username === 'admin') {
      return new AdminHelperBot()
  }
  if (username === 'guest') {
    return new GuestBot()
  }
  if (botConfig) {
    return new CustomBot(botConfig)
  }
  return new ConfigurationBot()
}

function runBot(username: string) {
  let botConfig = await getBotConfig(username)

  let bot = getBot(botConfig, username)

  while (true) {
    const input = await getUserInput()
    const response = await bot.replyToMessage(input)
    await sendResponseToUser(response)

    if (bot.hasFinished()) {
      // if (botConfig || username === 'admin' || username === 'guest') {
      //   break;
      // }
      // Actually we decide that this is simpler.
      if (botConfig && isRegularUser(username)) {
        break;
      }
      await sendResponseToUser(SWITCH_MSG)
      botConfig = await getBotConfig(username)
      bot = new CustomBot(botConfig)
    }
  }
}
```

Now, we decide that admin can also configure its `AdminHelperBot` through `InitialAdminBot`, and that `InitialAdminBot` can have `BotConfig`. After admin has configured its helper bot, it will continue conversation with its helper bot immediately.


```typescript

function getBot(botConfig?: BotConfig, username: string): ChatBot {
  if (username === 'admin') {
    if (botConfig) {
      return new AdminHelperBot(botConfig)
    }
    return new InitialAdminBot()
  }
  if (username === 'guest') {
    return new GuestBot()
  }
  if (botConfig) {
    return new CustomBot(botConfig)
  }
  return new ConfigurationBot()
}

function runBot(username: string) {
  let botConfig = await getBotConfig(username)

  let bot = getBot(botConfig, username)

  while (true) {
    const input = await getUserInput()
    const response = await bot.replyToMessage(input)
    await sendResponseToUser(response)

    if (bot.hasFinished()) {
      if (botConfig && username === 'guest') {
        break
      }
      if (isRegularUser(username)) {
        await sendResponseToUser(SWITCH_MSG)
      }
      botConfig = await getBotConfig(username)
      bot = getBot(botConfig, username)
    }
  }
}
```

Now, it is obvious that the logic implemented in `runBot` is not obvious. Things have become a bit more complicated, and we get even more complicated with each new change.

**So, how did we get into this situation?**

We got into this situation by taking the shortest path to getting immediate task done, and then continuing making changes in that spirit.

**Was there another way?**

Yes, there was. Obviously `getBot` function had to change, but its logic wasn't super complicated, and if it ever gets (because of many conditions) it can get refactored.

Important thing to note is that all other logic, except from `runBot` and `getBot` functions, could have been implemented in specific bot classes. For example, we could have bot class that encapsulates both `CustomBot` and `ConfigurationBot` bot, and after inner `ConfigurationBot` finishes its part, it could switch to `CustomBot` internally - effectively implementing composition of bots. The same thing could have been implemented with admin bots. That way `runBot` stays the same, it runs bots, no matter what those bots do internally.

Let's name it `CombinedBot`, the bot that first configures custom bot using `ConfigurationBot` and then switch the conversation to newly configured `CustomBot`.

```typescript
class CombinedBot {
  constructor(private readonly username: string) {
    this._config_bot = new ConfigurationBot();
    this._current_bot = this._config_bot;
  }

  async replyToMessage(message: string): string {
    const reply = this._current_bot.replyToMessage(message)
    if (this._current_bot === this._config_bot && this._current_bot.hasFinished()) {
      // After config bot has finished, config for the new
      // custom bot, for the username, should be available.
      const config = await getConfig(this.username)
      this._current_bot = new CustomBot(config)
      return SWITCH_MSG
    }

    return reply
  }

  hasFinished(): bool {
    return this._current_bot.hasFinished()
  }
}
```

This way, we have composed third bot from the other two. We could do the same with `AdminHelperBot` and `InitialAdminBot`.

Let's see how could our code look like now.

```typescript

function getBot(botConfig?: BotConfig, username: string): ChatBot {
  if (username === 'admin') {
    if (botConfig) {
      return new AdminHelperBot(botConfig)
    }
    return new CombinedAdminBot()
  }
  if (username === 'guest') {
    return new GuestBot()
  }
  if (botConfig) {
    return new CustomBot(botConfig)
  }
  return new CombinedBot()
}


function runBot(username: string) {
  const bot = getBot(username)
  while (!bot.hasFinished()) {
    const input = await getUserInput()
    const response = await bot.replyToMessage(input)
    await sendResponseToUser(response)
  }
}
```

That's it.

Obviously, `runBot` could have stayed the same all the time, we haven't had to change it and complicate it at all. Code is much cleaner, much easier to read, and the best of all it is closed for modifications, but open for extension. When ever we add new kind of bot, `runBot` can stay the same, and can keep its simplicity and readability.


### So how do we avoid messing things up?

There is no straightforward way to provide a definitive answer to that question. Numerous books have extensively covered this topic. Certain well-known concepts and principles, such as Separation of Concerns and SOLID, have been formulated by brightest minds of the field like Edsger W. Dijkstra and Robert C. Martin. We should get familiar with those for sure, even if, as I do, you have a lot of critique for the work of the later.

Why those principles represent intermediate goals, towards good software design, patterns and other techniques represent means for those goals. There are many other books and materials on the subject. We can not cover them all here. Notable examples of books include "Design Patterns: Elements of Reusable Object-Oriented Software" by the "Gang of Four," and "Refactoring" (especially the second edition) by Martin Fowler.

Additionally, consciously learning from our own mistakes and gaining more experience are crucial steps. It's equally important to draw lessons from the errors of others. That is why following, so called, best practices is important. As Andy Hunt, in the "Pragmatic Thinking and Learning" explains, difference between novice in cooking and expert is that novice has to follow the rules, or recipes, strictly, while more experienced expert knows intuitively how will certain changes in the recipe affect the taste of the final meal. The same is with software development, and you can consider best practices to be those recipes. So just follow them blindly until you get enough experience to judge them when appropriate.

Examples of best practices are:
* Prefer Composition over Inheritance
* Prefer Immutable over Mutable data

There are books with common best practices, written almost as a books of short recipes, those are, e.g. "Effective Java" by Joshua Bloch, or "Effective C++" by Scott Meyers, and many others. Some of the advices given in those books are specific to those programming languages (Java, C++), some to paradigm (Object-oriented in this case) and some can be applied more generally to software development.

Here, in the upcoming articles, we will also try to help make this task of writing better software without messing things up, a bit easier to grasp, focusing on the most important aspects so you can start quickly. Further mastering the skill will require experience and further reading, including, but not limited to, books, papers, and other materials referenced in this material.

#### Proactively moving development into right direction

It is important to understand the concept of proactively moving code into right direction. If we reactively fix bugs and add features, that is, we only act upon current requirement and don't think ahead at all, then some other forces are going to shape our code and its architecture. If we don't write code in a good enough way (at least), we can be sure that our colleagues will abuse it and lead it to a state of complete mess. So when ever we write code we need to think of it as writing example for our colleagues and team mates on how to continue the development.
