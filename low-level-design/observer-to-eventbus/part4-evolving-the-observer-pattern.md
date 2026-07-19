# Evolving the Observer Pattern - Part 4
# Building an Event-Driven Application Using Our EventBus

By the end of the previous article, we had built a fully functional EventBus capable of registering listeners, publishing events, and dispatching them in a type-safe manner.

While the implementation was complete, it still existed in isolation. We had built a piece of infrastructure, but we hadn't yet seen how it fits into a real application.

In this article, we'll build a small order processing system around our EventBus. More importantly, we'll see how the EventBus changes the way we think about application design. Instead of services coordinating every action themselves, they will simply publish facts about what has happened and allow interested components to react independently.

By the end of the article, we'll have a complete event-driven application consisting of services, listeners, domain events, and a single shared EventBus.

---

# The Architecture

Let's begin with the architecture we are trying to build.

```

                   EventBus
                       ▲
        ┌──────────────┼──────────────┐
        │              │              │
  OrderService   PaymentService  ShipmentService
        │              │
        │ publish      │ publish
        ▼              ▼
 OrderPlacedEvent  PaymentCompletedEvent
        │              │
        ├──────────┐   ├──────────────┐
        ▼          ▼   ▼              ▼
     Email    Analytics Invoice    Fraud Detection
```

The important observation here is that services no longer communicate with one another directly.

`OrderService` doesn't know about `EmailListener`.

It doesn't know about analytics.

It doesn't know about inventory.

It simply announces that an order has been placed.

Everything else happens independently.

---

# Creating the Domain

We'll keep our domain intentionally small.

```java
public record Order(
        long id,
        String customer,
        double amount
) {
}
```

```java
public record Payment(
        long orderId,
        double amount
) {
}
```

These classes aren't particularly interesting. They merely provide enough information for our events.

---

# Implementing Listeners

Each listener should have exactly one responsibility.

The EmailListener only sends confirmation emails.

```java
public class EmailListener
        implements EventListener<OrderPlacedEvent> {

    @Override
    public void onEvent(OrderPlacedEvent event) {

        System.out.printf(
                "Sending confirmation email for Order %d%n",
                event.order().id());

    }

}
```

Analytics is equally simple.

```java
public class AnalyticsListener
        implements EventListener<OrderPlacedEvent> {

    @Override
    public void onEvent(OrderPlacedEvent event) {

        System.out.printf(
                "Recording analytics for Order %d%n",
                event.order().id());

    }

}
```

Warehouse updates inventory.

```java
public class WarehouseListener
        implements EventListener<OrderPlacedEvent> {

    @Override
    public void onEvent(OrderPlacedEvent event) {

        System.out.printf(
                "Reserving inventory for Order %d%n",
                event.order().id());

    }

}
```

Notice something interesting.

None of these listeners know about each other.

They don't communicate.

They don't coordinate.

Each one performs a single action and returns.

This is one of the biggest advantages of event-driven systems. Responsibilities become naturally isolated.

---

# Building the Services

The OrderService has become remarkably small.

```java
public class OrderService {

    private final EventBus eventBus;

    public OrderService(EventBus eventBus) {
        this.eventBus = eventBus;
    }

    public void placeOrder(Order order) {

        System.out.printf(
                "Saving Order %d%n",
                order.id());

        eventBus.publish(
                new OrderPlacedEvent(order));

    }

}
```

Compare this to the implementation we started with in Part 1.

Back then, the service explicitly called

- EmailService
- AnalyticsService
- WarehouseService

Today, it knows nothing about them.

Its only responsibility is to save the order and publish an event describing what happened.

That is a much cleaner separation of concerns.

---

# Wiring Everything Together

One question remains.

Where should listeners be registered?

Certainly not inside the EventBus.

And certainly not inside individual services.

The best place is the composition root of the application, where all infrastructure is assembled.

```java
public class Main {

    public static void main(String[] args) {

        EventBus eventBus = new EventBus();

        eventBus.register(
                OrderPlacedEvent.class,
                new EmailListener());

        eventBus.register(
                OrderPlacedEvent.class,
                new AnalyticsListener());

        eventBus.register(
                OrderPlacedEvent.class,
                new WarehouseListener());

        OrderService orderService =
                new OrderService(eventBus);

        Order order =
                new Order(
                        101,
                        "Alice",
                        1250);

        orderService.placeOrder(order);

    }

}
```

Running the application produces

```
Saving Order 101

Sending confirmation email for Order 101

Recording analytics for Order 101

Reserving inventory for Order 101
```

Notice that the OrderService never directly invokes any of these listeners. It simply publishes an event, and the EventBus delivers it to every interested component.

---

# What We've Gained

The implementation is not significantly shorter than the direct method calls we started with. In fact, we've introduced more classes and a layer of indirection.

That is an important observation.

Design patterns rarely reduce the amount of code.

Instead, they improve the structure of the code.

Adding a new responsibility no longer requires modifying the OrderService. We simply implement another listener and register it with the EventBus.

Likewise, existing listeners remain completely independent of one another. Email failures cannot accidentally affect analytics because the listeners have no knowledge of each other.

The EventBus has become the central piece of infrastructure responsible for delivering events, while the services remain focused entirely on business logic.

---

# Where This Design Starts to Break Down

Although our EventBus is now perfectly usable, it is still intentionally simple.

Every listener executes synchronously on the calling thread. If one listener performs a slow network call, every other listener has to wait.

Similarly, if one listener throws an exception, the remaining listeners are never invoked.

These limitations don't make our implementation wrong. They simply reflect the next stage in its evolution.

In the next article, we'll address these problems by evolving our EventBus into a production-ready implementation capable of asynchronous execution, exception aggregation, listener priorities, and resilient event delivery.
