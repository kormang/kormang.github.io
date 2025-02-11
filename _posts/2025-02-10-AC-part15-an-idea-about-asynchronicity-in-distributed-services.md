---
layout: post
author: kormang
---

A particular kind of distributed system that is popular now consists of backend systems based on microservices or those dependent on external services that communicate via the HTTP protocol, or let's say gRPC.

Let's consider an example involving a payment service, which can either be a microservice within our own project or an external payment service provider, and a delivery service. Imagine we have a marketplace. When a user wants to purchase something, they make a payment, and then we wait for the payment to be confirmed via a webhook. After confirmation, we send instructions to the delivery service, which is responsible for physically delivering the product to the user's address. Webhooks are a form of callbacks represented by HTTP endpoints, which other services use to make requests, notifying our service that something has happened, such as the completion of a payment. Let's take a look at what the (pseudo)code might look like.

```typescript
  // This function handles HTTP request to endpoint 'payment/confirm-webhook'.
  // Requests are made by payment service when payment has been completed.
  @Post('payment/confirm-webhook')
  async webhook(req: Request, res: Response) {

    const order = await this
      .getOrderRepository()
      .findById(req.metadata.order_id);

    const {deliveryId} = await this.getDeliveryService().startDelivery(
      req.metadata.order_id,
      order.product_id,
      order.product_details,
      order.delivery_address
    );

    await this.getOrderRepository().update(
      req.metadata.order_id,
      {status: "PAYMENT_COMPLETED", delivery_id: deliveryId}
    );

    res.status(200).json({success: true});
  }

  // This function handles HTTP request to 'delivery/confirm-delivery'.
  // Requests are made by delivery service upon successful delivery.
  @Post('delivery/confirm-delivery')
  async webhook(req: Request, res: Response) {
    const order = await this
      .getOrderRepository()
      .findBy({delivery_id: req.deliveryId});

    await this.getOrderRepository().update(
      order.id,
      {status: "DELIVERY_COMPLETED"}
    );

    // Do something additional with order, e.g. emit some event.

    res.status(200).json({status: "OK"});
  }

  // This function handles POST HTTP request to 'orders'.
  // Requests are made by frontend when user wants to make a purchase.
  @Post('orders')
  async webhook(req: Request, res: Response) {
    const order = await this.getOrderRepository().create(
      {
        status: "INITIAL",
        product_id: req.product_id,
        product_details: req.product_details,
        delivery_address: req.delivery_address
      }
    );

    const product = await this.getProductRepository().findById(req.product_id);

    // Communicate with payment service.
    // When payment gets completed, payment service will call the webhook.
    const payment = await this.getPaymentService().startPayment({
      metadata: {order_id: order.id},
      amount: product.price * req.quantity
    });

    res.status(200).json({status: "OK", payment_id: payment.id});
  }

```

In the previous code, there is no error handling at all, so it has been simplified to its bare minimum. However, it's crucial to emphasize that error handling is of utmost importance. In the existing code, we do not perform necessary checks and only follow the "happy path".

This code shares a common problem with the algorithm implemented in the `on_character` function in the [Layers of Abstraction](./2025/01/12/AC-part2-layers-of-abstraction.html#application-using-event-handlers) part. We are required to manually maintain the state, essentially implementing a state machine. The key difference here is that we store the state in a database rather than in variables in memory. Consequently, this approach makes error handling challenging, performing necessary checks difficult, and most importantly, it prevents us from thinking locally. To fully comprehend the code and the potential consequences of any changes to it, we must analyze a significant portion of it, which is spread across multiple functions called at various points in time, potentially even residing in different files and directories. The ideal code would roughly resemble the following structure:

```typescript
  // This function handles POST HTTP request to 'orders'.
  // Requests are made by frontend when user wants to make a purchase.
  @Post('orders')
  async webhook(req: Request, res: Response) {
    const product = await this.getProductRepository().findById(req.product_id);

    // Communicate with payment service.
    // When payment gets completed, payment service will call the webhook.
    const payment = await this.getPaymentService().startPayment({
      metadata: {order_id: order.id},
      amount: product.price * req.quantity
    });
    const paymentResult = await this.waitForPaymentWebhook(payment.id);

    if (!paymentRequest.successful) {
      return res.status(400).json({status: "Failed"});
    }

    const {deliveryId} = await this
      .getDeliveryService()
      .startDelivery(
        order.id,
        req.product_id,
        req.product_details,
        req.delivery_address
      );
    const deliveryResult = await this.waitDeliveryResult(deliveryId);

    if (!deliveryResult.successful) {
      return res.status(400).json({status: "Failed"});
    }

    await this.getOrderRepository().create(
      {
        status: "DELIVERY_COMPLETED",
        product_id: req.product_id,
        product_details: req.product_details,
        delivery_address: req.delivery_address
      }
    );

    res.status(200).json({status: "OK"});
  }

```

Achieving this ideal code structure is not currently feasible due to several challenges. Firstly, HTTP requests must be serviced promptly, and other tasks should occur in the background. Furthermore, since various processes occur over an extended period and potentially on different machines (e.g., payment processing on another service), this function should maintain local state across server restarts. While this concept is appealing, it necessitates a different programming model to implement effectively.
