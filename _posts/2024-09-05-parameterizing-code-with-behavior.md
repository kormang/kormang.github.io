---
layout: post
author: kormang
---


There are few ways to parameterize generalized code.

* If the language we are using supports generics or templates (like C++), without type erasure (which means unlike Java, TypeScript), then we can create template class or function, and pass in concrete generic parameters to make the class or function concrete.
* Overriding methods from superclass, or interface.
* Pass the parameters that change behavior as call arguments to function.
* Enable certain parameters that change behavior to be passed to constructor - dependency injection.

## Generics

Generics or templates, often used to implement data structure classes (like lists, trees, etc), can also be used to parameterize behaviors. That is why is is also called *parametric polymorphism*.

Here is an example:

```c++
// Custom allocator class
template <typename T>
struct MyAllocator {
    using value_type = T;

    MyAllocator() noexcept {}

    template <typename U>
    MyAllocator(const MyAllocator<U>&) noexcept {}

    T* allocate(std::size_t n) {
        return my_alloc<T>(n);
    }

    void deallocate(T* p, std::size_t n) noexcept {
        my_free(p);
    }
};

int main() {
    // Create a vector using the custom allocator
    std::vector<int, MyAllocator<int>> customVector;

    for (int i = 0; i < 5; ++i) {
        customVector.push_back(i);
    }

    // Print the elements in the vector
    for (const int& value : customVector) {
        std::cout << value << " ";
    }

    return 0;
}
```

Here, we have vector class, that is written in such way that it can be used with different allocation mechanism for its elements. That makes is pretty generalized, and responsibility for allocation and deallocation of memory is delegated away from the vector class. The reason why it is done using template parameter (*parametric polymorphism*), instead of runtime polymorphism is performance.

In languages like Java and TypeScript such things are not possible, nor does it make sense. In TypeScript, when it gets compiled to JavaScript all types are erased and we always rely on runtime polymorphism.

Still this is just illustration of the way units of code and their behavior can be parameterized.

## Override method from superclass

Although it is considered better to use composition over inheritance, overriding methods from super class is another way to parameterize generalized (abstract) code.

```typescript
abstract class BankAccount {
  protected abstract performTransaction(): void;

    public executeTransaction(): void {
      // Public method that uses the protected method
      this.performPreTransactionSteps();
      this.performTransaction();
      this.performPostTransactionSteps();
    }
}

class SavingsAccount extends BankAccount {
    protected performTransaction(): void {
        // Savings account-specific logic
    }
}

class CheckingAccount extends BankAccount {
    protected performTransaction(): void {
        // Checking account-specific logic
    }
}
```

Here, again we have a way to write generalized (abstract) code that can be parameterized in some way to do concrete thing, thus reusing generalized code, and reducing duplication. Although in this case, word *parameterized* can be hardly used.

## Call arguments

This is used often, we'll give few examples.

```typescript
const numbers = [5, 1, 9, 3, 7];

numbers.sort((a, b) => b - a);
```

Here, `sort` method is written in such way that it can sort elements in any order, given that the client code provides necessary comparison function.

Let's see another example.

```typescript
const numbers = [10, 25, 5, 40, 15, 8, 30];

const oddNumbers = numbers.filter((num) => num % 2 !== 0);

const threshold = 20;
const numbersAboveThreshold = numbers.filter((num) => num > threshold);
```

Notice here, that second predicate function can be written in a generalized way like this:

```typescript
function isAboveThreshold(threshold) {
  return (num) => num > threshold
}

const numbersAboveThreshold = numbers.filter(isAboveThreshold(20));
```

Notice that we don't use `threshold` as a parameter to the predicate function, but as a wrapper function, that returns the predicate function. This wrapper function behaves like constructor for the predicate functions. This is necessary, because predicate function needs to conform the certain interface, namely to accept only one element, and return boolean. So we need another place, or make another level of indirection, to be able to write it in a generalized way. This is important detail to be aware of.

Here is another example. We want to create an Express.js middleware to check if the request contains the "X-Data" header with the value "positive" and return a 400 error if it's not present or doesn't match the expected value.

```javascript
// Custom middleware to check the "X-Data" header
const checkXDataHeader = (req, res, next) => {
  const xDataHeader = req.get('X-Data');

  if (!xDataHeader || xDataHeader !== 'positive') {
    return res.status(400).json({ error: 'Invalid or missing X-Data header' });
  }

  // If the header is valid, continue to the next middleware or route
  next();
};


// Use the custom middleware for specific routes
app.get('/protectedRoute', checkXDataHeader, (req, res) => {
  res.json({ message: 'You have access to this protected route!' });
});
```

Let's say we need to write such middlewares often. We might want to do this.

```javascript
const checkHeaderValue = (headerName, value) => (req, res, next) => {
  const headerValue = req.get(headerName);

  if (!headerValue || headerValue !== value) {
    return res.status(400).json({ error: `Invalid or missing ${headerName} header` });
  }

  // If the header is valid, continue to the next middleware or route
  next();
};

```

Now we can use it like this:

```javascript
// Use the custom middleware for specific routes
app.get('/protectedRoute', checkHeaderValue('X-Data', 'positive'), (req, res) => {
  res.json({ message: 'You have access to this protected route!' });
});
```

Again, we need another layer, that represents "constructor" for the middleware, in order to make it generally reusable.

### Optional: Readability when passing behavior as call arguments

Additionally, let's have a short discussion about readability of code that uses generalized functions.

We've seen that generalized functions could be hard to read and understand, but the code that uses them is often very clean. However, not always. Let's look at few examples.

Consider you have two arrays, A and B. Array A contains objects of shape `{x: number, y: number}`, and array B
contains object of shape `{z: number, y: number}`.

Now, can you guess what this function does? Give yourself no more than a minute, and move to the next one, maybe it is simpler.

```javascript
function processAB1(A, B) {
  const collection1 = []
  const collection2 = []
  const collection3 = []

  for (let i = 0; i < A.length; ++i) {
    let foundIndex = -1;
    for (j = 0; j < B.length; ++j) {
      if (A[i].y === B[j].y) {
        foundIndex = j;
        break;
      }
    }
    if (foundIndex > -1) {
      collection1.push({x: A[i].x, y: A[i].y, z: B[foundIndex].z });
    } else {
      collection2.push(A[i])
    }
  }

  for (let i = 0; i < B.length; ++i) {
    let foundIndex = -1;
    for (j = 0; j < collection1.length; ++j) {
      if (B[i].y === collection1[j].y) {
        foundIndex = j;
        break;
      }
    }
    if (foundIndex === -1) {
      collection3.push(B[i])
    }
  }

  return {collection1, collection2, collection3}
}
```

That is a lot of code. It isn't complicated, but it is a lot of code to read and try to understand.

Is the next version easier to understand?

```javascript
function processAB2(A, B) {
  const merge = (a, b) => b ? ({x: a.x, y: a.y, z: b.z }) : a
  const merged = A.map(a => merge(a, B.find(b => a.y === b.y)))

  const collection1 = merged.filter(e => 'z' in e)
  const collection2 = merged.filter(e => !('z' in e))
  const collection3 = B.filter(b => A.find(a => a.y === b.y) === undefined)

  return {collection1, collection2, collection3}
}
```

That is really compact. It is three times shorter than the previous one. But the code is really cryptic. It relies heavily on `map` and `filter`. Although in previous example using `filter` made code a bit easier to understand than `for` loop, here it is not so obvious. The reasons are:

- code density - computation of `merged` array contains a lot of logic in a single line, or two.
- dependencies on previous steps without steps being named - you as a reader, have to keep the results of one step in your head and process it in next step, and then again keep the intermediate results in your head till the end.

Maybe the third version is easier to understand?

```javascript
function differenceOfSets(X, Y, equals) {
  return X.filter(x => Y.find(y => equals(x, y)) === undefined)
}

function intersectionOfSets(X, Y, equals) {
  const xIntersection = []
  const yIntersection = []
  X.forEach(x => {
    const y = Y.find(y => equals(x, y))
    if (y !== undefined) {
      xIntersection.push(x)
      yIntersection.push(y)
    }
  })

  return [aIntersection, bIntersection]
}

function ABEquals(a, b) {
  return a.y === b.y
}

function processAB3(A, B) {
  const [aIntersect, bIntersect] = intersectionOfSets(A, B, ABEquals)

  const collection1 = aIntersect.map((a, i) => ({x: a.x, y: a.y, z: bIntersect[i].z }))
  const collection2 = differenceOfSets(A, B, ABEquals)
  const collection3 = differenceOfSets(B, A, ABEquals)

  return {collection1, collection2, collection3}
}

```

The third version a bit shorter then first one, and it is more readable. Although the `processAB3` function itself is as short as the second one, there are more additional functions which together add up to roughly same amount of code as the first version. It is separated into clear, distinct steps, each step is named, names are given by helper functions. It would be even more readable if JavaScript had `zip` function to use instead of `map`. Another nice thing about it the we have created ourselves three reusable functions (`differenceOfSets`, `intersectionOfSets`, `ABEquals`), and at least two of them are useful outside of this task. It is however, less efficient than the first one, which computes `collection1` and `collection2` in one go, while third version takes 3 steps for the same task.

In this particular case, if we value concise code, and reusability, we should take the third approach. If we want to have, maybe not so easily readable, but definitely understandable code, yet efficient code, we should take the first one. In fact, in that case we can definitely make it a bit shorter by using `find` or `findIndex` instead of inner loop. Also the second loop, to calculate `collection3` is hardly any better in terms of both readability and performance then `filter` + `find` as demonstrated in `differenceOfSets`. So some combination, would probably have the best of both worlds:

```javascript
function processAB4(A, B) {
  const collection1 = []
  const collection2 = []

  A.forEach(a => {
    const b = B.find(b => a.y === b.y)
    const aHasMatchInB = b !== undefined
    if (aHasMatchInB) {
      collection1.push({x: a.x, y: a.y, z: b.z });
    } else {
      collection2.push(a)
    }
  })

  const notInCollection1 = b => collection1.find(a => a.y === b.y) === undefined
  const collection3 = B.filter(notInCollection1)

  return {collection1, collection2, collection3}
}
```

It is probably the most readable version and also concise, and performant enough.
Notice how extracting `aHasMatchInB` and `notInCollection1` into separate, named, constants helps with readability. But again, some people might find other versions more readable and better, it is matter of taste, sometimes.


### Dependency injection through constructor

Dependency injection through constructor is a pattern that helps us implement one of the SOLID principles, namely, the Dependency Inversion principle.

Let's say that we want to create Java Servlet filter to check if the request contains the "X-Data" header with the value "positive" and return a 400 error if it's not present or doesn't match the expected value (Java Servlet analog of the Express.js middleware example above).

Hopefully, code is understandable enough even if you don't know Java, but you know TypeScript.

```java
public class XDataHeaderFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String xDataHeader = request.getHeader("X-Data");

        if (xDataHeader == null || !xDataHeader.equals("positive")) {
            response.setStatus(400);
            response.getWriter().write("Invalid or missing X-Data header");
            return;
        }

        // If the header is valid, continue with the request
        chain.doFilter(request, response);
    }

    // More code that is actually irrelevant, and in fact, we could go without it if Servlets weren't over-engineered.
}

```

Again, if need for such filter occurs frequently, we might want to create generalized, reusable filter like this.

```java
public class CustomHeaderFilter implements Filter {
    private String headerName;
    private String headerValue;

    public CustomHeaderFilter(String headerName, String headerValue) {
        this.headerName = headerName;
        this.headerValue = headerValue;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String value = request.getHeader(headerName);

        if (value == null || !value.equals(this.headerValue)) {
            response.setStatus(400);
            response.getWriter().write("Invalid or missing " + headerName + " header");
            return;
        }

        // If the header is valid, continue with the request
        chain.doFilter(request, response);
    }

    // More code that is actually irrelevant, and in fact, we could go without it if Servlets weren't over-engineered.
}
```

Now, we can use it like this:

```java
public class FilterSetup {
    public void configureFilter(ServletContext servletContext) {
        FilterRegistration.Dynamic filter = servletContext.addFilter("CustomHeaderFilter", new CustomHeaderFilter("X-Data", "positive"));
        String[] urlPatterns = {"/protectedRoute"};
        filter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, urlPatterns);
    }
}
```

As with the Express.js example above, `XDataHeaderFilter` has to implement certain interface, to be used by the parts of the system that use filters, so we can't add more parameters to `doFilter` method, and instead we parameterize it through constructor.

We can now go one step further, and compare not just the value of the header with the provided value, but check if it matches certain condition using predicate.

```java
public class CustomHeaderFilter implements Filter {
    private String headerName;
    private Predicate<String> headerValidator;

    public CustomHeaderFilter(String headerName, Predicate<String> headerValidator) {
        this.headerName = headerName;
        this.headerValidator = headerValidator;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String headerValue = request.getHeader(headerName);

        if (headerValue == null || !headerValidator.test(headerValue)) {
            response.setStatus(400);
            response.getWriter().write("Invalid or missing " + headerName + " header");
            return;
        }

        // If the header is valid, continue with the request
        chain.doFilter(request, response);
    }
}

```

We can also check if in the SQL table there is row with ID matching header value.

```java

class DataExistsPredicate implements Predicate<String> {
    private final DataSource dataSource;

    public DataExistsPredicate(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public boolean test(String headerValue) {
        try (Connection connection = dataSource.getConnection()) {
            String query = "SELECT 1 FROM data WHERE id = ?";
            try (PreparedStatement statement = connection.prepareStatement(query)) {
                statement.setString(1, headerValue);
                try (ResultSet resultSet = statement.executeQuery()) {
                    return resultSet.next(); // Row exists if there is a result
                }
            }
        } catch (SQLException e) {
            // Handle database errors, e.g., log or throw an exception
            e.printStackTrace();
        }
        return false; // Default to false if there's an error
    }
}


public class CustomHeaderFilter implements Filter {
    private String headerName;
    private Predicate<String> headerValidator;

    public CustomHeaderFilter(String headerName, Predicate<String> headerValidator) {
        this.headerName = headerName;
        this.headerValidator = headerValidator;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String headerValue = request.getHeader(headerName);

        if (headerValue == null || !headerValidator.test(headerValue)) {
            response.setStatus(400);
            response.getWriter().write("Invalid or missing " + headerName + " header");
            return;
        }

        // If the header is valid, continue with the request
        chain.doFilter(request, response);
    }
}

public class FilterSetup {
    private ServletContext servletContext;
    String filterName;
    private Filter filter;

    public FilterSetup(ServletContext servletContext, String filterName, Filter filter) {
        this.servletContext = servletContext;
        this.filterName = filterName;
        this.filter = filter;
    }

    public void configureFilter() {
        // Register the filter
        FilterRegistration.Dynamic filter = servletContext.addFilter(this.filterName, this.filter);

        // Specify the URL patterns for the filter
        String[] urlPatterns = {"/protectedRoute"};
        filter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, urlPatterns);
    }
}

```

Now we have `FilterSetup`, which has dependency on filter, and this particular implementation of filter has dependency on predicate, and this particular implementation of predicate has dependency on data source.

So normal instantiation of `FilterSetup` might look like this:

```java
new FilterSetup(
  getServletContext(),
  "CustomFilterName",
  new CustomHeaderFilter(
    "X-Data",
    new DataExistsPredicate(
      new DataSource(
        new DBConnectionPool(
          config.db_url,
          config.db_username,
          config.db_password
        )
      )
    )))
```

We can go on and on with this dependency chain, but it already looks tedious. In real world, dependencies can not only create long chains, but complex [direct acyclic graphs](https://en.wikipedia.org/wiki/Directed_acyclic_graph).

Let's see pretty complex example. It is made up, but examples like this are seen often in the real world.

```typescript
const database = new DatabaseService("mongodb://localhost:27017/mydb");
const logger = new Logger("app.log");
const errorHandler = new ErrorHandler(logger);
const emailService = new EmailService("smtp.example.com");
const userService = new UserService(database, emailService);
const notificationService = new NotificationService(emailService, logger);
const analyticsService = new AnalyticsService(database, logger);
const authorizationService = new AuthorizationService(database, userService);
const paymentService = new PaymentService(database, notificationService, errorHandler);
const reportingService = new ReportingService(analyticsService, logger);
const authService = new AuthService(database, userService, logger, notificationService, authorizationService, paymentService);
const userProfileService = new UserProfileService(database, userService, analyticsService, reportingService);
const webApp = new WebApp(authService, userProfileService, errorHandler);
```

or as a single statement

```typescript
const webApp = new WebApp(
    new AuthService(
        new DatabaseService("mongodb://localhost:27017/mydb"),
        new UserService(
            new DatabaseService("mongodb://localhost:27017/mydb"),
            new EmailService("smtp.example.com")
        ),
        new Logger("app.log"),
        new NotificationService(
            new EmailService("smtp.example.com"),
            new Logger("app.log")
        ),
        new AuthorizationService(
            new DatabaseService("mongodb://localhost:27017/mydb"),
            new UserService(
                new DatabaseService("mongodb://localhost:27017/mydb"),
                new EmailService("smtp.example.com")
            )
        ),
        new PaymentService(
            new DatabaseService("mongodb://localhost:27017/mydb"),
            new NotificationService(
                new EmailService("smtp.example.com"),
                new Logger("app.log")
            ),
            new ErrorHandler(new Logger("app.log"))
        )
    ),
    new UserProfileService(
        new DatabaseService("mongodb://localhost:27017/mydb"),
        new UserService(
            new DatabaseService("mongodb://localhost:27017/mydb"),
            new EmailService("smtp.example.com")
        ),
        new AnalyticsService(
            new DatabaseService("mongodb://localhost:27017/mydb"),
            new Logger("app.log")
        ),
        new ReportingService(
            new AnalyticsService(
                new DatabaseService("mongodb://localhost:27017/mydb"),
                new Logger("app.log")
            ),
            new Logger("app.log")
        )
    ),
    new ErrorHandler(new Logger("app.log"))
);

```

(Did you even notice that it is not Java but TypeScript?)

Now we see the problem that arises from dependency injection, and this is not all of it. Sometimes we want singleton instances, sometimes per-request instances, and so on.

This does not mean we should give up on dependency injection. It solves one important problem - making code reusable by parameterizing it. In the example above, we don't want to tightly couple `ReportingService` to a specific `AnalyticsService` or logging because in another place, we might want to use different implementations. Maybe you'll never need more than one instance of `UserService`, for example, but still, you'd like to easily replace parts of it when business needs require it. Also, at some point, you might want to reuse parts of `UserService` or some other service, and you thought you'd never going to need this, but as it turns out, that would be the most desirable way to implement some new request from the client.

You might think - OK, I need `UserProfileService` as a dependency, and I'll never want another user profile service, and even if I do, I'll just change that one line in my WebApp constructor where I instantiate the `UserProfileService`. This way of thinking has a few problems.
  * We are usually wrong with such assumptions.
  * Often, with such an approach, people tend to write code inside (in this example) `WebApp` so that it depends on the specifics of the current implementation of `UserProfileService`, rather than the interface. So, using dependency injection can prevent development from going in the right direction. Dependency injection can protect the codebase, if not from you, who are aware of the problem, then from your colleagues, who see that you're depending on a specific implementation instantiated in the constructor and automatically assume the wrong things (namely that that can use methods or other specifics of the implementation, rather then interface).
  * We're pushing the development in the wrong direction. If this becomes the norm, only when people are highly aware of the dependency injection pattern and the dependency inversion principle will they refactor it properly when the need to reuse parts of the code arises. Otherwise, the code might not even be refactored at all and may be duplicated instead.
  * Without a "dependency injection culture" people may not even tend to extract parts of the logic into separate units of code (classes). They might consciously or unconsciously think, "What is the purpose of extracting it to another class if we're going to instantiate it right here in the constructor? Wouldn't it be easier to just write that code right here in this class?" So what we end up with is a hopelessly tightly coupled mix of responsibilities.
  * We don't always need to know how to get an instance of a dependency. For example, when we write code for the `WebApp` constructor, we don't need to know how to get an instance of `UserProfileService`. It will be passed to us through the constructor, so we delegate responsibility for obtaining an instance of `UserProfileService` to other parts of the code. Additionally, in some cases, it also enables us to easily switch between singleton instances to per-request instances (in the case of backend software) or instances with different lifespans.

Even if you don't want to think about architecture, but you would like to have a good one, as a general rule of thumb, just pass things as dependencies to the constructor whenever you can. You will deal with instantiation later, somehow.

There are, of course, legitimate exceptions. For example, we need to instantiate some `set` like data structure in the constructor, and later use it in other methods of the class. This is perfectly normal. We might say that this `set` is implementation detail nobody should care about outside of the class. Dependencies are replaceable/configurable components, and dependency injection helps us not to depend on concretions and easily combine and swap-in different implementations.

One way to help you deal with dependency injection is to write getter or factory function that return dependencies, for example

```typescript
function getReportingService() {
  return new ReportingService(getAnalyticsService())
}
```

 But this solves only part of the problem.

 Usually people use dependency injection containers. For example, for backend TypeScript there is NestJS' implementation. For frontend there is Angular's DI. For Java, the most popular is Spring Framework and Google Guice. For Android we have Dagger and Yatagan. For C++ Boost.DI.

 Most of these libraries or frameworks use reflection to understand what dependencies certain classes have and to build a dependency graph. This is not possible for languages that have no or limited support for reflection, like C++. Some analyze code at compile time and generate instantiation code, while others allow us to specify dependencies in separate configuration files. This is a complex topic, and we should use DI frameworks only when things get complicated, or if we're already familiar with some of them, or if our framework for whatever we're trying to build (backend, frontend, mobile app) has one built in. Just remember to avoid hardwiring dependencies. If you decide to write your own solution because things are not too complex, make sure that you avoid reinventing the wheel and use a well-known and tested solution when things do get complex (unless no well-known solution meets your demands, which is more probable in more complex languages like C++ than with TypeScript or Java).

Another way to deal with dependency injection is Factory pattern.

```typescript
class ServiceDFactory {
    create(aOrC = 'a') {
        // Create and configure dependencies
        const depB = new DependencyB();

        // Create services with dependencies
        this.serviceB = new ServiceB(depB);

        const serviceAC = aOrC == 'c'
          ? new ServiceC(new DependencyC())
          : new ServiceA(new DependencyA());

        return new ServiceD(this.serviceB, serviceAC);
    }
}

```

Factories can also be used in combination with DI frameworks, and as dependencies themselves.

Similarly, we can use Facade pattern. Basically, components that accept injected dependencies, together with those dependencies, form a complex graph, and not everybody needs to know how to put all those peaces together.

We will not get into detail about Facade pattern, but we can take a look at simple example to illustrate the technique:

```typescript
class ServiceFacade {
    private serviceA: ServiceA;
    private serviceB: ServiceB;
    private serviceD: ServiceD;

    constructor(aOrC = 'a') {
        // Create and configure dependencies
        const depA = new DependencyA();
        const depB = new DependencyB();

        // Create services with dependencies
        this.serviceA = new ServiceA(depA)
        this.serviceB = new ServiceB(depB);

        const serviceAC = aOrC == 'c'
          ? new ServiceC(new DependencyC())
          : new ServiceA(depA);

        this.serviceD = new ServiceD(this.serviceB, serviceAC);
    }

    computeSomething() {
        // Encapsulate the complex interactions of services here
        const aResult = this.serviceA.methodA();
        return this.serviceD.methodF(aResult);
    }
}

```

We can have one big, or multiple smaller Facades like that one.

There are other solutions similar to Facade. For example, in tensorflow with keras, a framework for training deep neural networks, we can configure and then compile neural network. To do that we need to specify, so called, loss function we want to use, and optimizer algorithm.

```python
import tensorflow as tf
...
model.compile(ts.keras.losses.SparseCategoricalCrossentropy(),
              optimizer=tf.keras.optimizers.SGD(learning_rate=1e-5, momentum=0.9))
```

But, we can also go with simple variant like this:

```python
import tensorflow as tf
...
model.compile(loss='sparse_categorical_crossentropy', optimizer='sgd')
```

Of course, if we implement our own loss function, or our own optimizer, then we must pass in corresponding objects. But for well known built-in implementations, we can simply specify names as strings.

### Conclusion on dependency injection

Dependency injection is a powerful pattern that can help us write better and more reusable code. Even if we instantiate concrete implementations one level above the code that uses those instances (for example, we instantiate concrete instances and pass them to the constructor of the class immediately) it is still much better then instantiating concrete implementation in the place where they are used. Often, it is enough.

We should take our time, and think what to extract and where to instantiate certain implementations, and where to pass them. This time is not wasted, it will pay off.

Because people tend not to do it, and generally it is desirable to do it in most cases, if we are overwhelmed, and can't think about every detail, just extract everything that makes sense, and accept it through constructor. Then if we are lazy to think about where it should be instantiated, make it at least inside some factory function, in the place where the constructor is called (for example). No need to complicate, no need for anything special, just inject dependencies using some of the approaches presented here. When we have time, we should think more carefully where exactly to instantiate those dependencies and how to pass them to the place where they are used.

> Note that there is no rule that is applicable everywhere, and of course there are exceptions to every rule and best practice.

