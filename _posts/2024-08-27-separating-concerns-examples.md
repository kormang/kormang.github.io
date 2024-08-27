---
layout: post
author: kormang
---

It is recommended to read the previous post first - [Single responsibility and code reuse](./2024-08-24-single-responsiblity-and-code-reuse.md). Garbage collector does not free us from having to separate responsibilities that are specific to our software.

The best way to explain what different responsibilities are, and how to separate them is by showing as many examples as possible.

Examples:
[Sales report](#sales-report)
[Car market interface](#car-market-interface)
[Java IO streams](#java-io-streams)
[CDN service](#cdn-service)
[Gesture Web Site](#gesture-web-site)
[Filter users](#filter-users)
[Grouping items the reusable way](#grouping-items-the-reusable-way)

At the end we will try to draw some [conclusions](#conclusion), and we [consider reusability of modules](#considering-modules).


### Sales report

Let's pretend we're building the backend for an online store. It turned out that we have a few types of users. We have those who shop, and those who sell. They have some shared data and behaviors, and some users can actually be both sellers and buyers simultaneously. Let's assume we have a single table for all users, and we've implemented inheritance in the code. In the database, we use a flag called `is_seller`` to show that a user is also a seller. However, this isn't very important. What's important is this: we need a feature that gathers all the sales from the last month, adds them up, calculates any fees, and then sends a report to the email of the registered seller-user. Each item has its own fee applied when it's sold or purchased, and the percentage is saved in the database at that time.

Now, let's imagine that we have a class called `Seller`. We want to include a method called ` sendProfitReport` in that class. We use this method from another class, let's call it `SellerService`, like this:

```javascript
const seller = getSellerRepository().findById(seller_id);
seller.sendProfitReport();
```

Let's say we've implemented our method like this:

```javascript
class Seller {
  //...
  sendProfitReport = () => {
    const allSales = getSalesRepository().findWhere({seller_id: this.id})
    const currentMonthSales = allSales.filter(s => s.date.getMonth() = new Date().getMonth())

    const table = '<table>
      + currentMonthSales.map(s => `<tr><td>${s.itemName}</td><td>${s.quantity}</td><td>${s.price}</td><td>${s.price * s.quantity * (1 - curr.fee / 100.0)}</td><td>${s.fee}%</td></tr>`).join('')
      + '</table>'

    const total = currentMonthSales.reduce((acc, curr) => acc + curr.price * curr.quantity * (1 - curr.fee / 100.0), 0.0)
    const summary = `<div>Total: ${total}</div>`

    const mailContent = '<html><body>' + table + summary + '</body></html>'


    getEmailService().sendEmail(this.email, mailContent)
  }
}
```

You have probably never written code as bad as this. Probably, you identify that formatting email and constructing that HTML inside this function is probably a responsibility that somebody else should take. Usually, there are libraries for sending email, and those libraries have templating engines that fill email structure with provided data. So emailing libraries save us from making that mistake by taking that responsibility onto themselves and out of our code. So we would usually return some object containing all the necessary information. In JavaScript, we can return an object literal or go with the good old-fashioned constructor.

```javascript
class Seller {
  //...
  sendProfitReport = () => {
    const allSales = getSalesRepository().findWhere({seller_id: this.id})
    const currentMonthSales = allSales.filter(s => s.date.getMonth() = new Date().getMonth())

    const salesForReport = currentMonthSales.map(s => {
      quantity: s.quantity,
      price: s.price,
      fee: s.fee,
      total: s.quantity * s.price * (1 - curr.fee / 100.0)
    })

    const total = currentMonthSales.reduce((acc, curr) => acc + curr.price * curr.quantity * (1 - curr.fee / 100.0), 0.0)
    const mailData = new ProfitReportMailData(salesForReport, total)


    const templateName = 'profit-report'
    getEmailService().sendEmail(templateName, this.email, mailData)
  }
}
```

Maybe this is something you would usually write?

Can you identify multiple responsibilities this function has?

Let's finish with all this email stuff first. This class should not depend on an `EmailService`. It should not be **coupled** with that class, should not know about that class, and should not have the responsibility of sending email.

```javascript
class Seller {
  //...
  computeProfitReportData() {
    const allSales = getSalesRepository().findWhere({seller_id: this.id})
    const currentMonthSales = allSales.filter(s => s.date.getMonth() = new Date().getMonth())

    const salesForReport = currentMonthSales.map(s => {
      quantity: s.quantity,
      price: s.price,
      fee: s.fee,
      total: s.quantity * s.price * (1 - curr.fee / 100.0)
    })

    const total = currentMonthSales.reduce((acc, curr) => acc + curr.price * curr.quantity * (1 - curr.fee / 100.0), 0.0)
    const mailData = new ProfitReportMailData(salesForReport, total)

    return mailData
  }
}

class SellerService {
  // ...
// This can be in SellerService,
// but it is not important right now:
  const mailData = seller.computeProfitReportData()
  const templateName = 'profit-report'
  getEmailService().sendEmail(templateName, seller.email, mailData)
}
```

OK, but what benefits do we have from that?

We can now use that data in some other place, possibly send it not only via email but also through some other service, or display it in a UI interface directly to the user. Now that `computeProfitReportData` no longer has the responsibility of sending emails, it is **reusable** in other places. That demonstrates the **relationship between single responsibility and code reuse**, and the benefits it brings.

However, we still have more than one responsibility in our method.

Firstly, let's say that the fee is no longer computed at the moment of purchase, but at the end of the month, during the payout to the seller, or at the time of generating the report. Additionally, the fee is now represented using fractions, e.g. 0.5, instead of 50%.

```javascript
class Seller {
  //...
  calcFee(s) {
    //...
  }

  computeProfitReportData() {
    const allSales = getSalesRepository().findWhere({seller_id: this.id})
    const currentMonthSales = allSales.filter(s => s.date.getMonth() = new Date().getMonth())

    const salesForReport = currentMonthSales.map(s => {
      quantity: s.quantity,
      price: s.price,
      fee: this.calcFee(s),
      total: s.quantity * s.price * this.calcFee(s)
    })

    const total = currentMonthSales.reduce((acc, curr) => acc + curr.price * curr.quantity * this.calcFee(s), 0.0)
    const mailData = new ProfitReportMailData(salesForReport, total)

    return mailData
  }
}
```

```javascript
class SellerService {
  //...
  const mailData = seller.computeProfitReportData()
  const templateName = 'profit-report'
  getEmailService().sendEmail(templateName, seller.email, mailData)
  //...
}
```

You might have already noticed that calculating the fee that the platform takes from the seller is not a responsibility that should belong to the `Seller` class. This is particularly true if we want to have different kinds of fee calculation schemes for different scenarios. There could also be situations where we want to experiment with a few fee calculation schemes while developing startup software, and we are not yet sure which one is better or which will meet the market's needs. Alternatively, we might want to use multiple schemes simultaneously to run an AB test/experiment. In such cases, we can create an interface like this:

```typescript
// Here we use TypeScript because JavaScript does not have interfaces, and trying to emulate them here is only going to make things less clear.

interface FeeCalc {
  calcFee(s: Sale): number;
}

```

Then we can implement that interface in different ways and pass a specific implementation to the `Seller` class. Depending on the situation, it may or may not be possible to pass that implementation to the constructor of the `Seller` class, but nonetheless, this is what we're trying to achieve. This practice is known as **Dependency Injection** because we _inject_ a dependency into the `Seller` class.

```javascript
class Seller {
  //...

  constructor(..., feeCalc) {
    this.feeCalc = feeCalc
  }

  calcFee(s) {
    return this.feeCalc.calcFee(s)
  }

  computeProfitReportData() {
    const allSales = getSalesRepository().findWhere({seller_id: this.id})
    const currentMonthSales = allSales.filter(s => s.date.getMonth() = new Date().getMonth())

    const salesForReport = currentMonthSales.map(s => {
      quantity: s.quantity,
      price: s.price,
      fee: this.calcFee(s),
      total: s.quantity * s.price * this.calcFee(s)
    })

    const total = currentMonthSales.reduce((acc, curr) => acc + curr.price * curr.quantity * this.calcFee(s), 0.0)
    const mailData = new ProfitReportMailData(salesForReport, total)

    return mailData
  }
}
```

**Dependency Injection** is the approach used to achieve **Inversion of Control**. We construct the `Seller` object, then we call some of its methods, and those methods call back some code that we have provided while constructing the object. This way, the seller object transfers the control of execution to some outside code, to its dependency.

This example is sometimes referred to as the **Strategy pattern**, which is simply the normal way of using interfaces, the way they are designed to be used. This approach enables us to _delegate_ certain parts of behavior to other units of code. It allows us to extract a part of our class and create different implementations of it, which are interchangeable.

**NOTE**: We could have used simple function, instead of interface (object) with one method.

```javascript
class Seller {
  //...
  constructor(..., calcFee) {
    this.calcFee
  }

  calcFee(s) {
    return this.calcFee(s)
  }
}
```

Interfaces with one method are essentially just functions, and vice versa (in languages that treat functions as "first-class citizens", capable of behaving like any other value, and languages that have closures). Both approaches have their pros and cons but are fundamentally the same.

Moving the logic/behavior related to fee calculation out of the `Seller` class now enables the use of `FeeCalc` implementations in various places, thus promoting code reuse. On the other hand, our method `computeProfitReportData` and our class `Seller` are no longer tied to a particular implementation of fee calculation (not **tightly coupled**). This allows for multiple combinations from both sides: different `FeeCalc` implementations in various places, and various `FeeCalc` implementations in this specific case. This also serves as an example of how **separating concerns** makes code **reusable**—illustrating how a unit of code with **single responsibility** (`FeeCalc`) can be reused in different contexts.

Now, let's revisit the task of removing dependencies from our method.

```javascript
class Seller {
  //...

  findSalesForCurrentMonth() {
    const allSales = getSalesRepository().findWhere({seller_id: this.id})
    const currentMonthSales = allSales.filter(s => s.date.getMonth() = new Date().getMonth())
    return currentMonthSales
  }

  computeProfitReportData(currentMonthSales) {
    const salesForReport = currentMonthSales.map(s => {
      quantity: s.quantity,
      price: s.price,
      fee: this.calcFee(s),
      total: s.quantity * s.price * this.calcFee(s)
    })

    const total = currentMonthSales.reduce((acc, curr) => acc + curr.price * curr.quantity * this.calcFee(s), 0.0)
    return new ProfitReportData(salesForReport, total)
  }

  computeProfitReportDataForCurrentMonth() {
    const currentMonthSales = seller.findSalesForCurrentMonth()
    const reportData = seller.computeProfitReportData(currentMonthSales)
    return reportData
  }
}
```

```javascript
class SellerService {
  //...
  sendProfitReport(seller) {
    const mailData = seller.computeProfitReportDataForCurrentMonth()
    const templateName = 'profit-report'
    getEmailService().sendEmail(templateName, seller.email, mailData)
  }
}
```

Now, `computeProfitReportData` is a mostly pure function. It depends (to a certain extent) only on its inputs and does not produce side effects. It also adheres to a single responsibility principle. It can be used to calculate report data for different periods, although we usually require it for the current month, which is why we've created a shortcut method for that purpose.

If our `Seller` class represents an entity from a database and holds data from a specific row, it might be bearing too much responsibility for such a class. Perhaps it would be more appropriate to place all that logic within a `SellerService`, converting `Seller` into a pure data class (only holding data, no behavior). The `findSalesForCurrentMonth` logic could then be moved to a `SalesRepository` since that class should handle the retrieval of complex data from the database. However, let's defer the architecture discussion for now and focus on a small detail: how to correctly determine if a sale is from the current month.

A bug has been identified in this code. It became apparent after a year, when reports began including sales from the same month the previous year we were comparing only the month, not the year. To make things worse, anyone who needed that logic elsewhere copied that single line, leading to this minor bug proliferating in many places.

_This demonstrates that sometimes even a single line is sometimes too much to be duplicated across the codebase, as it can hold relatively complex logic and, of course, bugs._

For this reason, we are going to create a function specifically for this purpose. We can place it in a `utils/date.js` (`utils/date.ts`) file or something similar.

```javascript
function areSameMonth(dateA, dateB) {
  return dateA.getMonth() == dateB.getMonth() && dateA.getYear() == dateB.getYear()
}
```

Maybe there is another bug there, or it can be computed in a better way, but at least, now it is easy to fix it, no need to find all such places.

Now, we have this.

```javascript
  findSalesForCurrentMonth(seller_id) {
    const allSales = getSalesRepository().findWhere({seller_id})
    const isCurrentMonth = areSameMonth.bind(new Date())
    const currentMonthSales = allSales.filter(isCurrentMonth)
    return currentMonthSales
  }
```

Of course, there might be better ways to implement this entire function; perhaps all of this can be calculated in one SQL statement. In such a case, wherever we needed sales for the current month, we can improve it in one go. Compare that to our initial implementation of `sendProfitReport`.

It is important to note that you're not always going to make it perfect and correct. Sometimes you'll, without thinking, add another responsibility to an existing function or class. Other times, you'll put multiple responsibilities into a single unit of code from the start. That is natural; just constantly do refactoring with this in mind. To get better at it, I recommend the book "Refactoring" by Martin Fowler. The second edition is especially good; in the first chapter, you have a much more complete and better example than this one.

### Car market interface

Let's say we're building an automated purchasing bot that, upon identifying favorable deals for cars or parts on car-selling websites, automatically generates purchase requests. Our aim is to scan multiple such sites, identifying the best deals across all of them simultaneously. Additionally, we intend to utilize these platforms to sell cars or parts as well.

We'll have simple interface for buying and selling.

```typescript
interface MarketInterface {
  async buy(itemId: string): BuyResult;
  async putOnSale(item: ItemData): SellResult;
}
```

For each of the car market sites, we'll make different implementation.

We're making certain algorithms, and these algorithms need to be independent of what market (site) we're working with currently.

Let's say we have three implementations for now:

```javascript
class SiteAMarketInterface {
  async buy(itemId) {
    // Do some preparation and make request.
    const data = await fetch(siteABuyUrl...)
    // ... site specific code
    return result
  }
  async putOnSale(item) {
    // Do some preparation and make request.
    const data = await fetch(siteASellUrl...)
    // ... site specific code
    return result
  }
}

class SiteBMarketInterface {
  async buy(itemId) {
    // Do some preparation and make request.
    const data = await fetch(siteBBuyUrl...)
    // ... site specific code
    return result
  }
  async putOnSale(item) {
    // Do some preparation and make request.
    const data = await fetch(siteBSellUrl...)
    // ... site specific code
    return result
  }
}

class SiteCMarketInterface {
  async buy(itemId) {
    // Do some preparation and make request.
    const data = await fetch(siteCBuyUrl...)
    // ... site specific code
    return result
  }
  async putOnSale(item) {
    // Do some preparation and make request.
    const data = await fetch(siteCSellUrl...)
    // ... site specific code
    return result
  }
}
```

Simplified look of one of our algorithms look like this:

```javascript
class AlgorithmB {
  //...

  // Find item to buy on first site, then buy it.
  const itemId = await findItemToBuy()
  const result = await site1MarketInterface.buy(itemId)
  if (result.isSuccessful) {
    // Put same item on sale in another site.
    await site2MarketInterface.putOnSale(prepareItemData(result))
  }

  // ...
}
```

This algorithm of course does not know which exactly site we're dealing with here, it can work with any combination of sites.

We have few such algorithms, they all work at the same time, doing their stuff. Somewhere, in one of them there is code like this:

```javascript
class AlgorithmB {
  //...

  // Go through the list of items and buy each one.
  const results = items.map(item => marketInterface.buy(item.it))
  // Process results.

  // ...
}
```

Now, we've figured out that sometimes requests fail because of bad network, or site returns internal server error for some reason, and we need to try again.

We can add that code to market interface.

```javascript
class SiteAMarketInterface {
  async buy(itemId) {
    // Do some preparation and make request.
    let successful = false;
    while (!successful) {
      try {
        const data = await fetch(siteABuyUrl...)
        successful = true
      } catch(e) {
      }
    }
    // ... site specific code
    return result
  }
  async putOnSale(item) {
    // Do some preparation and make request.
    let successful = false;
    while (!successful) {
      try {
        const data = await fetch(siteASellUrl...)
        successful = true
      } catch(e) {
      }
    }
    // ... site specific code
    return result
  }
}
```

Now we have duplication of this code, and we of course decide to refactor it and make single function out of it.

```javascript
async function fetchWithRetries(...args) {
  let successful = false
  let result = null
  while (!successful) {
    try {
      result = await fetch(...args)
      successful = true
    } catch(e) {
    }
  }
  return result
}

class SiteAMarketInterface {
  async buy(itemId) {
    // Do some preparation and make request.
    const data = await fetchWithRetries(siteABuyUrl...)
    // ... site specific code
    return result
  }
  async putOnSale(item) {
    // Do some preparation and make request.
    const data = await fetchWithRetries(siteASellUrl...)
    // ... site specific code
    return result
  }
}
```

Now we have extracted this logic into function with single responsibility, and we can reuse it in both functions. But we also need to use it in both functions of other implementations of the market interface. If we're going to make new implementations we need to keep in mind that this functions should be used. So this is new requirement for implementors, part of the interface contract that needs to be respected, but it is not explicit, it is hidden, and implementor can easily forget about it. Another problem is that some implementations might not use `fetch` but instead client library of the market site. In that case this function can not help. Yet another problem is that in some cases, in call sites of some algorithms we might not want to retry. So maybe we can rewrite it a bit differently, and use it in places where we call methods of the market interface, on call sites.

```javascript
async function retry(func) {
  let successful = false
  let result = null
  while (!successful) {
    try {
      result = await func()
      successful = true
    } catch(e) {
    }
  }
  return result
}
```

```javascript
class AlgorithmA {
  //...

  // Find item to buy on first site, then buy it.
  const itemId = await findItemToBuy()
  const result = await retry(() => site1MarketInterface.buy(itemId))
  if (result.isSuccessful) {
    // Put same item on sale in another site.
    await retry(() => site2MarketInterface.putOnSale(prepareItemData(result)))
  }

  // ...
}
```

```javascript
class AlgorithmB {
  //...

  // Go through the list of items and buy each one.
  const results = items.map(item => retry(() => marketInterface.buy(item.it)))
  // Process results.

  // ...
}
```

This is much better, we have reusable function with single responsibility that we can use where ever we need. However now each algorithm needs to know how to use it, and this is another responsibility of the algorithm class. We can do a bit better. We can create class like this.

This is more object oriented version:

```javascript
class RetryMarketInterface {
  constructor(marketInterface) {
    this.marketInterface
  }
  async buy(itemId) {
    return retry(() => this.marketInterface.buy(itemId))
  }
  async putOnSale(item) {
    return retry(() => this.marketInterface.putOnSale(item))
  }
}
```

Or more functional version:

```javascript
function withRetry(marketInterface) {
  return {
    async buy(itemId) {
      return retry(() => this.marketInterface.buy(itemId))
    },
    async putOnSale(item) {
      return retry(() => this.marketInterface.putOnSale(item))
    }
  }
}
```

So we have made wrapper around market interface that also implements that same interface adding retry functionality to it.

Now in algorithms we can do this:

```javascript
  marketInterface = withRetry(marketInterface)
```

or (if you prefer object oriented way)

```javascript
  marketInterface = new RetryMarketInterface(marketInterface)
```

_(In another part of this work we'll see differences between these two approaches, and why it is important in all modern programming languages. For now we will use object oriented way, at least that is more natural for most popular programming languages.)_

If `marketInterface` was passed to constructor of `AlgorithmA`, and `AlgorithmB`, now we can do something like this:

```javascript
class AlgorithmA {
  constructor(site1MI, site2MI) {
    this.site1MarketInterface = site1MI
    this.site2MarketInterface = site2MI
  }
  //...

  // Find item to buy on first site, then buy it.
  const itemId = await findItemToBuy()
  const result = await this.site1MarketInterface.buy(itemId)
  if (result.isSuccessful) {
    // Put same item on sale in another site.
    await this.site2MarketInterface.putOnSale(prepareItemData(result))
  }

  // ...
}
```

```javascript
class AlgorithmB {
  constructor(marketInterface) {
    this.marketInterface = marketInterface
  }
  //...

  // Go through the list of items and buy each one.
  const results = items.map(item => this.marketInterface.buy(item.it))
  // Process results.

  // ...
}
```

Usage example:

```javascript
const marketInterface = new RetryMarketInterface(new SiteAMarketInterface())
const ab = new AlgorithmB(marketInterface)
```

This pattern is commonly referred to as the **Chain of Responsibility**. Depending on circumstances that slightly modify its behavior, it can also be recognized as the **Decorator** pattern (or sometimes called the **Wrapper** pattern). Essentially, we're enhancing the core behavior with additional features, without requiring the core behavior to be aware of these additions.

Within this framework, we have two dimensions to consider: different algorithms or usage scenarios, and diverse market interfaces. These dimensions can be combined in various ways, leveraging the flexibility of the `RetryMarketInterface`. This demonstrates a clear connection between **code reuse** and **single responsibility**. The `RetryMarketInterface` maintains a single responsibility, each market interface that implements the interface for different markets has a single responsibility, and the algorithms each has a single responsibility. With a single wrapper market interface (optional) and options for 2 algorithms and 3 distinct markets, we can generate `(1 + 1) * 2 * 3 = 12` unique combinations.

The advantages of this approach do not end here. We can introduce an additional dimension to the combination space. Our `retry` function has a bug—it could potentially lead to an infinite loop—thus, we should establish a limit on the number of retries. Moreover, we might wish to add a delay before initiating a retry, but not in all situations. Sometimes, we may want to experiment with both delayed and non-delayed retries (the ability to experiment is valuable for startups and established companies alike). Additionally, we might consider employing retries with an exponential backoff strategy, exponentially increasing delays between retries. We could implement these types of retries as `NTimesRetryMarketInterface`, `DelayRetryMarketInterface`, `ExponentialBackoffRetryMarketInterface`, and so forth.

Usage example:

```javascript
const marketInterface = new ExponentialBackoffMarketInterface(new SiteAMarketInterface())
const ab = new AlgorithmB(marketInterface)

const aa = new AlgorithmA(
  new DelayRetryMarketInterface(
    new SiteAMarketInterface(),
    2 /* seconds */),
  new NTimesRetryMarketInterface(
    new SiteBMarketInterface(),
    3 /* n retries */)
)
```

Now we have 3 retry wrapper interfaces, 3 real/core market interfaces, and 2 algorithms, and we can combine them all like Lego to get `(3 + 1) * 2 * 3 = 24` combinations.

Yet, there is one more dimension to add to it. There is yet another benefit. We can add other kinds of behaviors to core market interfaces. We can, for example, make `CheckBalanceMarketInterface` that checks balance, to avoid making buy request if there is not enough money. We can also make `DBLoggerMarketInterface`, and `ConsoleLoggerMarketInterface` that log all operations to database, or console. And we can use both of them at the same time. The possibilities are endless.

Important thing here is that all components have single responsibility.

Example:

```javascript
const marketInterface = new CheckBalanceMarketInterface(
  new ExponentialBackoffMarketInterface(
    new SiteAMarketInterface()))
const ab = new AlgorithmB(marketInterface)

const aa = new AlgorithmA(
  new DBLoggerMarketInterface(
    new CheckBalanceMarketInterface(
      new DelayRetryMarketInterface(
        new SiteAMarketInterface(),
        2 /* seconds */)
    dbParams),
  new ConsoleLoggerMarketInterface(
    new CheckBalanceMarketInterface(
      new NTimesRetryMarketInterface(
        new SiteBMarketInterface(),
        3 /* n retries */))
)
```

We can just add more and more behavior easily, and combine them as needed with great flexibility.

We can revert order of balance checking and retry wrappers easily after figuring out that maybe we want to retry later if we expect te receive some money.

Can you even imagine how implementing something like this would look like without the using reusable wrappers, each with a single responsibility, as we have done here? The resulting complexity would undoubtedly be a chaotic mess, wouldn't it?

### Java IO streams

This is a real-world example of the same pattern we just saw.
In the Java programming language, and probably in many other programming languages, `OutputStream` is the interface that can be implemented to point to a network connection, file, memory buffer, Unix pipe between two processes, or just about anything to which we can send bytes.

Then there are `FilteredOutputStream` subclasses like `BufferedOutputStream` that buffer written bytes (don't send them to the underlying stream immediately), `CipherOutputStream` that encrypts written bytes before writing them to the underlying stream, `DeflaterOutputStream` that compresses bytes before writing them to the underlying output stream, and many others.

So, if we need to implement our own output stream – for example, to send bytes to our custom hardware over a custom protocol – we only need to implement things specific to our protocol. If we, as developers of client software, need to add encryption, compression, or other byte processing, existing classes can be reused.

If we want to buffer data, then send it to compression, then encrypt it and send it to our device over our protocol, we can do this.

```java
OutputStream os = new BufferedOutputStream(
  new DeflaterOutputStream(
    new ChiperOutputSteam(
      new CustomOutputSteam(config),
      chipher
    )
    deflater
  )
);

os.write(bytes);
```

Notice that in all these examples, components are not just _reusable_, they are **replaceable**, like Lego.

This would be hard to accomplish using inheritance, for each combination you'd probably need a new class. Besides, composition offers runtime configuration, that can change dynamically in a flexible way. This is one of the examples why people prefer **composition over inheritance**.

There are also some classes that expand this interface with additional methods, for example `PrintOutputStream` that has `print` and `println` method to make it easier to send strings to output stream.

### CDN service

Let's look at the example based on true story.
A team had to upload images from backend to Content Delivery Network (CDN).
Each image had to be converted to few other formats and sizes and each had to be uploaded.

Code presented here violates just about any good programming principle, or at least all SOLID principles.

The team was in a hurry, of course, as always, so they made something like this:

```javascript
class CDNUpload {
  uploadImages(path, imageBytes) {
    const uploadConfig = {
      someParam: 'someValue',
      basePath: basePath,
      otherParam: 'otherValue',
    }

    const someVarRelatedToConversion = something + config.something
    const anotherVarRelatedToConversion = something + config.something
    const someVarRelatedToCDNProvider = proc(config.cdn.val1)
    const otherVarForCDNProvider = proc2(config.cdn.va2)

    const formatOptions = {
      opt1: val1
      opt2: val2
    }

    formats.forEach(f => {
      formatOptions.format = f.format

      const converted = await convertor
        .format(imageBytes, formatOptions)
        .resize(f.size)
      imagePath = someCondition
        ? (path + someData.join('/')).replace('one', 'two')
        : (path + someData.join('/'))

      data = {
        config: uploadConfig,
        path: imagePath,
        bytes: converted
      }

      return CDNProviderA.files.upload(data)
    })
  }
}
```

We see here that it violates **S**ingle responsibility principle.

The code resembles the code from the true story, but obviously here some lines are pseudo code. When those lines are copied in next examples, that means that they were really copied in the true story. Don't mind the details, as some parts are just there to illustrate complexity of the code, not to be meaningful. The real code was more complicated then the one presented here.

Then they've figured out it is better to save images to local directory when developing locally and use CDN in production.

So the code transformed into this:

```javascript
class CDNUpload {
  uploadImagesToCDNProviderA(path, imageBytes) {
    const uploadConfig = {
      someParam: 'someValue',
      basePath: basePath,
      otherParam: 'otherValue',
    }

    const someVarRelatedToConversion = something + config.something
    const anotherVarRelatedToConversion = something + config.something
    const someVarRelatedToCDNProvider = proc(config.cdn.val1)
    const otherVarForCDNProvider = proc2(config.cdn.va2)

    const formatOptions = {
      opt1: val1
      opt2: val2
    }

    formats.forEach(f => {
      formatOptions.format = f.format

      const converted = await convertor
        .format(imageBytes, formatOptions)
        .resize(f.size)
      imagePath = someCondition
        ? (path + someData.join('/')).replace('one', 'two')
        : (path + someData.join('/'))

      data = {
        config: uploadConfig,
        path: imagePath,
        bytes: converted
      }

      return CDNProviderA.files.upload(data)
    })
  }

  uploadImagesToLocalDirectory(path, imageBytes) {
    const someVarRelatedToConversion = something + config.something
    const anotherVarRelatedToConversion = something + config.something

    const formatOptions = {
      opt1: val1
      opt2: val2
    }

    formats.forEach(f => {
      formatOptions.format = f.format

      const converted = await imageTools
        .format(imageBytes, formatOptions)
        .resize(f.size)
      imagePath = someCondition
        ? (path + someData.join('/')).replace('one', 'two')
        : (path + someData.join('/'))

      imageTools.saveBufferToFile(imagePath, converted)
    })
  }

  uploadImages(path, imageBytes) {
    if (this.uploadType === 'CDNProviderA') {
      return await this.uploadImagesToCDNProviderA(path, imageBytes)
    } else {
      return await this.uploadImagesToLocalDirectory(path, imageBytes)
    }
  }
}
```

This violates **O**pen-Closed principle.

To make things worse, this is how it was supposed to be called:

```javascript
class SomeClass {
  getImageUploadPath() {
    return config.cdnProvider == 'CDNProviderA' ? '' : config.localCDNDirPath
  }

  someOtherMethod() {
    // ...
    cdnUpload.uploadImages(this.getImageUploadPath() + subpath, imageBuffer)
    // ...
  }
}
```

This violates **L**iskov substitution principle.

We see that who ever wanted to upload an image, needs to know about local and production CDN, and has to take care of adjusting path to it. Why it is not encapsulated into `CDNUpload` class is not known.

Then there was need to upload audio file too. So we got this code:

```javascript
class CDNUpload {
  uploadImagesToCDNProviderA(path, imageBytes) {
    const uploadConfig = {
      someParam: 'someValue',
      basePath: basePath,
      otherParam: 'otherValue',
    }

    const someVarRelatedToConversion = something + config.something
    const anotherVarRelatedToConversion = something + config.something
    const someVarRelatedToCDNProvider = proc(config.cdn.val1)
    const otherVarForCDNProvider = proc2(config.cdn.va2)

    const formatOptions = {
      opt1: val1
      opt2: val2
    }

    formats.forEach(f => {
      formatOptions.format = f.format

      const converted = await convertor
        .format(imageBytes, formatOptions)
        .resize(f.size)
      imagePath = someCondition
        ? (path + someData.join('/')).replace('one', 'two')
        : (path + someData.join('/'))

      data = {
        config: uploadConfig,
        path: imagePath,
        bytes: converted
      }

      return CDNProviderA.files.upload(data)
    })
  }

  uploadImagesToLocalDirectory(path, imageBytes) {
    const someVarRelatedToConversion = something + config.something
    const anotherVarRelatedToConversion = something + config.something

    const formatOptions = {
      opt1: val1
      opt2: val2
    }

    formats.forEach(f => {
      formatOptions.format = f.format

      const converted = await imageTools
        .format(imageBytes, formatOptions)
        .resize(f.size)
      imagePath = someCondition
        ? (path + someData.join('/')).replace('one', 'two')
        : (path + someData.join('/'))

      imageTools.saveBufferToFile(imagePath, converted)
    })
  }

  uploadImages(path, imageBytes) {
    if (this.uploadType === 'CDNProviderA') {
      return await this.uploadImagesToCDNProviderA(path, imageBytes)
    } else {
      return await this.uploadImagesToLocalDirectory(path, imageBytes)
    }
  }

  uploadAudioToCDNProviderA(path, audioBuffer) {
    const uploadConfig = {
      someParam: 'someValueAudio',
      basePath: basePath,
      otherParam: 'otherValueAudio',
    }

    const someVarRelatedToCDNProvider = proc(config.cdn.val3)
    const otherVarForCDNProvider = proc2(config.cdn.va4)

    audioPath = someCondition
      ? '/audio' +(path + someData.join('/')).replace('one', 'two')
      : '/audio' + (path + someData.join('/'))

    data = {
      config: uploadConfig,
      path: audioPath,
      bytes: audioBuffer
    }

    return CDNProviderA.files.upload(data)
  }

  uploadAudioToLocalDirectory(path, buffer) {
    audioPath = someCondition
      ? '/audio' +(path + someData.join('/')).replace('one', 'two')
      : '/audio' + (path + someData.join('/'))

    // This too, is pseudo code.
    if ('mv' in buffer && typeof buffer.mv === 'function') {
        buffer.mv(audioPath)
        if (err) {
          throw err
        }
        console.log(`File ${fileName} uploaded locally to ${audioPath}`)
      } else {
        const err = fs.writeFile(audioPath, buffer.data)
        if (err) {
          throw err
        } else {
          console.log(`File ${fileName} uploaded locally to ${audioPath}`)
        }
      }
  }

  uploadAudios(path, audioBuffer) {
    if (this.uploadType === 'CDNProviderA') {
      return await this.uploadAudioToCDNProviderA(path, audioBuffer)
    } else {
      return await this.uploadAudioToLocalDirectory(path, audioBuffer)
    }
  }
}
```

This violates **I**nterface segregation principle.

Given direct dependency on image conversion library, it also violated **D**ependency inversion principle.

It is not critical if we violate some of the principles, sometimes, but violating them all is. This is typical example of a task that can easily be implemented properly, which we're going to do shortly.

The good thing is that there was no need to implement support for another CDN provider, as that would obviously be nightmare to do. Just image all those conditions `config.cdnProvider == 'CDNProviderA' ? '' : config.localCDNDirPath`,
there was 12 of them spread across the project.

But there was need to upload simple PDF file. Doing this in the spirit of existing code would mean copying function for audio file upload and altering few words, and names of the functions. There is no way to reuse existing code, other then copying it, and analyzing for things that need to be altered.

The team did this in a hurry to meet goals and deadlines. Soon, however, other members of the team had to implement upload of other kinds of files. That other members initially thought that everything is set up for them, and that all it takes is to reuse that existing code especially given that they were inexperienced with CDN and how it works. At the end, a lot of time was wasted trying to understand the code, then to untangle parts that are related to upload itself, and other parts that are related to e.g. image processing.

Let's see now a better way to implement it.

```javascript
class CDNProviderAUpload {
  upload(path, buffer) {
    const uploadConfig = {
      someParam: 'someValueAudio',
      basePath: basePath,
      otherParam: 'otherValueAudio',
    }

    const someVarRelatedToCDNProvider = proc(config.cdn.val3)
    const otherVarForCDNProvider = proc2(config.cdn.va4)

    audioPath = someCondition
      ? '/audio' +(path + someData.join('/')).replace('one', 'two')
      : '/audio' + (path + someData.join('/'))

    data = {
      config: uploadConfig,
      path: audioPath,
      bytes: buffer
    }

    return CDNProviderA.files.upload(data)
  }
}

class LocalCDNUpload {
  upload(path, buffer) {
    destPath = someCondition
      ? '/audio' +(path + someData.join('/')).replace('one', 'two')
      : '/audio' + (path + someData.join('/'))

    // No more need to do this outside this class.
    destPath = config.localCDNDirPath + destPath

    // This too, is pseudo code.
    if ('mv' in buffer && typeof buffer.mv === 'function') {
        const err = buffer.mv(destPath)
        if (err) {
          throw err
        }
        console.log(`File ${fileName} uploaded locally to ${destPath}`)
      } else {
        const err = fs.writeFile(destPath, buffer.data)
        if (err) {
          throw err
        } else {
          console.log(`File ${fileName} uploaded locally to ${destPath}`)
        }
      }
  }
}
```

Now we have two classes that (implicitly) implement same interface, and we can use them interchangeably. We no longer need to check which implementation is it when we use it. We can have one function that initializes one of these two implementations depending on config.

```javascript
function getCDNUpload() {
  return config.cdnProvider == 'CDNProviderA'
    ? new CDNProviderAUpload()
    : new LocalCDNUpload()
}
```

```javascript
// ...
  getCDNUpload().upload(path, audioFile)
// ...
  getCDNUpload().upload(path, PDFFile)
// ...
```

For images, we need to convert them to different formats and sizes, and upload all of them, so we will create class that has that additional method. We can implement same interface as other classes, with `upload` method and add additional method, or we can have completely new interface with only this, new method.

```javascript
class ImageConverter {
  constructor() {
    this.someVarRelatedToConversion = something + config.something
    this.anotherVarRelatedToConversion = something + config.something

    this.formatOptions = {
      opt1: val1
      opt2: val2
    }
  }

  convert(buffer) {
    return formats.map(f => {
      this.formatOptions.format = f.format

      const converted = await imageTools
        .format(imageBytes, this.formatOptions)
        .resize(f.size)
      imagePath = someCondition
        ? (path + someData.join('/')).replace('one', 'two')
        : (path + someData.join('/'))

      return {imageBuffer: converted, format}
    })
  }
}

class ImagesUpload {
  constructor() {
    this.imageConverter = new ImageConverter()
    this.cdnUpload = getCDNUpload()
  }

  uploadImages(path, buffer) {
    this.imageConverter.convert(buffer).forEach(imgData => {
      const imagePath = this.getImagePath(imgData.format, path)
      this.cdnUpload.upload(imagePath, imgData.imageBuffer)
    })
  }

  getImagePath(format, path) {
    //...
  }
}
```

```javascript
  getImagesUpload().uploadImages(path, buffer)
```

Again we have classes with single responsibility, so that the team can leverage things that are already implemented to move faster rather than being slowed down by technical debt.

### Gesture Web Site

Let's say we're creating a website with a unique UI and UX. The application's UI is divided into several main components known as `Blocks`, where each block has its own logic. All blocks have a method called `onEvent` which is designed to handle various events from the system, including UI-related events.

One of the blocks might be structured as follows:

```javascript
class BlockA {
  // ..

  onEvent(event) {
    if (event.hasSomeProp()) {
      doSomething()
    }

    if (isOfSomeKind(event)) {
      handleEventOfThatKind(event)
    } else if (ofAnotherKind(event)) {
      handleEventOfAnotherKind(event)
    }
  }

  // ..
}
```

Let's consider a scenario where we receive a specific event when a user performs a gesture using a mouse or a finger. The UI library sending these events doesn't understand the gesture type; it provides us with timing and coordinates in a raw format. Our task involves processing this data and determining the nature of the gesture and the subsequent actions to be taken. It's possible for a single gesture to trigger multiple handlers within a block. For instance, it could trigger both a data refresh and an independent visual effect.

Suppose that we're already handling some standard gestures within the functions `handleSomething`, but that code is tailored specifically to those known gestures.

Whether this approach is considered good or bad is not crucial here, let's assume it's given to us. What's crucial is how we respond to change requests, considering that we've observed in other examples that well written code can gradually become overly complex when modified incrementally.

Imagine we receive a request to implement new functionality: when a user forms an infinity sign gesture, the block should pop out of the screen. Our initial idea might be to approach it in the following manner:

```javascript
class BlockA {
  // ..

  onEvent(event) {
    if (event.hasSomeProp()) {
      doSomething()
    }

    if (isInfinityGesture(event)) {
      this.popout()
    }

    if (isOfSomeKind(event)) {
      handleEventOfThatKind(event)
    } else if (ofAnotherKind(event)) {
      handleEventOfAnotherKind(event)
    }
  }

  isInfinityGesture(event) {
    const rawGestureData = event.getGestureData()
    result = false
    // Some initialization here.
    for (let i = 0; i < rawGestureData.length; ++i) {
      /*
       * Complicated code goes here, and possible changes
       * result to true.
       */
    }

    return result
  }

  // ..
}
```

That would be a reasonable starting point. Perhaps we should fine-tune the infinity gesture detection algorithm, experiment with different approaches, and commit the changes once we're satisfied with the results.

It's important not to overlook the principles of single responsibility and code reuse. Being proactive, let's initiate a refactoring session. Although we've explored several methods for implementing the infinity gesture detection, none of them have proven entirely satisfactory.Somehow we have managed to come up with something that is good enough. However, we assume that in the future we'd like to improve it. Maybe make AB test with two or more implementations.

To address this, let's extract the responsibility of infinity gesture detection from our current class.

```javascript
class BlockA {
  constructor(/*..., */ infinityDetector) {
    // ...
    self.infDetector = infinityDetector
  }
  // ..

  onEvent(event) {
    if (event.hasSomeProp()) {
      doSomething()
    }

    if (isInfinityGesture(event)) {
      this.popout()
    }

    if (isOfSomeKind(event)) {
      handleEventOfThatKind(event)
    } else if (ofAnotherKind(event)) {
      handleEventOfAnotherKind(event)
    }
  }

  isInfinityGesture(event) {
    const rawGestureData = event.getGestureData()
    return this.infDetector.isInfinityGesture(rawGestureData)
  }

  // ..
}

class InfinityDetectorA {
  isInfinityGesture(rawGestureData) {
    result = false
    // Some initialization here.
    for (let i = 0; i < rawGestureData.length; ++i) {
      /*
       * Complicated code goes here, and possible changes
       * result to true.
       */
    }

    return result
  }
}
```

Here we have used classical **composition** with delegation of responsibility to other classes, thus implementing **inversion of control** via **dependency injection**. You could also say that this is example of **strategy pattern**.

Now, what we can do, we can implement more than one `InfinityDetector` with different algorithms.

- We can implement `InfinityDetectorA` and `InfinityDetectorB` and try them both in AB test.
- We can also reuse them in other classes that need them, make some other `BlockB` also use `InfinityDetectorA`
- We can use one of them in one place, and another in another place. For example, `BlockB` can use `InfinityDetectorA`, a `BlockA` can use `InfinityDetectorB`, or the other way around, or we could benefit from reusability of the detectors, and use `InfinityDetectorA` in both cases.
- We might use `BlockA` in two places, and in one place we can use `InfinityDetectorA` (passing it to constructor of `BlockA`), and in another place we can use `InfinityDetectorB`.

Again, this is one more example, how components that have **single responsibility** can be **reused** in different places **interchangeably**, like lego blocks. That makes our code really flexible.

Of course, we could do this refactoring when the need for that arises, but it is also good to proactively prevent bad things in the future or direct development in the right direction. That is especially true if the change is not complicated, as in this example, and it does look like we could use that reusability.

Don't underestimate this technique, as it is widely used. This example might seem strange, because in the real world, we might want to try different gestures instead of the infinity gesture. We could also have a more powerful tool that can tell us which gesture it is out of tens of gestures that it can recognize. Just remember that this is just an example, and we need to think about each real-world case separately and consciously make decisions. There is no golden rule that solves all problems.

However, we will see how this same approach can help us in that case too.

First, let's say that we have a library that can tell us what gesture it is.

The initial code could look like this:

```javascript
class BlockA {
  // ..

  onEvent(event) {
    if (event.hasSomeProp()) {
      doSomething()
    }

    if (gestureLib.detect(event.getRawGestureData()) === 'infinity') {
      this.popout()
    }

    // ...
  }

  // ..
}
```

With our approach it would look something like this.

```javascript
class InfinityDetectorA {
  isInfinityGesture(rawGestureData) {
    return gestureLib.detect(rawGestureData) === 'infinity'
  }
}
```

In case we are not really sure that we actually need infinity as the gesture (maybe the "circle" gesture is better?), but want to make it configurable, initial code would look like this.

```javascript
class BlockA {
  constructor(/*..., */ popupGestureName) {
    // ...
    self.popupGestureName = popupGestureName
  }
  // ..

  onEvent(event) {
    if (event.hasSomeProp()) {
      doSomething()
    }

    if (gestureLib.detect(event.getRawGestureData()) === this.popupGestureName) {
      this.popout()
    }
    // ...
  }

  // ..
}
```

Our code would change like this. First instead of having `InfinityDetector` interface we would have `GestureDetector` with method `detect(rawGestureData)` that would return true in case provided gesture data is what we're looking for, what ever it is.

In TypeScript we would have interface like this:

```typescript
interface GestureDetector {
  detect(rawGestureData: GesturePoint[]): bool;
}
```

```javascript
class BlockA {
  constructor(/*..., */ popupGestureDetector) {
    // ...
    self.popupGestureDetector = popupGestureDetector
  }
  // ..

  onEvent(event) {
    if (event.hasSomeProp()) {
      doSomething()
    }

    if (isPopupGesture(event)) {
      this.popout()
    }

    if (isOfSomeKind(event)) {
      handleEventOfThatKind(event)
    } else if (ofAnotherKind(event)) {
      handleEventOfAnotherKind(event)
    }
  }

  isPopupGesture(event) {
    const rawGestureData = event.getGestureData()
    return this.popupGestureDetector.detect(rawGestureData)
  }

  // ..
}

class InfinityDetectorA {
  detect(rawGestureData) {
    return gestureLib.detect(rawGestureData) === 'infinity'
  }
}
```

```javascript
const block1 = new BlockA(/*...*/, new InfinityDetectorA())
const block2 = new BlockA(/*...*/, new CircleDetectorA())
```

Now we can experiment with lib or without lib (using our custom algorithm), and dependency on `gestureLib` is not spread across the project, but localized inside the implementation of specific detectors.

To make things a bit more interesting and reusable let's **refactor** our code into this:

```javascript
class GestureLibGestureDetector {
  constructor(gestureName) {
    this.gestureName = gestureName
  }
  detect(rawGestureData) {
    return gestureLib.detect(rawGestureData) === this.gestureName
  }
}

function createGestureDetector(gestureName) {
  if (gestureName === 'hrkljus') {
    // gestureLib does not now about this strange gesture,
    // but we need it, so we have our own implementation.
    return new HrkljusGestureDetector()
  }

  return GestureLibGestureDetector(gestureName)
}
```

```javascript
const block1 = new BlockA(/*...*/, createGestureDetector('infinity'))
const block2 = new BlockA(/*...*/, createGestureDetector('circle'))
const block3 = new BlockB(/*...*/, createGestureDetector('hrkljus'))
```

Depending on the situation different solutions can fit better or worse, but the core principle remains the same.

### Filter users

We have this code:

```javascript
function getDailyActiveUsers() {
  const dailyActiveUsers = []
  const users = getAllUsers()
  const now = Date.now()

  for (let i = 0; i < users.length; ++i) {
    const lastActivity = users[i].getLastActivityTimestamp()
    if (now - lastActivity < 24 * 60 * 60 * 1000) {
      dailyActiveUsers.push(users[i])
    }
  }

  return dailyActiveUsers
}

function getLongerAudioTracks(lengthMs) {
  const longAudios = []
  const audios = getAudioTracks()

  for (let i = 0; i < audios.length; ++i) {
    const len = audios[i].samples.length * 1000 / audios[i].sampleRate
    if (len > lengthMs) {
      longAudios.push(audios[i])
    }
  }

  return longAudios
}
```

Can you see what responsibilities each of these functions have?

Let's try to refactor it like this:

```javascript
function filter(array, predicate) {
  const result = []
  for (let i = 0; i < array.length; ++i) {
    if (predicate(array[i])) {
      result.push(array[i])
    }
  }
  return result
}

function wasUserActiveInLast24h(user) {
  const now = Date.now()
  const lastActivity = user.getLastActivityTimestamp()
  return now - lastActivity < 24 * 60 * 60 * 1000
}

function computeAudioLengthMs(audio) {
  return audio.samples.length * 1000 / audio.sampleRate
}

// ----- //

function getDailyActiveUsers() {
  const users = getAllUsers()
  return filter(users, wasUserActiveInLast24h)
}

function getLongerAudioTracks(lengthMs) {
  const audios = getAudioTracks()
  return filter(
    audios,
    audio => computeAudioLengthMs(audio) > lengthMs
  )
}
```

Which version is better?

Maybe you like first version better, but can you name objective (not subjective) benefits of it?

The second version however has following benefits:

- `filter` is reusable (thanks to having only one responsibility), when ever we need to apply that operation we can use `filter` function to save some time and characters. In other languages it is often possible to filter over different data structures, and it doesn't matter which underlying data structure it is, interface is always the same, giving more reusability to filtering code.
- `wasUserActiveInLast24h` is reusable (thanks to having only one responsibility), we can use it in other places too, and we don't have to think about how to actually check if user was active in the last 24h, we just call the function. When we do use it, it is immediately clear what it does thanks to naming.
- `computeAudioLengthMs` is reusable (thanks to having only one responsibility), we can use it in other places too, and the name clearly says what it does so it can probably save few milliseconds of understanding code while reading it.

This example also illustrates that separating concerns has to do with an old programming technique for tackling complex problems - breaking tasks into simpler ones, which can be solved independently.

To be honest there is one major advantage of the first version - depending on JavaScript engine, it can be more performant, functions there run faster. This is partially due to limitations of JavaScript and its implementations. Partially is it characteristic of programming in any programming language. Sometimes, there is trade of between writing readable code, following good programming principles and making code performant. Modern programming languages are trying to tackle that issue by using smart compiler optimizations, or features like lazy evaluation, however it is still an issue. Usually it is not issue big enough, unless we get into a state of [uniformly slow code](https://wiki.c2.com/?UniformlySlowCode). However, usually the most effective way to improve performance is to find and fix bottlenecks. That requires making changes to the code, and as we already saw, making changes is easier when the code follows good programming principles. Thus improving performance of a program is usually easier when the code base respects single responsibility principle.

Of course we can write filter like this:

```javascript
Array.prototype.filter = function (predicate) {
  const result = []
  for (let i = 0; i < this.length; ++i) {
    if (predicate(this[i])) {
      result.push(this[i])
    }
  }
  return result
}
```

and use it like this

```javascript
function getDailyActiveUsers() {
  const users = getAllUsers()
  return users.filter(wasUserActiveInLast24h)
}

function getLongerAudioTracks(lengthMs) {
  const audios = getAudioTracks()
  return audios.filter(
    audio => computeAudioLengthMs(audio) > lengthMs
  )
}
```

Which is, of course, already built into JavaScript.

Most languages have small algorithms abstracted away into reusable components. One of the most famous examples is the C++ Standard Template Library and its `algorithm` header. JavaScript is not the best example, as it does not offer a wide variety of data structures (there is `Iterator` with implementations like `Array` and `Map`, but you can build your own too) and parallel processing. In other languages like C++, Java, (especially) Scala, etc., abstracting these simple algorithms allows us to write a single algorithm and apply it to different data structures, or use parallel processing to speed up the algorithm without changing the algorithm itself (which helps keep the algorithm simple).

People usually rely on reusable components provided by libraries or built-in components, but do not build their own. This example illustrates how we can build our own reusable algorithms and other components (of course, do not reinvent the wheel; build new reusable components that do not already exist).

### Grouping items the reusable way

Prior to introducing `groupBy` (which is still experimental feature in JavaScript at the time of writing), to group elements by some key you'd have to write small algorithm like this:

```javascript
const array = [
  {a: 1, b: 'x'},
  {a: 2, b: 'y'},
  {a: 3, b: 'x'},
  {a: 4, b: 'x'},
  {a: 5, b: 'y'},
]

// grouping by key 'b'
const result = {}
for (let e of array) {
  if (!result[e.b]) {
    result[e.b] = []
  }
  result[e.b].push(e)
}

console.log(result)

/*
{
  x: [{a: 1, b: 'x'}, {a: 3, b: 'x'}, {a: 4, b: 'x'} ],
  y: [{a: 2, b: 'y'}, {a: 5, b: 'y'}]
}
*/
```

But, if you have to do it more often, you can consider making this small algorithm reusable.

```javascript
function groupBy(iterable, keyGetter) {
  const result = {}

  for (let e of iterable) {
    const key = keyGetter(e)
    if (!(key in result)) {
      result[key] = []
    }
    result[key].push(e)
  }

  return result
}

const array = [
  {a: 1, b: 'x'},
  {a: 2, b: 'y'},
  {a: 3, b: 'x'},
  {a: 4, b: 'x'},
  {a: 5, b: 'y'},
]

const result = groupBy(array, e => e.b)

console.log(result)
```

We can also use it with other data structures that implement iterator protocol like `Map`.

```javascript
const map = new Map()
map.set('a', 'x')
map.set('b', 'x')
map.set('c', 'y')
map.set('d', 'y')
map.set('e', 'x')

console.log(groupBy(map, ([k, v]) => v))
/*
{
  x: [['a', 'x'], ['b', 'x'], ['e', 'x']],
  y: [['c', 'y'], ['d', 'y']]
}
*/
```

We can even use it to solve the task from the introduction:

```javascript
function separate(numbers) {
  return groupBy(numbers, n => n % 2 == 0 ? 'even' : 'odd')
}

separate([1, 2, 3, 4, 5, 6, 7])

/*
{ odd: [ 1, 3, 5, 7 ], even: [ 2, 4, 6 ] }
*/

```

Yes, that's it.

We can do that with other algorithms that are common in our codebase, even if we use them just once. This approach will help us steer our code in the right direction and enable us to write code with better test coverage. If the algorithm is not specific to our project (like `groupBy`), and a better implementation is provided by the standard library at some point, we can switch to that implementation easily.

## Conclusion

We've seen examples of small reusable parts like `filter`, as well as larger ones like `UploadService`. The larger the reusable part the better. If we could make whole module reusable that would be the best. Sometimes people extract modules out of their project and publish them as reusable libraries. Libraries are reusable by definition. When we develop software, we should try to develop libraries and applications as applications of those libraries, to achieve ultimate reusability.

Hopefully, the **relationship between code reusability and the single responsibility principle**, as well as why that is important, is clearer now.

Don't forget that you don't need to make it right immediately, but do remember to refactor proactively to prevent bad code and direct development in the right direction.

In the examples above, we've mostly used the wrapper and strategy patterns, but it should be enough to kick-start your imagination. Just remember to separate concerns, extract responsibilities, until there is only one left in the unit of code. To learn more about design patterns, you can read other books that are dedicated to that subject.

## Considering modules

All said previously applies not only to unit of code but also to bigger modules as well. In fact, it applies to bigger modules even more so then to small units of code. We can allow our selves to write few small units of code in a bad way, if effects of that code are localized. The way bigger modules are written has much bigger consequences, and affects larger part of the project.

It is much harder to write examples involving whole projects, so let's just see few more things to have in mind when it comes to bigger modules:
* It is hard to say what is single responsibility on the scale of module, because modules are usually made as toolboxes, able to accomplish many tasks. It is preferable that all those tasks make meaningful group. Another technique to make sure module has only one principle is to avoid too many dependencies on other modules and their internals, but instead aim at making module reusable by many other modules, at least in perspective. One good practice that can lead us in that direction is avoiding circular dependencies between modules, and defining strict rules which modules can depend on which other modules.
* Separating units of code, so that some units of code do not know, and thus can not depend on some other units, is important. It is, however, not nearly important as defining borders and dependencies between modules (or microservices, if you like).
* Code duplication is bad, sometimes even if one line is duplicated, but duplicating functionality of modules is terrible. It is however OK to have duplicate small units in two distinct modules. Maybe those units will have separate paths of evolution in the future, especially if they are made by different teams. Otherwise, those duplicated units could become one Frankenstein unit of code desperately trying to meet demands of both teams and both modules.
* Building two large modules, in such way that one should depend on another, but that other one should not know about the first one, is the way to make modules reusable. Otherwise we have circular dependency. It can be explicit, e.g. A imports classes from B, and B also imports classes from A. We should be aware of implicit dependencies. Just because in B we don't import classes from A does not mean that B does not depend on A. If B emits an event, with such data, in such place, and time, that it is particularly designed to be handled by some class in A, if it is specially crafted for internals of A, them B depends on A, even though not explicitly.
* The bigger modules are, the more borders between them should follow borders between departments in the company, and teams that work on the project.
