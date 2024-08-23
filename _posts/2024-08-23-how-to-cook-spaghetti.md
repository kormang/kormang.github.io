---
layout: post
author: kormang
---

At first we make perfectly readable and beautiful code, but somehow it turns into spaghetti, or even worst, just a pile of garbage. How do we let that happen? Let's try to illustrate it with few examples.

One of the advantages of de-duplicated code is the ability to make changes in only one place.However, that is not always true, nor is it always cheaper to make changes in only one place or a few places rather than many places.

### Mess example 1

Consider this case: we have a function, and for some reason, we either don't want or our programming language doesn't allow us to add another argument with a default value. Instead, we need to locate each call site and add another argument. This is relatively easy to do, even if the number of call sites is large. On the other hand, if we frequently have to make changes like this, then even if the changes are simple, the cost is high.

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

**How did we get into this situation?**

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

### Mess example 2

Let's see one more example of code getting complicated.

Imagine that we're building chat bot to entertain users. We already have one chat bot that you can talk to about certain topics, it has its name and other parameters, like main topic of the conversation, basic instruction on how to behave, and so on. User can configure those parameters depending on what user finds interesting. Each user can configure their own bot.

We retrieve the bot configuration by username using the getBotConfig function. This function could be implemented in various ways, such as searching a database for the bot's configuration by username, but the specific implementation isn't important right now. Once we find the config, we instantiate the bot for the chat session. If no config exists for the username (meaning the user hasn't configured the bot yet), we instantiate a generic bot instead.

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

After some time it is decided that we want to enable configuration of custom bot through another bot. We want "config bot" that would ask user some questions in order to create config for user's custom bot. For example, "config bot" might ask the user "How would you like to name your new bot?", and save the name user has selected to the bot config. After user has answered all the questions, bot should say to user something like this "Thank you for answering all the questions. You can now continue conversation with the bot you have just configured". After that user can simply continue conversation with the configured bot.

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

We have slightly increased complexity of `runBot` functions that runs the chat session. Hopefully it is possible to understand how it works, although we aim to present at the end much clearer version, which is in fact the point of this example. This example illustrates some bad practices that makes code harder to understand. For example, additional explanation is probably needed to clarify the we use `break` when there is no `botConfig`, because there is no `botConfig`, it must have been `ConfigurationBot` rather then `CustomBot`, that just finished the conversation with the user, and then we need to immediately connect user with newly configured `CustomBot`.

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


## So how do we avoid messing things up?

There is no straightforward way to provide a definitive answer to that question. Numerous books have extensively covered this topic. Certain well-known concepts and principles, such as Separation of Concerns and SOLID, have been formulated. We should get familiar with those for sure, even if, as I do, you have a lot of critique for the later.

Why those principles represent intermediate goals, towards good software design, patterns and other techniques represent means for those goals. There are many other books and materials on the subject. We can not cover them all here. Notable examples of books include "Design Patterns: Elements of Reusable Object-Oriented Software" by the "Gang of Four," and "Refactoring" (especially the second edition) by Martin Fowler.

Additionally, consciously learning from our own mistakes and gaining more experience are crucial steps. It's equally important to draw lessons from the errors of others. That is why following, so called, best practices is important. As Andy Hunt, in the "Pragmatic Thinking and Learning" explains, difference between novice in cooking and expert is that novice has to follow the rules, or recipes, strictly, while more experienced expert knows intuitively how will certain changes in the recipe affect the taste of the final meal. The same is with software development, and we can consider best practices to be those recipes. So just follow them blindly until you get enough experience to judge them when appropriate.

Examples of best practices are:
* Prefer Composition over Inheritance
* Prefer Immutable over Mutable data

There are books with common best practices, written almost as a books of short recipes, those are, e.g. "Effective Java" by Joshua Bloch, or "Effective C++" by Scott Meyers, and many others. Some of the advices given in those books are specific to those programming languages (Java, C++), some to paradigm (Object-oriented in this case) and some can be applied more generally to software development.

Here, in the upcoming articles, we will also try to help make this task of writing better software without messing things up, a bit easier to grasp, focusing on the most important aspects so you can start quickly. Further mastering the skill will require experience and further reading, including, but not limited to, books, papers, and other materials referenced in this material.

### Proactively moving development into right direction

It is important to understand the concept of proactively moving code into right direction. If we reactively fix bugs and add features, that is, we only act upon current requirement and don't think ahead at all, then some other forces are going to shape our code and its architecture. If we don't write code in a good enough way (at least), we can be sure that our colleagues will abuse it and lead it to a state of complete mess. So when ever we write code we need to think of it as writing example for our colleagues and team mates on how to continue the development.

### Cleaning up your living space

One of the best analogies about maintaining code base I've seen so far is presented by Sarah Mei at RailsConf 2018, keynote presentation named Livable Code. Basic idea is that software is not like factory production, nor construction works. It is more like sharing and maintaining living space. Our code base is our virtual living space. We share it with our teammates (analogous to roommates), we leave some functions in the wrong place sometimes (analogous to leaving e.g. piece of clothing in wrong place for short period of time), but later we refactor it and clean it (analogous to taking that piece of clothing to proper place, and cleaning the house).

To get that idea properly, and many other ideas and advices it is highly recommended to watch the full presentation [Livable Code](https://www.youtube.com/watch?v=lI77oMKr5EY).
