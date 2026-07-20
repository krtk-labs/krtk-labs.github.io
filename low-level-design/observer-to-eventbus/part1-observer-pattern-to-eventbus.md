# Evolving the Observer Pattern: From Simple Callbacks to an Event Bus

When we learn the Observer Pattern, we're usually presented with the final design—a publisher, a list of observers, an event bus, maybe even generics. It all looks neat, but it also raises an obvious question:

> **Why would anyone build all of this for something as simple as sending an email after an order is placed?**

The answer is that they probably wouldn't.

Good software rarely starts with sophisticated abstractions. It starts with the simplest solution that solves today's problem. The interesting part is not learning the final design, but understanding **how and why it evolves**.

Let's walk through that evolution.

---

## Step 1: The Simplest Solution

Imagine you're building an e-commerce application. The first requirement is straightforward.

> When an order is placed, send a confirmation email.

Most developers would write something like this.

```java
public class OrderService {

    private final EmailService emailService = new EmailService();

    public void placeOrder(Order order) {

        save(order);

        emailService.sendConfirmation(order);
    }
}
```

There is nothing wrong with this implementation.

In fact, introducing the Observer Pattern here would make the code worse. You would be adding extra classes and indirection without solving any real problem.

A useful question to ask yourself is:

> **If I removed the abstraction, would the code become simpler?**

At this stage, the answer is yes.

---

## Step 2: Requirements Grow

Applications rarely stay this simple.

Soon, the business asks for more.

* Record analytics
* Reserve inventory
* Award loyalty points
* Notify the warehouse

Your service slowly starts growing.

```java
public void placeOrder(Order order) {

    save(order);

    emailService.sendConfirmation(order);

    analyticsService.record(order);

    warehouseService.reserve(order);

    rewardService.addPoints(order);
}
```

The code still works, but something feels different.

Originally, `OrderService` was responsible for creating orders.

Now it is responsible for emails, analytics, inventory, rewards, and whatever the next sprint brings.

The business logic hasn't become more complicated.

The follow-up actions have.

This is usually the first sign that a design is ready to evolve.

---

## Step 3: Introducing the Observer Pattern

Instead of asking the `OrderService` to perform every follow-up action, we can ask it to simply announce that an order has been placed.

Anyone interested can react to that notification.

We start with a simple interface.

```java
public interface OrderListener {

    void onOrderPlaced(Order order);

}
```

Each responsibility becomes its own implementation.

```java
public class EmailListener implements OrderListener {

    @Override
    public void onOrderPlaced(Order order) {

        emailService.sendConfirmation(order);

    }
}
```

```java
public class AnalyticsListener implements OrderListener {

    @Override
    public void onOrderPlaced(Order order) {

        analyticsService.record(order);

    }
}
```

`OrderService` no longer depends on individual services.

Instead, it maintains a list of listeners.

```java
private final List<OrderListener> listeners = new ArrayList<>();
```

When an order is placed, it simply notifies them.

```java
for (OrderListener listener : listeners) {

    listener.onOrderPlaced(order);

}
```

Notice what changed.

`OrderService` no longer knows who is listening.

It only knows that somebody might be interested.

This is the classical Observer Pattern.

---

## Is This the Final Design?

At this point, many tutorials immediately introduce an `EventBus`.

I don't think that's the right next step.

Instead, let's continue building the application.

Suppose we now introduce payments.

We create a new service.

```java
public class PaymentService {

    ...
}
```

It also needs to notify interested parties.

So naturally, we create another listener interface.

```java
public interface PaymentListener {

    void onPaymentCompleted(Payment payment);

}
```

Now compare the two interfaces.

```java
interface OrderListener {

    void onOrderPlaced(Order order);

}
```

```java
interface PaymentListener {

    void onPaymentCompleted(Payment payment);

}
```

Different names.

Different parameter types.

The same idea.

We've discovered our next duplication.

---

## Step 4: Generalising the Listener

Instead of creating a new interface for every type of event, we can define a generic contract.

```java
public interface EventListener<T> {

    void onEvent(T event);

}
```

Now our listeners become

```java
class EmailListener
        implements EventListener<OrderPlacedEvent>
```

and

```java
class FraudDetectionListener
        implements EventListener<PaymentCompletedEvent>
```

An interesting observation is that this doesn't reduce the number of listener implementations.

You'll still have one listener for email, one for analytics, one for fraud detection, and so on.

So what did we gain?

We removed duplicated interfaces.

More importantly, we now have a single abstraction that every event in the application can use.

That becomes valuable when we start building reusable infrastructure.

---

## Step 5: Another Pattern Appears

Let's compare our services.

Both `OrderService` and `PaymentService` now have

* a list of listeners
* a `register()` method
* an `unregister()` method
* a loop that notifies listeners

The business logic is different.

The listener management is identical.

Why should every service know how to register listeners?

Why should every service know how to notify them?

Those responsibilities have nothing to do with orders or payments.

They belong somewhere else.

---

## Step 6: Extracting an EventBus

This is where an `EventBus` naturally appears.

Instead of every service managing its own listeners, one shared component manages them for the whole application.

The services become remarkably small.

```java
public class OrderService {

    private final EventBus eventBus;

    public void placeOrder(Order order) {

        save(order);

        eventBus.publish(new OrderPlacedEvent(order));

    }
}
```

The `EventBus` becomes responsible for

* registering listeners
* unregistering listeners
* finding listeners for an event
* publishing events

Each service only publishes facts about what happened.

The infrastructure decides who receives those facts.

---

## What Actually Evolved?

Notice that every step in this journey was motivated by a concrete problem.

We didn't start by designing an EventBus.

We discovered it.

| Stage                    | Problem                                          | Solution                   |
| ------------------------ | ------------------------------------------------ | -------------------------- |
| Direct method calls      | Only one follow-up action                        | Keep it simple             |
| Growing responsibilities | Too many follow-up actions inside `OrderService` | Observer Pattern           |
| Multiple domains         | Duplicate listener interfaces                    | Generic `EventListener<T>` |
| Multiple publishers      | Duplicate listener management                    | EventBus                   |

Every abstraction solved an existing problem.

None of them were introduced because they "might be useful someday."

---

## Final Thoughts

One of the biggest mistakes we make while learning design patterns is treating them as starting points.

They're not.

They're destinations.

The Observer Pattern isn't about publishers and subscribers.

It's about removing responsibilities from a class that has slowly accumulated too many of them.

The generic listener isn't about fancy type systems.

It's about removing duplicated contracts.

The EventBus isn't a more advanced Observer Pattern.

It's simply the next refactoring after you've noticed that every service is managing listeners in exactly the same way.

When you approach patterns this way, they stop feeling like academic concepts and start feeling like what they really are:

**Solutions that emerge naturally once your code has grown enough to need them.**
