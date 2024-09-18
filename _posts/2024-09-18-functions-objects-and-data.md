---
layout: post
author: kormang
---


(This section requires understanding code in different languages to fully understand the concepts. Detailed knowledge of those languages is not required, just basic reading and understanding of the code.)

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

In JavaScript (and TypeScript) functions, are literally objects, with prototype `Function`, and those objects can be called, either using regular call syntax, or calling method `call` on them.

This is not only characteristic of JavaScript.
The same thing is in Python, functions are objects with `__call__` method. In Python, one can even make its own class with `__call__` method, and call the instance of the class as if it is regular function.

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

Hopefully, we've seen how functions and callable objects can be used interchangeably in different scenarios and languages. All these languages have different rules, limitations, and implementations when it comes to using functions as objects. However, what is common is that semantically, functions are objects and can, one way or another, be used interchangeably.

But this is not enough to say that functions are, in fact, objects. For functions to be fully recognized as objects, they need to be able to hold and update internal state. Certainly, in C, a function is just a place in memory where a sequence of machine instructions is stored, without any state, but C is really low level language, that lacks many of higher order constructs. This is not a critique of C, but it does not have objects at all, and is really close to assembly, so it's not that relevant for our talk here. However, certain patterns we see here can be used to better reason about code, and to design it better, even if it is written in C.

Let's see how closure functions can be used to implement stateful object, in TypeScript.

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

As you can assume, the reason we use outer and inner function, is that the outer function serves as a constructor for our closure. Of course, constructors are not always as explicit as in this case, but constructors always do exist.

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

As we can see, `struct ClosureData` is used to store data that will later be used inside equivalent of the `run` function. To support that, we pass additional parameter to it, since in C the function is not a member of a class. The parameter `arg` emulates, `this` keyword in object oriented languages. Since, we don't have templates and generics, `pthread` expects function that accepts `void*` as argument, and later inside of the function we cast it to our struct, to get the data we need.

Here is one way to implement it in Python that illustrates the concept we're trying to understand.

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

We can, and in fact, we should, parameterize our objects and functions with other behavior, not just with data.

Let's say we have a video hosting platform, where people can upload their videos, and those who are subscribed can get notification about new video being uploaded (its not YouTube).

When new video gets uploaded, an event is enqueued into a queue. Then different event handlers, or queue consumers, which are subscribed to specific event get notified about the new video. Each does its own thing, but some things are common to all of them. For example, each consumer should validate data, as the event is send over the network as it not to be trusted. This can be optimized so that it is validated only once per process, but will serve us as and example anyway.

Let's see how generalized consumer for 'video_published' event could look like.

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
  new NotifySubscribersConsumer(getSubsFinder(), getEmailService())
)
```

In this example, we use some subscribers finder to find subscribers for the author of the video, and email service to deliver notifications via email. We can also create another consumer that would send push notifications.

```typescript
new VideoPublishedConsumer(
  queueConnection,
  new NotifySubscribersConsumer(getSubsFinder(), getPushNotifService())
)
```

(This is not the most optimal way to implement it, but serves as an example for parameterizing with behavior)

Similarly we can create other consumers.

```typescript
new VideoPublishedConsumer(
  queueConnection,
  new UpdateStatsConsumer(getStatsService())
)
```

We can use parameters, that are not meant to be state, in order to change behavior of certain class. Some of those parameters are behavior themselves. In class `NotifySubscribersConsumer` we have `percentage` parameter, that also changes the behavior of the class, but it is data that changes behavior. If it is less then 1, only specified percentage of randomly sampled subscribers will get notified (it is left to reader to imagine why would anyone on Earth want that).

Both `VideoPublishedConsumer` and `NotifySubscribersConsumer` (`UpdateStatsConsumer` too, but the code is not presented) have parameters that are not data, and certainly not state, but rather behavior.

Now, we will see how that looks like in more pure form of behavior - functions.

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

## State

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

Object Oriented Programming (OOP) promises encapsulation, and loose coupling, but almost every course on OOP teaches us about class `Vehicle`, and subclass `Car` that has properties like `model`, `engine` and so on. Also, we've been thought to make getters, and setters, to encapsulate those fields so that someone from the outside can't access those fields directly. But the truth is - it doesn't help. Yes, we can put some logic in setter to make sure that the value is valid, or to do some other necessary logic on setting the value. But we've still exposed that field, and other code depends on that field, and expects that it can get value after setting it.

Later it drives us towards API design that requires client to "always set certain value, before calling certain methods". That in its turn, leads us to nasty type of bugs like "Oh, I've forgot to return old value, after calling that method", or "that value has been set by someone in the mean time, before I got chance to call the method".

These classes of bugs are eliminated when using pure functional programming, which implies that no state is mutable, or that all data is immutable - once initialized it can not change anymore. It is however not always practical, and often much more efficient to have mutable variables, arrays, etc. Mutability is often more efficient when we talk about local data, local state, local variables, in the "micro world". But in "macro world", when we talk about scale of modules, and large programs, keeping track of who changed what state is difficult and leads to mentioned bugs. So the more state is localized, and hidden the better, and if we can have immutable objects (like `String` class in many languages, like Python, JavaScript, and Java) even better. If having immutable object is impractical or inefficient, than at least we should hide it, as if it is local state of the closure function, and definitely avoid getters and setters.

Basically, functions are objects, and as it seems objects are also just functions?

Well, yes and no. Most of the objects are just behavior, or should be just behavior, thus should behave like closure functions.

So why do we even use objects, why don't we always use closure functions?

First, there are objects that are not behavior.
Then there are functions that are often used together, for example `addEventListener`, `removeEventListener`. In such case it is easier to implement it as object. We can implement it like this.

```javascript
function createEventEmitter() {
  const handlers = {}
  return {
    addEventListener(eventName, listener) {
      /* add listener to handlers */
    }
    removeEventListener(eventName, listener) {
      /* remove listener from handlers */
    }
    // other methods
  }
}
```

But, here we have get to the third reason not to use functions all the time. We certainly can use functions always, and that approach has some advantages due to that uniformity and isomorphism. Certainly, LISP, has everything represented as lists, and that feature of LISP, called homoiconicity, makes it extremely flexible and expressive, but at the same time it is hard for humans to read, exactly because everything looks the same. It seems that we humans need certain marks in the code to be able to more easily identify the meaning of it. When we see words like `class` and `constructor` we better and more instantaneous identify the meaning of the code compared to seeing word `function` (and the `function` is in its turn "easier" on the eyes than reading code that consists exclusively of lists). We need to analyze that structure of the `function` to recognize the pattern it represents. Also depending on the language it might be impossible or impractical to create object with methods as shown in the example above. So both approaches have their pros and cons.

Let's now return to the first point - not all objects represent behavior. Some objects are explicitly made to hold data. Such objects behave like `struct`s in C language (have just properties, no methods), or have getter, setters, and few other helper function that do not mutate state (e.g. `toString`, or comparison methods, `equals` or overloaded operator `==`). Sometimes they are called Data Transfer Objects (DTOs), or instances of data classes. It is good practice to treat them as immutable **values**. Good example of that pattern are data classes in Kotlin language.

There is also third type of objects, that are explicitly made to hold mutable state. Those objects are data structures. For example, list, hash table, and so on. Their whole purpose is to have mutable state. They however, don't have getters and setters for individual fields.


## Conclusion

We have seen that functions are just callable objects.

We have seen that objects can belong to one of the three categories, and that we should avoid objects with setters.

The three categories are:

1. Behavior objects - basically functions, or group of functions. They better be immutable, but even if they have mutable state, it should be hidden and obscure the same way internal state of closure is hidden and obscure. They can be parameterized with some data and other behavior that affects the behavior of instantiated object, and that data and behavior can be saved as local immutable state of the object/closure.
2. Data objects - like `struct`s in C, but can have methods like `toString`, `equals`, etc. Usually treated as immutable values. A good example is data classes in Kotlin.
3. Data structures - their whole purpose is to hold and manage mutable state in an efficient way (e.g. list, vector, hash table, and so on).

