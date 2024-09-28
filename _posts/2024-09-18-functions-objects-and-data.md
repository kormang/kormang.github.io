---
layout: post
author: kormang
---

## Functions are objects

```typescript
class ClsA {
  call(a: number, b: number): number {
    return a + b
  }
}
```

is equivalent to

```typescript
function funcA(a: number, b: number): number {
  return a + b
}
```

and here is why

```typescript
funcA(1, 2)
// Is same as:
const objA = new ClsA()
objA.call(1, 2)
// Is same as:
funcA.call(null, 1, 2)
// Yes, in JavaScript functions are literally objects that have `call` method.
```

In JavaScript (and TypeScript), functions are literally objects, with the prototype `Function`, and these objects can be called either using regular call syntax or by calling the `call` method on them.

This is not only a characteristic of JavaScript.
The same applies in Python, where functions are objects with the `__call__` method. In Python, one can even create a custom class with a `__call__` method and call an instance of that class as if it were a regular function.

```python
class ClsA:
  def __call__(self, a, b):
    return a + b

objA = ClsA()
objA(1, 2)
```

In Java 8+, if interface has only one abstract method, in its place a function can be used, static function, or lambda.

Here is one such interface:

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```

We might have function like this one, that accepts such interface to filter numbers.

```java
static void printMatching(List<Integer> numbers, Predicate<Integer> predicate) {
  for (Integer n : numbers) {
    if (predicate.test(n)) {
      System.out.println(n);
    }
  }
}
```

To that function, we can pass either, explicit interface implementation, with `test` method implemented, or lambda anonymous function (`->`), or static function.

```java
class Main {
  static boolean isOdd(int i) {
    return i % 2 != 0;
  }

  public static void main(String[] args) {
    List<Integer> ints = Arrays.asList(1, 2, 3, 4, 5, 6, 7);

    printMatching(ints, new Predicate<Integer>() {
      public boolean test(Integer i) {
        return i % 3 == 0;
      }
    });
    printMatching(ints, n -> n % 2 == 0);
    printMatching(ints, Main::isOdd);
  }
}
```

In Scala, functions are also objects, with `apply` method.

In C++, we have call operator, so we can make object behave like functions.

```c++
class StringLengthSorter {
public:
    bool operator()(const std::string& str1, const std::string& str2) {
        return str1.length() < str2.length();
    }
};

std::vector<std::string> strings = {"apple", "banana", "cherry", "date", "elderberry"};

// Use std::sort with the custom StringLengthSorter functor,
// to sort strings by their length.
std::sort(strings.begin(), strings.end(), StringLengthSorter());
```

The above code can be simplified using lambda function.

```c++
std::vector<std::string> strings = {"apple", "banana", "cherry", "date", "elderberry"};

// Use std::sort with the custom lambda compare functor,
// to sort strings by their length.
std::sort(strings.begin(), strings.end(), [](const auto& s1, const auto& s2) {
  return s1.length() < s2.length();
});
```

Lambdas in C++ are in fact implemented as objects with call operator.

We can use normal C-style function in place of object, or lambda.

```c++
// Custom comparison function for sorting by string length
bool CompareByStringLength(const std::string& str1, const std::string& str2) {
    return str1.length() < str2.length();
}

std::vector<std::string> strings = {"apple", "banana", "cherry", "date", "elderberry"};

// Use std::sort with the custom comparison function
std::sort(strings.begin(), strings.end(), CompareByStringLength);

```

Hopefully, we've seen how functions and callable objects can be used interchangeably in different scenarios and languages. Each of these languages has different rules, limitations, and implementations when it comes to using functions as objects. However, what is common is that, semantically, functions are objects and can, in one way or another, be used interchangeably.

But this is not enough to say that functions are, in fact, objects. For functions to be fully recognized as objects, they need to be able to hold and update internal state. In C, for instance, a function is just a place in memory where a sequence of machine instructions is stored, without any state. But C is a low-level language that lacks many higher-order constructs. This is not a critique of C, but it does not have objects at all and is very close to assembly, so it's not entirely relevant for our discussion here. However, certain patterns we observe can be useful in reasoning about and designing code more effectively, even when writing in C.

Now, let's see how closure functions can be used to implement stateful objects in TypeScript.

```typescript
class ClsA {
  constructor(initialValue: number) {
    this._val = initialValue;
  }

  add(value: number): number {
    this._val += value;
    return this._val;
  }
}

const objA = new ClsA(1)
console.log(objA.add(2))
console.log(objA.add(3))
```

Since, our class has only one method, it is semantically equivalent to closure.

```typescript
function funcA(initialValue: number) {
  let _val = initialValue
  return function(value: number): number {
    _val += value
    return _val
  }
}

const fA = funcA(1)
console.log(fA(2))
console.log(fA(3))
```

As you can assume, the reason we use an outer and inner function is that the outer function serves as a constructor for our closure. Of course, constructors are not always as explicit as in this case, but they always exist in some form.

Read [this]({% post_url 2024-09-06-parameterizing-code-with-behavior %}#call-arguments) part on code with behavior, to see another real world example where such pattern in used, and to better understand it. Read [this]({% post_url 2024-09-06-parameterizing-code-with-behavior %}#dependency-injection-through-constructor) part about dependency injection to see example where classes and constructors are used equivalently.

**The moment, the place in code, where function, or object, is constructed, is of utmost importance.** After construction, object/function has to conform to certain interface (interface not just as implied by the keyword in many languages, but in a broader sense), but during construction it can be parameterized with anything, thus possibly making it highly reusable.

## Taking your data with you

(This section requires the understanding the concept of threads of execution. The concepts here could be illustrated using different examples, but threads are ideal way to illustrate it. The only drawback is that it might not be familiar to many readers. In next section we will try to explain these concepts in TypeScript which doesn't even have threads.)

Being able to accept some data during construction and then use it during execution, is the primary benefit of objects, including closures, which enables them to be reusable in an effortless way.

Let's see examples in three programming languages, to show how threads are started and used.

First, let's see how to do it in Java. We implement interface `Runnable` that has `run` method. Whatever is in `run` method will later be executed in new thread.

```java
class InThreadRepeater implements Runnable {
    private String message;
    private int N;

    public InThreadRepeater(String message, int N) {
        this.message = message;
        this.N = N;
    }

    @Override
    public void run() {
        for (int i = 0; i < N; i++) {
            System.out.println(message);
        }
    }
}

```

We parameterize our run function, with the message it should repeat in the new thread, and the number of repetitions. This allows us to reuse this `run` function, and start different threads with different messages and different number of repetitions.

```java

Thread thread1 = new Thread(new InThreadRepeater("Hello from thread 1", 5));
// Start the thread
thread1.start();

Thread thread2 = new Thread(new InThreadRepeater("Hello from thread 2", 2));
// Start the thread
thread2.start();

Thread thread3 = new Thread(new InThreadRepeater("Hello from thread 3", 15));
// Start the thread
thread3.start();

// Wait for threads to finish.
thread1.join();
thread2.join();
thread3.join();
```

In C language, where we don't have closures and objects, we need to emulate this behavior. Luckily, `pthread` library, for starting threads, is already made in such way to emulate closures.

```c
#include <stdio.h>
#include <pthread.h>

struct ClosureData {
  const char* message;
  int N;
};

// Function that the thread will execute
void runInThread(void *arg) {
  struct ClosureData* closure_data = (struct ClosureData*)arg;

  for (int i = 0; i < closure_data->N; i++) {
      printf("%s\n", closure_data->message);
  }

  // Exit the thread
  pthread_exit(NULL);
}

int main() {
    pthread_t thread1;
    pthread_t thread2;
    pthread_t thread3;

    struct ClosureData threadData1 = { "Hello from thread 1",  5 };
    struct ClosureData threadData2 = { "Hello from thread 2",  2 };
    struct ClosureData threadData3 = { "Hello from thread 3",  15 };

    // Create and run threads.
    pthread_create(&thread1, NULL, runInThread, (void *)&threadData1);
    pthread_create(&thread2, NULL, runInThread, (void *)&threadData2);
    pthread_create(&thread3, NULL, runInThread, (void *)&threadData3);

    // Wait for threads to finish.
    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);
    pthread_join(thread3, NULL);

    return 0;
}

```

As we can see, `struct ClosureData` is used to store data that will later be used inside the equivalent of the `run` function. To support that, we pass an additional parameter to it, since in C, the function is not a member of a class. The parameter `arg` emulates the `this` keyword in object-oriented languages. Since we don't have templates and generics, `pthread` expects a function that accepts `void*` as an argument, and later inside the function, we cast it to our struct to retrieve the necessary data.

Here is one way to implement this concept in Python, which illustrates the idea we're trying to grasp.

```python
import threading

# Function that the thread will execute
def createInThreadRepeater(message, N):
  def run():
    for _ in range(N):
        print(message)

  return run

# Create a new thread
thread1 = threading.Thread(target=createInThreadRepeater("Hello from thread 1", 5))
# Start the thread
thread1.start()
# Create a new thread
thread2 = threading.Thread(target=createInThreadRepeater("Hello from thread 2", 5))
# Start the thread
thread2.start()
# Create a new thread
thread3 = threading.Thread(target=createInThreadRepeater("Hello from thread 3", 5))
# Start the thread
thread3.start()

# Wait for threads to finish.
thread1.join()
thread2.join()
thread3.join()
```

Here, `createInThreadRepeater` returns inner function, closure, that runs inside thread. Implicitly `createInThreadRepeater` creates equivalent of instance of the `struct ClosureData` for the `run` function.

We have seen three different approaches to pass data along with the function, in three different languages. Semantically, however, they are the same. The difference is syntactic.

## Parameterize with behavior

We can, and in fact, we should, parameterize our objects and functions with behavior, not just with data.

Let’s say we have a video hosting platform where people can upload their videos, and those who are subscribed can get notifications about new videos being uploaded (it’s not YouTube).

When a new video gets uploaded, an event is enqueued into a queue. Then, different event handlers, or queue consumers, subscribed to specific events get notified about the new video. Each handler does its own task, but some responsibilities are common to all of them. For example, each consumer should validate the data, as the event is sent over the network and thus cannot be fully trusted. This validation could be optimized so that it's only performed once per process, but for now, it serves as a good example.

Let’s take a look at how a generalized consumer for the 'video_published' event might look.

```typescript
export class VideoPublishedConsumer {
    constructor(
        queueConnection: QueueConnection,
        readonly consumer: Consumer<VideoPublished>
    ) {
        queueConnection.subscribeQueueConsumer('video_published', this._onMessage);
    }

    _onMessage = async (msg: object) => {
        const videoPublishedData = msg as VideoPublished;
        validateVideoPublished(videoPublishedData);
        await this.consumer.consume(videoPublishedData);
    };
}
```

Now let's say that we want to instantiate new consumer that would notify subscribed users about new video. (We don't need to set `subFinder` and other params like this, we can use it directly as constructor parameter, but it might be more clear like this.)

```typescript
class NotifySubscribersConsumer {
  constructor(percentage: number, subsFinder: SubsFinder, videoNotifService: VideoNotifService) {
    this.percentage = percentage;
    this.subsFinder = subsFinder;
    this.videoNotifService = videoNotifService;
  }

  consume(data: VideoPublished) {
    let subscribers = this.subsFinder.findSubscribers(data.authorId);
    if (this.percentage < 1.0) {
      subscribers = randomSample(
        subscribers,
        Math.round(this.percentage * subscribers.length)
      )
    }
    subscribers.forEach(s => {
      this.videoNotifService.sendNewVideoNotif(
        s,
        data.videoName,
        data.videoDescription,
        data.authorName)
    })
  }
}
```

```typescript
new VideoPublishedConsumer(
  queueConnection,
  new NotifySubscribersConsumer(1.0, getSubsFinder(), getEmailService())
)
```

In this example, we use some subscribers finder to find subscribers for the author of the video, and email service to deliver notifications via email. We can also create another consumer that would send push notifications.

```typescript
new VideoPublishedConsumer(
  queueConnection,
  new NotifySubscribersConsumer(0.7, getSubsFinder(), getPushNotifService())
)
```

(In certain scenarios this might not be the most optimal way to implement it, but serves as an example for parameterizing with behavior)

Similarly we can create other consumers.

```typescript
new VideoPublishedConsumer(
  queueConnection,
  new UpdateStatsConsumer(getStatsService())
)
```

We can use parameters to change the behavior of a class, and some of these parameters represent behavior themselves. These parameters are not intended to represent state. In the class `NotifySubscribersConsumer`, for example, the `percentage` parameter changes the behavior of the class—though it is data, it directly influences behavior. If the percentage is less than 1, only a specified portion of randomly sampled subscribers will be notified (I’ll leave it to the reader to ponder why anyone would want that).

Both `VideoPublishedConsumer` and `NotifySubscribersConsumer` (as well as `UpdateStatsConsumer`, though its code is not shown) have parameters that are neither pure data nor state, but rather influence behavior.

Now, let’s explore how this concept appears in a purer form of behavior — functions.

```typescript
export async function subscribeVideoPublishedConsumer(
    queueConnection: QueueConnection,
    readonly consume: (videoPublishedData: VideoPublished) => Promise<void>
) {
    queueConnection.subscribeQueueConsumer(
      'video_published',
      async (msg: object) => {
        const videoPublishedData = msg as VideoPublished;
        validateVideoPublished(videoPublishedData);
        await consume(videoPublishedData);
    });
}
```

```typescript
function notifySubscribersConsumer(percentage: number, subsFinder: SubsFinder, videoNotifService: VideoNotifService) {
  return (data: VideoPublished) => {
    let subscribers = subsFinder.findSubscribers(data.authorId);
    if (percentage < 1.0) {
      subscribers = randomSample(
        subscribers,
        Math.round(percentage * subscribers.length)
      )
    }
    subscribers.forEach(s => {
      videoNotifService.sendNewVideoNotif(
        s,
        data.videoName,
        data.videoDescription,
        data.authorName)
    })
  }
}
```

```typescript
subscribeVideoPublishedConsumer(
  queueConnection,
  notifySubscribersConsumer(getSubsFinder(), getEmailService())
)
```

```typescript
subscribeVideoPublishedConsumer(
  queueConnection,
  notifySubscribersConsumer(getSubsFinder(), getPushNotifService())
)
```

```typescript
subscribeVideoPublishedConsumer(
  queueConnection,
  updateStatsConsumer(getStatsService())
)
```

We have seen that all of it can be implemented just combining functions.

Instead of interface like

```typescript
interface Consumer<T> {
  consume: (T) => Promise<void>;
}
```

We have simply function type like

```typescript
type Consumer = <T>(data: T) => Promise<void>
```

As we can see both objects and functions represent behavior.

## Managing State

Let's see one more time how objects and function, both can have state.

First let's see function that has state.

```typescript
function lengthCounter() {
  let _val = 0
  return function(text: string): number {
    _val += text.length
    return _val
  }
}

const lc = lengthCounter()
console.log(lc("Hello"))
console.log(lc("World"))
```

Now, let's see a class with the same behavior and internal state.

```typescript
class LengthCounter {
  constructor() {
    this._val = 0;
  }

  count(text: string): number {
    this._val += text.length;
    return this._val;
  }
}

const lc = new LengthCounter(1)
console.log(lc.count("Hello"))
console.log(lc.count("World"))
```

Here, we do have internal state, but we don't have any getter or setter for that state.

Object-Oriented Programming (OOP) promises encapsulation and loose coupling, but almost every course on OOP teaches us about the class Vehicle and its subclass Car, which has properties like model, engine, and so on. We've also been taught to create getters and setters to encapsulate those fields so that external code can't access them directly. But the truth is—it doesn't help. Yes, we can put some logic in the setter to ensure the value is valid or to perform other necessary actions when setting the value. But we've still exposed that field, and other code depends on it, expecting that it can retrieve the value after setting it.

This approach eventually leads us to API designs that require the client to "always set a certain value before calling certain methods." This, in turn, leads to nasty bugs like, "Oh, I forgot to return the old value after calling that method," or "That value was set by someone else in the meantime before I had a chance to call the method."

Here’s an example of the problem I’ve encountered, which, while extreme, effectively illustrates the issue. I’ve seen a class with a method we’ll call `doSomething()`. However, before calling `doSomething()`, the caller is expected to call `setFieldA()` and `setFieldB()` with the appropriate values. These fields were never used by any other method in the class and could have easily been replaced by arguments in the `doSomething()` method itself.

Interfaces like these create several problems. For instance, consider the following code:

```java
obj.setFieldA(aObj);
obj.setFieldB(bObj);
obj.doSomething();
```

but then immediately following that code we add:

```java
subprocedure(obj);
obj.setFieldB(bObj2);
obj.doSomething();
```

Field A could be changed by a `subprocedure`. While this may not happen now, in the future, someone might introduce a change that does modify it. In place of the call to `subprocedure`, there could be a large block of code — 50 lines or more — where such a change could be introduced. This makes the code highly prone to break in the future.

Setters are particularly problematic. While getters can be useful in certain languages to ensure immutability of fields (if no other mechanism exists), setters, on the other hand, allow anyone, from any part of the code, to modify the value. This often leads to the aforementioned bugs and state management issues. It's crucial to ensure that state cannot be changed from arbitrary parts of the code.

These types of bugs are eliminated when using pure functional programming, which implies that no state is mutable and that all data is immutable—once initialized, it cannot change. However, this isn't always practical, and it's often much more efficient to have mutable variables, arrays, etc. Mutability is often more efficient when we talk about local data, local state, and local variables in the "micro world." But in the "macro world," when we talk about modules and large programs, keeping track of who changed what state is difficult and leads to the aforementioned bugs. So, the more state is localized and hidden, the better. If we can use immutable objects (like the String class in many languages such as Python, JavaScript, and Java), even better. If having immutable objects is impractical or inefficient, we should at least hide the state as if it were local to a closure function and definitely avoid getters and setters.

Basically, functions are objects, and, as it seems, objects are also just functions?

Well, yes and no. Most objects are just behavior, or should be just behavior, and thus should behave like closure functions.

So why do we even use objects? Why don't we always use closure functions?

First, there are objects that are not purely behavior. Then there are functions that are often used together, such as `addEventListener` and `removeEventListener`. In such cases, it is easier to implement them as objects. We can implement them like this.

```javascript
function createEventEmitter() {
  const handlers = {};

  return {
    addEventListener(eventName, listener) {
      if (!handlers[eventName]) {
        handlers[eventName] = [];
      }
      handlers[eventName].push(listener);
    },
    removeEventListener(eventName, listener) {
      if (!handlers[eventName]) return;

      handlers[eventName] = handlers[eventName].filter(l => l !== listener);
    },
    emit(eventName, ...args) {
      if (!handlers[eventName]) return;

      handlers[eventName].forEach(listener => listener(...args));
    }
  };
}
```

Note that `addEventListener`, `removeEventListener`, and `emit` share a common state. Objects that have more than one function are not analogous to a function — or more precisely, a closure — that has its internal state. They are similar to multiple closures sharing that state. However, just like closures that share internal state via captured local variables, classes should keep fields private, treating them as implementation details.

If you introduce a setter that modifies a value, it becomes reasonable to expect a corresponding getter to retrieve that value later. This turns the field from being an implementation detail to becoming part of the public interface, despite the control and encapsulation that setters and getters can provide.

Being aware that classes are just one way to group functions so they share common internal/private state can be challenging if you haven’t worked in languages that use closures extensively. Even if you’re programming in C++, it's beneficial to explore languages that rely heavily on closures to shift your paradigm and broaden your perspective. C++11 lambdas can help, but not as much.

Use classes, or groups of closures, to make them share common private/internal state.

### Closures and readability

Here we arrive at the third reason not to use functions all the time. While it’s certainly possible to always use functions—and that approach has advantages due to its uniformity and isomorphism—there are downsides. LISP, for example, represents everything as lists, and this feature, called homoiconicity, makes LISP extremely flexible and expressive. However, it also makes LISP hard for humans to read because everything looks the same. It seems that we humans need certain markers in the code to more easily identify its meaning. When we see words like `class` and `constructor`, we more quickly and easily grasp the meaning of the code compared to seeing the word `function`. Even `function` is "easier" on the eyes than reading code that consists exclusively of lists. We have to analyze the structure of a `function` to recognize the pattern it represents. Additionally, depending on the language, it might be impossible or impractical to create objects with methods as shown in the earlier example. So, both approaches have their pros and cons.

### Not all objects are behavior

Now, let's return to the first point—not all objects represent behavior. Some objects are explicitly designed to hold data. Such objects behave like `struct`s in C (they only have properties, no methods), or they have getters, setters, and a few other helper functions that do not mutate state (e.g., `toString`, comparison methods like `equals`, or overloaded operators like `==`). These are sometimes called Data Transfer Objects (DTOs) or instances of data classes. It is a good practice to treat them as immutable **values**. A good example of this pattern is the data classes in Kotlin.

There is also a third type of object, explicitly designed to hold mutable state: data structures. For example, lists, hash tables, and so on. Their entire purpose is to maintain mutable state. However, they don't have getters and setters for individual fields.


## Conclusion

We have seen that functions are just callable objects.

We've also established that objects can belong to one of three categories, and we should avoid objects with setters.

The three categories are:

1. **Behavior Objects**: Essentially functions or groups of functions. They are ideally immutable, but even if they have mutable state, it should be hidden and obscured, much like the internal state of a closure. They can be parameterized with data and other behaviors that influence the instantiated object's behavior, and this data and behavior can be saved as local immutable state within the object or closure. (More about that in [in this post]({% post_url 2024-09-06-parameterizing-code-with-behavior %}))

2. **Data Objects**: Similar to `struct`s in C, but they can also have methods like `toString`, `equals`, etc. They are usually treated as immutable values. A good example of this pattern is data classes in Kotlin.

3. **Data Structures**: Their primary purpose is to hold and manage mutable state efficiently (e.g., lists, vectors, hash tables, etc.).
