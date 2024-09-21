---
layout: post
author: kormang
---

As explained in the previous post, generalized code can take various forms. It can be abstract classes that require inheritance, functions that accept other functions as parameters, or generic functions that utilize generic parameters (template arguments).

However, there are no fundamental differences between these forms as significant as those between the approaches we are about to explore.

This discussion focuses on the classification of approaches to enabling one unit of code to perform different tasks depending on the context. There are two basic approaches to making code generalized. Let’s look at a trivial example to illustrate these two approaches.

We have function `performOperation(a: number, b: number, c: number, d: number, operation: string): number`, which performs binary operation on 4 numbers, and the type of operation depends on the string parameter (if "+" then it will perform addition `(a + b) + (c + d)`, if "-" subtraction `(a - b) - (c - d)`, and so on).

```typescript

// In languages like C++ and Java we'd usually use enums to ensure that only
// valid operations can get passed to the function.
type OperationLiteral = "+" | "-" | "*" | "/";

function performOperation(a: number, b: number, c: number, d: number, operation: OperationLiteral): number {
  switch (operation) {
    case "+":
      return (a + b) + (c + d);
    case "-":
      return (a - b) - (c - d);
    case "*":
      return (a * b) * (c * d);
    case "/":
      return (a / b) / (c / d);
  }
}

```

If we want to add a new type of operation we need to modify the code. So if we want to support modulo operation, we need to add `case "%": return (a % b) % (c % d);` to the switch statement.

Then we can use it like this:

```typescript
console.log(performOperation(1, 2, 3, 4, "+"));
console.log(performOperation(1, 2, 3, 4, "%"));

```

Another approach is to parameterize a generalized function with behavior, like this:

```typescript
function performOperation(a: number, b: number, c: number, d: number, operation: (number, number) => number): number {
  return operation(operation(a, b), operation(c, d));
}


console.log(performOperation(1, 2, 3, 4, (a, b) => a + b));
console.log(performOperation(1, 2, 3, 4, (a, b) => a % b));
```

When parameterizing code with behavior it opens up possibilities to do more and combine it in many ways, not just those assumed by the author of the code.

```typescript
console.log(performOperation(1, 2, 3, 4, (a, b) => a + b % a))
```

Sometimes, passing behavior might seem clumsy and it might not be as easy to use for standard cases. For that reason, we can use a hybrid approach.

```typescript
type OperationLiteral = "+" | "-" | "*" | "/";

const operationsMap: Record<OperationLiteral, (number, number) => number> = {
  "+": (a, b) => a + b,
  "-": (a, b) => a - b,
  "*": (a, b) => a * b,
  "/": (a, b) => a / b,
};

// In languages such as C++, OperationLiteral would be enum,
// we would have two overloaded functions, one with operation being enum,
// and another operation being callback.
function performOperation(a: number, b: number, c: number, d: number, operation: OperationLiteral | (number, number) => number): number {
  const operationF = typeof operation === "function" ? operation : operationsMap[operation];
  return operationF(operationF(a, b), operationF(c, d));
}


console.log(performOperation(1, 2, 3, 4, "+"));
console.log(performOperation(1, 2, 3, 4, (a, b) => a % b));

```

The hybrid approach may be the best option in some cases, especially if we frequently use standard operators and find it significantly improves the developer experience to use parameters like in the first approach. Otherwise, the second approach is objectively the best, as it is simple yet as flexible, reusable, and generalized as possible.

The first approach (parameterizing with *behavior-modifying data* rather than behavior itself) is usually the least favorable, but in rare cases, it can be justified.

Some may find the hybrid approach to be the best if they prefer the simplicity of passing common operations as strings or enums. Others might prefer the second approach, especially if we have a library of named operation objects (lambdas), as this eliminates the need to switch on operation types, and an IDE would likely provide suggestions for available predefined operation objects.

An even more common example of the first approach is adding flags to modify behavior, such as passing a boolean parameter.

```typescript
async function updateProducts(seller: Seller) {
  const response = await fetch(`${API_BASE}/sellers/${seller.id}/products/`);
  const products = await response.json();

  const state = getState();
  state.setProducts(products);
}
```

Imagine that we want to make the `updateProducts` function more generalized. Sometimes, we might want to assign a seller object to a product object, but not always (assume that we receive product object from backend without seller object as their attribute). Therefore, we want to reuse the function while adding this additional functionality.

```typescript
async function updateProducts(seller: Seller, assignSeller: boolean) {
  const response = await fetch(`${API_BASE}/sellers/${seller.id}/products/`);
  const products = await response.json();

  if (assignSeller) {
    products.forEach(p => {
      p.seller = seller;
    });
  }

  const state = getState();
  state.setProducts(products);
}
```

So, that is the first approach to generalization. Now, what would happen if, in some cases, we want to do something else instead of assigning a seller? Would we opt for another flag? Or maybe an enum? Of course not.

The second approach is similar to what we have seen in the previous example.

```typescript
async function updateProducts(seller: Seller, enchanceProducts: (Array<Product>) => Array<Product>) {
  const response = await fetch(`${API_BASE}/sellers/${seller.id}/products/`);
  const products = await response.json();

  const enchancedProducts = enchanceProducts(products);

  const state = getState();
  state.setProducts(enchancedProducts);
}
```

Then we can use it like this.


```typescript
// Version without modification of products.
await updateProducts(seller, products => products);

// Version that assigns seller.
await updateProducts(seller, products => {
  products.forEach(p => p.seller = seller);
  return products;
});

// Version that does something else.
await updateProducts(seller, doSomethingArbitraryWithProducts);
```

In this approach, we are parameterizing the function with another behavior. We're extracting a part of the responsibility outside the function, and then we pass it in.

There is also a third approach that doesn't parameterize the function with behavior. Instead, we can cut the function into smaller pieces, each with a single responsibility. Then, combine them or assemble them in different ways.

```typescript
async function fetchProducts(seller: Seller) {
  const response = await fetch(`${API_BASE}/sellers/${seller.id}/products/`);
  const products = await response.json();
  return products;
}

function updateStateWithProducts(products: Array<Product>) {
  const state = getState();
  state.setProducts(products);
}
```

Now we can combine them in different ways.

```typescript
async function updateProducts(seller: Seller) {
  updateStateWithProducts(await fetchProducts(seller));
}


async function updateProductsWithAssignedSellers(seller: Seller) {
  function assignSellers(products) {
    products.forEach(p => p.seller = seller);
    return products;
  }
  updateStateWithProducts(assignSellers(await fetchProducts(seller)));
}


async function updateProductsWithSomething(seller: Seller) {
  updateStateWithProducts(
    doSomethingArbitraryWithProducts(await fetchProducts(seller))
  );
}
```

*Notice the subtle difference in the way products are fetched in these two examples—it can make a big difference when it comes to maintainability. Always aim to split responsibilities whenever possible. Build large, multifunctional "super-functions" only if they are composed of smaller, well-defined pieces that can be reused independently outside of the super-function.*

Both the second and third approaches are valid and should be chosen based on the situation. The third approach (avoiding generalization altogether) may not be very useful in our first example (with `performOperation`), but in the second example (with products), it might actually be better. The choice depends on the circumstances, so we should carefully consider the context before deciding.

It's important to note that the third approach isn't about generalization—it's about avoiding it. This is often preferred because, as mentioned, generalized code can become more complex. However, there are times when generalization is beneficial. We can categorize two approaches to generalization:
1. Using flags and other parameters to modify behavior.
2. Parameterizing generalized code with specific behaviors.

Let’s explore one more example, which is based on real-world code but modified for the purposes of this discussion.

```typescript
async function findDomainUserByPhone(phoneNumber: string): Promise<DomainUser | null> {
  const userRepository = getRepository(User);
  const user = await userRepository
    .createQueryBuilder("user")
    .where("user.phone = :phone", { phone: phoneNumber })
    .orderBy("user.createdAt", "DESC")
    .getOne();

  if (!user) {
    return null;
  }

  return validateDomainUser(user);
}

async function findDomainUserByEmail(email: string): Promise<DomainUser | null> {
  const userRepository = getRepository(User);
  const user = await userRepository
    .createQueryBuilder("user")
    .where("user.email = :email", { email: email })
    .orderBy("user.createdAt", "DESC")
    .getOne();

  if (!user) {
    return null;
  }

  return validateDomainUser(user);
}
```

To avoid duplication in this code we can create generalized function using first approach.

```typescript
async function findDomainUserBy(fieldName: "phone" | "email", value: string): Promise<DomainUser | null> {
  const userRepository = getRepository(User);
  let query = userRepository.createQueryBuilder("user")
  if (fieldName === "phone") {
    query = query.where("user.phone = :phone", { phone: value })
  } else {
    query = query.where("user.email = :email", { email: value })
  }
  query = query.orderBy("user.createdAt", "DESC")

  const user = await query.getOne();

  if (!user) {
    return null;
  }

  return validateDomainUser(user);
}

async function findDomainUserByPhone(phoneNumber: string): Promise<DomainUser | null> {
  return await findDomainUserBy("phone", phoneNumber);
}

async function findDomainUserByEmail(email: string): Promise<DomainUser | null> {
  return await findDomainUserBy("email", email);
}
```

Of course, we can write it a bit more generalized, but it is still just sub-approach of the first approach.

```typescript
async function findDomainUserBy(fieldName: string, value: string): Promise<DomainUser | null> {
  const userRepository = getRepository(User);
  const user = await userRepository
    .createQueryBuilder("user")
    .where(`user.${fieldName} = :value`, { value: value })
    .orderBy("user.createdAt", "DESC")
    .getOne();

  if (!user) {
    return null;
  }

  return validateDomainUser(user);
}

async function findDomainUserByPhone(phoneNumber: string): Promise<DomainUser | null> {
  return await findDomainUserBy("phone", phoneNumber);
}

async function findDomainUserByEmail(email: string): Promise<DomainUser | null> {
  return await findDomainUserBy("email", email);
}
```

The second approach would involve passing another behavior as a dependency via function arguments.

```typescript
async function findDomainUserWhere(enrichQuery: (Query<User>) => Query<User>): Promise<DomainUser | null> {
  const userRepository = getRepository(User);
  let query = userRepository.createQueryBuilder("user")
  query = modifyQuery(query);
  query = query.orderBy("user.createdAt", "DESC")

  const user = await query.getOne();

  if (!user) {
    return null;
  }

  return validateDomainUser(user);
}

async function findDomainUserByPhone(phoneNumber: string): Promise<DomainUser | null> {
  return await findDomainUserWhere(query => query.where("user.phone = :phone", { phone: phoneNumber }));
}

async function findDomainUserByEmail(email: string): Promise<DomainUser | null> {
  return await findDomainUserWhere(query => query.where("user.email = :email", { email: email }));
}
```

That is one way to implement the second approach. Both approaches have their pros and cons, and it is up to programmer to decide which approach is better in particular case. Usually the second approach is better and more flexible and powerful, however in this particular situation it might not be the case.

In case we want to avoid generalization and also reduce code duplication we should split function into smaller pieces with less responsibility.


```typescript
function createUserQuery() => Query<User> {
  const userRepository = getRepository(User);
  return userRepository.createQueryBuilder("user");
}

async function fetchDomainUserForQuery(query: Query<User>) => Promise<DomainUser | null> {
  const user = await query
    .orderBy("user.createdAt", "DESC")
    .getOne();

  if (!user) {
    return null;
  }

  return validateDomainUser(user);
}

async function findDomainUserByPhone(phoneNumber: string): Promise<DomainUser | null> {
  const query = createUserQuery()
    .where("user.phone = :phone", { phone: phoneNumber });

  return await findDomainUserFromQuery(query);
}

async function findDomainUserByEmail(email: string): Promise<DomainUser | null> {
  const query = createUserQuery()
    .where("user.email = :email", { email: email });

  return await fetchDomainUserForQuery(query);
}

// Now we can easily make "find" function for whatever query we want without much duplication.
async function findDomainUserByIrregularQuery(olderThenYears: number, registeredAfter: Date): Promise<DomainUser | null> {
  const query = createUserQuery()
    .where("user.createdAt = :registeredAfter AND NOW() - user.dateOfBirth > :age", { registeredAfter: registeredAfter. age: olderThenYears});

  return await fetchDomainUserForQuery(query);
}
```

Again, this is not the only way to do it, but it illustrates how we can avoid generalization (and thus, arguably, avoiding additional complexity) by splitting code into smaller pieces, while at the same time decently reducing code duplication.

Hopefully, examples presented here were good enough to understand the essence of the two approaches to generalization, and the approach to avoid the generalization. Deciding when to use each approach takes experience, but being aware of these classifications should be a helpful starting point.
