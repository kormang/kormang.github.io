---
layout: post
author: kormang
---

There are a few ways to parameterize generalized code:

* If the language supports generics or templates (like C++), without type erasure (unlike Java or TypeScript), we can create template classes or functions and pass in concrete generic parameters to make the class or function concrete.
* Overriding methods from a superclass or interface.
* Passing parameters that change behavior as arguments to functions.
* Enabling certain parameters that alter behavior to be passed to the constructor—this is known as dependency injection.

## Generics
Generics or templates, often used to implement data structure classes (like lists, trees, etc.), can also be employed to parameterize behaviors. This is why it is also called *parametric polymorphism*.

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

Here, we have a vector class designed to work with different allocation mechanisms for its elements. This makes it quite generalized, as the responsibility for memory allocation and deallocation is delegated away from the vector class. The choice to use a template parameter (*parametric polymorphism*) instead of runtime polymorphism is primarily for performance reasons.

In languages like Java and TypeScript, such implementations are not possible, nor do they make sense. In TypeScript, when the code is compiled to JavaScript, all types are erased, and we always rely on runtime polymorphism.

Still, this serves as an illustration of how units of code and their behavior can be parameterized.

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

Notice that we don't use `threshold` as a parameter to the predicate function; instead, we use a wrapper function that returns the predicate function. This wrapper function acts like a constructor for the predicate functions. This approach is necessary because the predicate function needs to conform to a specific interface: it must accept only one element and return a boolean. Therefore, we require another level of indirection to write it in a generalized way. This is an important detail to be aware of.

Here’s another example: we want to create an Express.js middleware to check if the request contains the "X-Data" header with the value "positive" and return a 400 error if it's not present or doesn't match the expected value.

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

### Dependency Injection Through Constructor

Dependency injection through the constructor is a pattern that helps us implement one of the SOLID principles, specifically the Dependency Inversion Principle.

Let's say we want to create a Java Servlet filter to check if the request contains the "X-Data" header with the value "positive" and return a 400 error if it's not present or doesn't match the expected value (the Java Servlet analog of the Express.js middleware example above).

Hopefully, the code is understandable enough even if you don't know Java but are familiar with TypeScript.

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

As with the Express.js example above, `XDataHeaderFilter` must implement a certain interface to be used by the parts of the system that handle filters. This means we can't add more parameters to the `doFilter` method; instead, we parameterize it through the constructor.

We can now take it a step further and not only compare the value of the header with the provided value but also check if it matches a certain condition using a predicate.

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

Now we encounter the problems that can arise from dependency injection, and this isn’t all of it. Sometimes we want singleton instances, other times per-request instances, and so on.

This doesn’t mean we should abandon dependency injection. It addresses an important issue: making code reusable by parameterizing it. In the example above, we don’t want to tightly couple `ReportingService` to a specific `AnalyticsService` or logging implementation because, in other contexts, we might want to use different implementations. You might never need more than one instance of `UserService`, for example, but you still want the flexibility to replace parts of it when business needs change. Additionally, there may come a time when you want to reuse parts of `UserService` or some other service, even if you initially thought that wouldn't be necessary.

You might think, "OK, I need `UserProfileService` as a dependency, and I'll never want another user profile service. Even if I do, I'll just change that one line in my WebApp constructor where I instantiate the `UserProfileService`." However, this way of thinking has a few problems:

* We are often mistaken in such assumptions.

* With this approach, developers tend to write code inside the `WebApp` that depends on the specifics of the current implementation of `UserProfileService`, rather than the interface. Dependency injection can prevent development from going in the wrong direction. It can protect the codebase from colleagues who may mistakenly assume they can use methods or other specifics of the implementation instantiated in the constructor, just because you have used concrete implementation in the constructor.

* This can push development in the wrong direction. If this becomes the norm, only those who are highly aware of the dependency injection pattern and the dependency inversion principle will refactor the code properly when the need for reuse arises. Otherwise, the code might not be refactored at all and could end up duplicated.

* Without a "dependency injection culture," people may hesitate to extract parts of the logic into separate units of code (classes). They might consciously or unconsciously think, "What’s the point of extracting it to another class if we’re just going to instantiate it right here in the constructor? Wouldn’t it be easier to write that code directly in this class?" This leads to a tightly coupled mix of responsibilities.

* We don’t always need to know how to obtain an instance of a dependency. For instance, when writing code for the `WebApp` constructor, we don’t need to know how to get an instance of `UserProfileService`; it will be provided through the constructor. This allows us to delegate the responsibility for obtaining an instance of `UserProfileService` to other parts of the code. Moreover, in some cases, it enables easy switching between singleton instances and per-request instances (in the case of backend software) or instances with different lifespans.

Even if you don't want to think about architecture, but you would like to have a good one, as a general rule of thumb, just pass things as dependencies to the constructor whenever you can. You will deal with instantiation later, somehow.

There are, of course, legitimate exceptions. For example, we need to instantiate some `set` like data structure in the constructor, and later use it in other methods of the class. This is perfectly normal. We might say that this `set` is implementation detail nobody should care about outside of the class. Dependencies are replaceable/configurable components, and dependency injection helps us not to depend on concretions and easily combine and swap-in different implementations.

One way to help you deal with dependency injection is to write getter or factory function that return dependencies, for example

```typescript
function getReportingService() {
  return new ReportingService(getAnalyticsService())
}
```

But this only solves part of the problem.

Typically, people use dependency injection containers. For example, in backend TypeScript, there’s NestJS; for frontend, Angular provides its own DI system. In Java, the most popular frameworks are Spring and Google Guice. For Android, there’s Dagger and Yatagan. In C++, you might use Boost.DI.

Most of these libraries or frameworks utilize reflection to determine the dependencies of certain classes and build a dependency graph. This approach isn’t feasible in languages that have no or limited support for reflection, like C++. Some frameworks analyze code at compile time and generate instantiation code, while others allow us to specify dependencies in separate configuration files. This is a complex topic, and we should use DI frameworks only when things get complicated, or if we're already familiar with a particular framework, or if our chosen framework (for backend, frontend, or mobile app development) has one built in. It’s crucial to avoid hardwiring dependencies. If you decide to write your own solution because the situation isn’t overly complex, ensure you don’t reinvent the wheel; rely on a well-known and tested solution when complexity increases—unless you have specific needs that no established solution can meet, which is more likely in more complex languages like C++ than in TypeScript or Java.

Another approach to handling dependency injection is the Factory pattern.

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

Factories can also be used in conjunction with DI frameworks, and they can serve as dependencies themselves.

Similarly, we can employ the Facade pattern. Essentially, components that accept injected dependencies, along with those dependencies, create a complex graph, and not everyone needs to understand how to assemble all those pieces.

We won't delve into the details of the Facade pattern, but we can look at a simple example to illustrate the technique:

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

We can have one large facade or multiple smaller facades like the one mentioned.

There are other solutions similar to the Facade pattern. For example, in TensorFlow with Keras, a framework for training deep neural networks, we can configure and compile a neural network. To do this, we need to specify a loss function and an optimizer algorithm.

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

Dependency injection is a powerful pattern that can help us write better, more reusable code. Even if we instantiate concrete implementations just one level above the code that uses those instances (for example, by passing them to the constructor of a class immediately), it is still much better than instantiating concrete implementations directly where they are used. Often, this approach is sufficient.

We should take our time to consider what to extract, where to instantiate certain implementations, and how to pass them. This time spent is not wasted; it will pay off in the long run.

However, because people often overlook this, and since it's generally desirable to do so, if we feel overwhelmed and can't think through every detail, we can start by extracting everything that makes sense and accepting it through the constructor. If we're unsure where it should be instantiated, we can at least handle that within a factory function at the point where the constructor is called. There's no need to complicate things or create anything special—just inject dependencies using some of the approaches discussed here. When we have more time, we can think more carefully about where to instantiate those dependencies and how to pass them to the places where they are used.

> Note that there is no rule that is applicable everywhere, and of course there are exceptions to every rule and best practice.

