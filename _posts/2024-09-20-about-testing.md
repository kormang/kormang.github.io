---
layout: post
author: kormang
---

As Dijkstra wrote in his paper "Notes on Structured Programming": "Program testing can be used to show the presence of bugs, but never to show their absence!"

That is why it is important to have clean, readable, and understandable code.

Still, testing is an important part of making software reliable. There are many classifications of testing; we will focus on unit and integration automated testing.

There are other books and articles that focus on testing in detail; here, we will just go through a summary that should help us make good decisions. Basically, these are my short thoughts on testing and an overview of testing, not a detailed guide or even an introduction to testing. In other words, familiarity with testing is a prerequisite for reading the text below.

## Unit tests

Unit testing focuses on ensuring that separate units of code (smaller parts like classes, functions, UI components) work properly. These tests are usually simple but numerous and provide good coverage. The goal is to test a single unit of code in isolation, but test it thoroughly, covering all possible scenarios.

Since writing unit tests is not as difficult as writing integration tests, and we test only one unit in isolation, we can usually cover all possible scenarios.

Writing unit tests is relatively easy, especially if the unit does not have internal state and does not depend on any global variables. As we mentioned earlier, having units of code that do not have state (including global state) or have obscure hidden state, and requiring all dependencies to be injected through the constructor, during compile time (e.g., templates), or otherwise, are all desirable properties of good code. This also simplifies writing tests, as the unit of code can be reused in tests as well as in production.

Writing unit tests makes us think about these properties, and if we strive to make tests easier to write, we will simultaneously improve the quality of our code.

## Integration tests

Integration tests help ensure that different units of code are connected and interact properly. If testing involves the whole system, it is usually called end-to-end testing.

Since we're dealing with many different units and their interactions, the number of scenarios can grow exponentially, making it difficult to test all of them. This is why unit tests are valuable, but unit tests alone cannot verify whether units are interacting correctly.

Another reason integration tests, especially end-to-end tests, are challenging is that they often require many dependencies, possibly from outside the program, and additional infrastructure. For example, backend code might require a database and possibly other services.

The database is usually not mocked; instead, a schema for testing in a real database is used to ensure all database-related code is correct.

Other services, particularly if they are external or third-party, are typically mocked or simulated. This is because service providers may not offer a testing environment, may not appreciate being overwhelmed with test requests, or we may want tests to run faster by assuming those services return specific results.

In special cases, we might test API calls to third-party services or libraries to ensure they still function as expected after an update.

## TDD or not TDD
TDD (Test-Driven Development) proposes the following approach to writing tests:

1. Write a test for code that hasn't been written yet, possibly against an empty API or no code at all at this point.
2. Run the test and ensure the newly added test fails.
3. Write the minimal implementation needed to make the test pass.
4. Run the tests and make sure all tests pass.
5. Refactor the code to improve it, using tests to ensure nothing is broken after refactoring.
6. Repeat.

This approach is helpful in several ways. First, it forces us to think ahead about the interface our code will have, often leading to better code and architecture. It also prevents us from skipping tests or writing too few, which can result in poorly tested code, which is often unreliable code.

In many cases, we can develop code purely from the testing perspective, running and even debugging solely by executing tests, without needing to run the actual program. This can be much faster since the feedback loop is short. We can quickly determine if our code works. Another factor that speeds up development is that tests automatically set all necessary preconditions, eliminating the need to manually run the program, navigate through UI steps, or set up the environment before testing the critical part. For example, starting a program, clicking buttons, and entering inputs just to reach the part of the code we care about can be slow and frustrating. By developing code through tests, we automate this repetitive process entirely.

However, there are often situations where TDD is impractical.

For example, when implementing a feature we've never worked on before, we may not even know what functions or classes we'll need, let alone what tests to write. In such cases, we typically write some exploratory code to learn how to solve the problem, improving it iteratively as we gain understanding. Once the code works, we can write tests for it. If we were mindful of testing while writing the code, we'd likely arrive at a good, testable architecture, but this isn't quite TDD, where tests are written before functionality. Some may argue that we should discard the exploratory code and start over using pure TDD, but that approach is often slower, and the existing code is usually good enough.

When working on something that directly interacts with users, such as developing a UI, we need to try it out ourselves to "feel" how it works. Writing tests for it in advance isn't always feasible.

If we're writing a function or a class that is supposed to have single responsibility and be highly reusable, that often TDD makes sense. For example - we're find ourselves in situation where we'd like to transform all keys of a json from CamelCase to snake_case, and there is not ready made package that does it. In such case, we may write code like this:

```javascript
function myFunction() {
 // ...some logic

  camelCasedObj = await getObj();

  snake_cased_obj = {
    attr_a: camelCasedObj.attrA,
    attr_b: camelCasedObj.attrB
  }

 // more logic...
}
```

If an object is large, expected to change frequently, has unknown attributes, or we need to perform a similar transformation in multiple places, it’s generally better to write a function that converts an object’s camel-cased attribute names to snake-cased ones. This way, we can handle changes and transformations consistently across the codebase.


```javascript
function myFunction() {
 // ...some logic

  camelCasedObj = await getObj();
  snake_cased_obj = objToSnakeCase(camelCasedObj);

 // more logic...
}
```

In such cases, we should write a reusable function, and it’s better to do so using a TDD approach. We know the expected inputs and outputs, and we should consider all edge cases. Then, we write the tests and, after that, the implementation. This way, we can have much greater confidence in our function and use it freely, without worries, in many places. It will have a much greater chance of standing the test of time. This is what we usually expect from functions that belong to well-known libraries. In this example, it is not difficult to achieve such a level of reliability with our own code. It will also save us time when we encounter a similar situation elsewhere, even if it involves a slightly different edge case. Our function will just work, instead of discovering a problem in production.

## To test or not to test

To write integration and, especially, end-to-end tests, we need a testing infrastructure, usually provided by testing frameworks. These frameworks offer functions to simplify common tasks, like simulating mouse clicks or setting up the database state for tests. They are often designed for specific purposes, such as backend testing, frontend testing, or tailored to well-established projects. However, if the project is new and doesn’t fit into standard categories like frontend or backend, it’s likely that no ready-made framework or infrastructure exists to assist with integration and end-to-end testing. Additionally, there may not be enough time to build this infrastructure from scratch. In such cases, a practical approach is to start building the testing infrastructure gradually, focusing first on testing some functionality and improving it over time.

In fast-changing environments like startups, where requirements are constantly evolving and the product itself is frequently redefined, writing tests can seem like a waste of time. It often feels inefficient to write a test for something that may be abandoned or changed the next day, especially when that time could be spent building the next feature. However, frequent bugs, particularly in established or frequently reused code, can slow down progress even more. Therefore, finding a balance is key.

When requirements are constantly changing, it’s crucial to write small, reusable pieces of code with a single responsibility. These can be easily combined in different scenarios or rearranged to meet new requirements. Good candidates for unit testing are pieces of code that are reused often. Even if a unit isn’t frequently reused but has the potential for general applicability beyond current requirements, it’s still a good candidate for unit testing. The goal is to build reliable and reusable components that can be confidently used across the project.

We want to develop and test these components so they are dependable and reliable, allowing us to use them without worrying about whether they will break. This is how we treat third-party libraries—we generally trust their reliability, which enables us to work quickly. Over time, building our own library of reliable, reusable components will help us move faster as well.