---
layout: post
author: kormang
---

Suppose you have a class like this:

```typescript
class Data {
  async set(value: T): void {...}
  async get(): T {...}
  async exists(): boolean {...}
}
```

It stores some data, somewhere, asynchronously.

Suppose you find yourself writing these two pieces of code:

```javascript
if (!(await data.exists())) {
    const blabla = await downloadData(param1, param2);
    result = {}
    for (const b of blabla) {
      processBlabla(result, b)
    }
    await data.set(result);
}
return data.get();
```

```javascript
if (!(await data.exists())) {
  const x = await fetchit(p1);
  const result = x.map(e => x.y)
  await data.set(result);
}
return data.get();

```

You might start thinking - Well it seems that some pattern is popping up, might be worth extracting it into a function, to avoid code duplication.

If you extract method of Data class like this:

```typescript
getOrSet(ifNotExistsGet: () => T) {
  if (!this.exists()) {
    const result = await ifNotExistsGet();
    await this.set(result);
  }
  return this.get();
}
```

you can use it like this:

```typescript
return data.getOrSet(() => {
  const blabla = await downloadData(param1, param2);
  result = {}
  for (const b of blabla) {
    processBlabla(result, b)
  }
  return result;
});
```

```typescript
return data.getOrSet(() => {
 const x = await fetchit(p1);
 return x.map(e => x.y)
})
```

Now you could say that this method is somewhat abstract, or generic, because it needs that `ifNotExistsGet` parameter to concretize it.

_Note_ that term "abstract"" is often used for classes or functions that can be concretized via inheritance, and term "generic" is often used for units of code that are concretized via generic or template parameters. There are multiple ways to make unit of code generalized, and simply passing parameters, like in this case, is one of them.
So, let's use term **generalized** for all kinds of reusable code that needs to be concretized one way or another, to do specific thing.

#### Drawbacks of making code generalized

There are couple of drawbacks of making code **generalized**:

**It could be hard to understand due to its abstract/generic/generalized nature, and universality.**
Endless layers of abstraction can conceal the true purpose of the code.

Sometimes, a little bit of duplication is good, if it makes code more clear, and it is not repeated too many times.

**Resorting too often to making generalized utility function, results in messy "utils" directory** with function like `formatDate` and `formattedDate` that essentially do the same thing, but author of the latter did not see that there is already function like that.
Even worse, a third developer could use neither and write the formatting code inline, duplicating it again. If language and IDE support it, making extension methods also helps. There is a reason why people usually use `Object.assign` in JavaScript, rather then writing their own, so there might be a way to organize utils in such way that people tend to use it more often than not.

**Having function that is shared by two or more modules can lead to growth of its complexity.**
Let's say you put a function into utils and make it shared between two modules. Initially, it does only one thing, but soon, one module needs something a bit different. So, you add a parameter to change behavior in some cases. Then the authors of the other module do the same, and soon you end up with a function with an unclear purpose, a mishmash of completely different use cases and a desire to reduce duplication and increase reuse. UI components are especially prone to this scenario. To prevent it, duplicate them when the need to change behavior arises. If possible, you can refactor them to internally call a shared piece of logic, but even that is not always necessary.

**Generalized functions may even be hard to read if they do not conform to some standard or even better if they are not well known**, like `map`, `reduce`, etc. There is definitely value in being well known when it comes to readability and understanding code. Naming is also very important, their names should be short, yet very clean about what these functions do. It still doesn't mean that you should never wevenrite your own function with special purpose and name, but they better be used often so that learning their meaning and semantics pays off to other people working on the project.

**It is hard to write it properly. This is especially true for languages with generics like TypeScript.**
Look at this function for example:

```typescript
  export function groupBy<V, K extends KeyTypes>(array: V[], keyGetter: (v: V) => K): Partial<Record<K, V[]>> {...}
```

Author tried hard to make type system happy, but it would be much easier to write, and even read, to write that function for specific use case (we wouldn't event need `keyGetter`), with specific types (we wouldn't need all these generics). But on the other hand, it is worse to have to implement it every time we need to group things like that, so it is a trade off. This is the case when less skilled developers could benefit from functions written by more skilled colleagues.

#### Advantages of making code generalized

Advantages of making code generalized are usually clear to everybody, but not enough obviously, because people would usually not write generalized code (mostly because of last point - it is harder). So we will point out to some of them:

**Less code, less bugs, faster development when generalized code is already written.**

**Easier to change implementation, or fix bugs, or improve performance by making change only in one place.**
Consider again `getOrSet` method above. Let's say, the data is some remote cache, and by the time you call `get` after `exists` returned `true`, data could expire. To fix this race condition, you would only need to make change in one place.

**Prevention of proliferation of bad code.**
Copying code around could lead to huge number of copies of bad pieces of code, or code that contains bugs. Function `getOrSet` from above, is one example, without it, but containing race condition would be copied all over the place, and when somebody finds it, change needs to be made on many places, if that someone even considers fixing it everywhere and not simply patch current bug and move on.

Having said all that, all pros and cons, people are more inclined towards duplicating bad code than writing unnecessary complicated generalized code. Which makes code duplication greater problem.
