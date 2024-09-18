---
layout: post
author: kormang
---

As Dijkstra wrote in his paper "Notes On Structured Programming": "Program testing can be used to show the presence of bugs, but never to show their absence!"

That is why it is important to have clean, readable, and understandable code.

Still, testing is an important part of making software reliable. There are many classifications of testing; we will focus on unit and integration automated testing.

There are other books and articles that focus on testing in detail; here, we will just go through a summary that should help us make good decisions. Basically these are my short thoughts on testing, and overview on testing, not detailed guide or even introduction to testing. In other words, familiarity with testing is prerequisite for reading the text below.

## Unit tests

Unit testing focuses on making sure separate units of code (smaller parts, like classes, functions, UI components) work properly. Usually, such tests are simple but numerous and have good coverage. The goal is to test a single unit of code in isolation, but test it thoroughly, covering all possible scenarios.

Since writing unit tests is not as difficult as writing integration tests, and we test only one unit in isolation, we can usually test all possible scenarios.

Writing unit tests is relatively easy, especially if the unit does not have internal state and does not depend on any global variables. As we already said, having units of code that do not have state (including global one) or have obscure hidden state, and require all dependencies to be injected through the constructor, during compile time (e.g., templates), or otherwise, are all desirable properties of good code. At the same time, it helps by making tests simpler to write (because, in fact, you can say that the unit of code is reused in tests as well as in production code).

Writing unit tests makes us think about these properties, and if we strive to make tests easier to write, at the same time, we will get better code.

## Integration tests

Integration tests help us make sure different units of code are connected and interact properly. If testing involves the whole system, then it is usually called end-to-end testing.

Since we're dealing with many different units and the interaction between them, the number of scenarios can grow exponentially, so often we cannot test all of them. That is why unit tests are useful. However, unit tests cannot test if units are properly interacting.

Another reason why integration tests, and especially end-to-end tests, are hard is because they require many dependencies, possibly from outside of the program, and additional infrastructure. For example, backend code might require a database and possibly other services.

The database is usually not mocked, but a schema for testing in a real database is used to make sure all database-related code is correct.

Other services, especially if they are outside of the project or third-party services, are usually mocked or imitated somehow. That is because providers of the service may not provide an environment for testing, or they may not appreciate being overwhelmed by testing requests, or simply, we want tests to run faster and assume that those services return certain results.

In special cases, we might test API calls to third-party services or libraries to make sure that after an update, they work as we expected.

## TDD or not TDD

TDD - Test Driven Development proposes following way of writing tests.

* Write test for yet to be written code, against empty API, or no code at all at this point
* Run tests, and make sure the newly added test fails
* Write minimal implementation that makes the test pass
* Run tests and make sure all tests pass
* Refactor code, to make it better, and use tests to make sure nothing is broken after refactor
* Repeat

This is helpful in a few ways. First, it makes us think in advance about the interface that our code will have, which usually leads to better code and better architecture. It also helps us avoid skipping tests and moving on to the next feature or writing too few tests, which could result in poorly tested code.

Usually, we can literally develop code only from the testing perspective, running and even debugging code solely by executing and debugging tests, without the need to start the real program, and often without the need to leave our editor. This can be much faster, as the feedback loop is really short. We can much faster get information if our code works or not. One more aspect that makes such development faster is the fact that we automatically set all the necessary preconditions instead of manually running code and leading the program to the desired state before executing the critical part. For example, if we had to start a program, then click on a bunch of buttons, enter a few inputs here and there to set things up, and then click a button to execute the code that we're interested in, that is quite slow and frustrating. By developing code through tests, we skip that boring and repeating part entirely by automating it.

However, there are often cases when TDD is infeasible.

If we have to implement some functionality that we've never had to deal with before, in such a case, we usually don't even know what functions and classes we're going to have, let alone what tests should be written for them. So, we write some code to learn how to solve the problem and iteratively improve it as we expand our understanding until we are satisfied with the code. At that point, the code already works, and then we can write tests for it. If we were thinking about testing it afterward, we had probably arrived with a good, testable architecture, but it is not quite TDD, where we write tests before functionality. One could argue that we should abandon that code from the learning phase and start from scratch using pure TDD, but that is much slower, and the code we already have is usually good enough as it is.

If we're doing something that is directly interacting with the user, as is often the case with developing UI, then we need to try it out ourselves. We need to "feel" how it works; we cannot write tests for it in advance.

## Test or no test

To be able to write integration, and especially end-to-end tests, we require some testing infrastructure, usually provided by testing frameworks. Those frameworks provide us with functions to make common tasks easier, such as moving a mouse to a button and clicking it, or preparing state in the database for the tests, and so on. These frameworks are often made for a certain purpose, like backend testing, frontend testing, or they are specialized for certain already long-living, established projects. However, if the project is not big, already established, but rather new, and it does not fall into some standard categories, like frontend or backend, then it is likely that there is no framework and no infrastructure ready to help in integration and end-to-end testing. Also, chances are that you don't have time to build it yourself. In that case, probably good advice would be to start building such infrastructure, for a start, at least to test some functionality and continue improving it gradually.

Let's say we work in fast-changing environments like startups, which are yet to define their requirements and are constantly changing the description of the product itself. In such cases, writing tests might seem like a waste of time. Often it is because - why write a test for something that will be abandoned tomorrow and changed for something else when we can spend that time building that other thing. On the other hand, constant bugs that pop up, especially if they are in already established, or code that is often reused, can slow us down even more. So we need to find a balance.

If we're constantly changing requirements, it is important to write small pieces of reusable code with a single responsibility, which can be easily combined in different scenarios and combinations. Just changing certain parts or rearranging them can help us meet those new requirements. So, good candidates for unit testing are those units of code that are often reused. Even if a unit of code is not reused that often but has such potential—maybe it is not related explicitly to current requirements but is generally applicable even outside of the project—it is then a good candidate to be unit tested. We want to make certain components reusable and reliable. We want to write them, test them, make sure they are reliable. Then we want to use them wherever we need to and not worry whether they will break or function properly; we just want to use them and not worry too much. That is the way we do it when we use third-party libraries; we generally have confidence in functions from those libraries, and that enables us to move fast. We want to make our own libraries of reliable and reusable components over time to make us move faster.
