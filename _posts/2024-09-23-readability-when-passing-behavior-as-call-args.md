---
layout: post
author: kormang
---

Let's have a short discussion about readability of code that uses generalized functions.

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

The third version a bit shorter then first one, and it is more readable. Although the `processAB3` function itself is as short as the second one, there are more additional functions which together add up to roughly same amount of code as the first version. It is separated into clear, distinct steps, each step is named, names are given by helper functions. It would be even more readable if JavaScript had `zip` function to use instead of `map`. Another nice thing about it is that we have created ourselves three reusable functions (`differenceOfSets`, `intersectionOfSets`, `ABEquals`), and at least two of them are useful outside of this task. It is however, less efficient than the first one, which computes `collection1` and `collection2` in one go, while third version takes 3 steps for the same task.

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
