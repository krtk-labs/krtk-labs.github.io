# Beyond `if-else`: Designing Composable Business Rules with the Specification Pattern

> *"The hardest part of building a rules engine isn't evaluating rules. It's preventing the rules themselves from becoming the engine."*

As engineers, we've all seen code like this.

```java
if (marketOpen
    && !exchangeHoliday
    && availableMargin >= requiredMargin
    && impliedVolatility >= 18
    && daysToExpiry <= 5
    && !strategyAlreadyRunning
    && riskChecksPassed) {

    deployStrategy();
}
```

It starts innocently enough.

Then a new requirement arrives.

> *"Deploy if IV is above 18, unless it's expiry day. Except for Iron Condors, where expiry day is allowed. But don't deploy if the portfolio delta exceeds the configured threshold, unless the strategy is marked as high priority."*

The single `if` statement becomes a hundred-line boolean expression. Eventually someone extracts pieces into helper methods.

```java
if (isMarketEligible()
        && hasEnoughMargin()
        && passesRiskChecks()
        && isStrategyEligible()) {
    ...
}
```

It looks cleaner, but has the architecture actually improved?

Not really.

The business rules are still tightly coupled together. They can't be reused. They can't be composed differently for another strategy. Every new rule requires modifying existing code.

The problem isn't the `if`.

The problem is that **the business rules are not first-class objects**.

---

# Thinking Differently

Suppose instead of asking

> "Should I deploy this strategy?"

we ask

> "What are the individual rules that determine deployment?"

Immediately, the problem changes.

Instead of one giant decision, we have many independent decisions.

- Is the market open?
- Is there enough margin?
- Is implied volatility above the threshold?
- Are risk checks passing?
- Is the strategy already running?

Each one has exactly one responsibility.

Each one answers a simple question.

```text
Yes

or

No
```

Now imagine if every one of these questions were represented as an object.

Not a boolean.

An object.

---

# Enter the Specification Pattern

The Specification Pattern models a business rule as an object.

Instead of writing

```java
boolean eligible = ...
```

we create

```java
Specification<StrategyDeploymentContext>
```

whose only responsibility is answering

```java
boolean isSatisfiedBy(T candidate)
```

Conceptually,

```text
                  Strategy
                      │
                      │
          "Can I deploy this?"
                      │
                      ▼
          Specification evaluates
                      │
             true / false
```

Each specification represents exactly one business concept.

---

# A Small Interface with Big Consequences

```java
@FunctionalInterface
public interface Specification<T> {

    boolean isSatisfiedBy(T candidate);

    default Specification<T> and(Specification<T> other) {
        return candidate ->
                this.isSatisfiedBy(candidate)
                        && other.isSatisfiedBy(candidate);
    }

    default Specification<T> or(Specification<T> other) {
        return candidate ->
                this.isSatisfiedBy(candidate)
                        || other.isSatisfiedBy(candidate);
    }

    default Specification<T> negate() {
        return candidate ->
                !this.isSatisfiedBy(candidate);
    }
}
```

At first glance, nothing seems remarkable.

Until you notice something subtle.

```text
Specification

AND

Specification

↓

Specification
```

The result of combining two specifications is... another specification.

This property is called **closure**.

The API is *closed over its own type*.

That single design decision makes unlimited composition possible.

---

# Simple and Composite Become the Same Thing

Imagine these two rules.

```text
Market Open

Margin Available
```

Both are specifications.

Now combine them.

```text
Market Open

AND

Margin Available
```

What is the result?

Another specification.

To the caller, there is no distinction between a simple specification and one built from twenty smaller specifications.

```text
                 Specification
                  /        \
                 /          \
        Simple Rule     Composite Rule
```

This is where the **Composite Pattern** enters the picture.

The caller treats both objects exactly the same.

---

# A Real Trading Example

Let's model strategy deployment.

Instead of a bag of booleans, create a context object.

```java
public record StrategyDeploymentContext(

        boolean marketOpen,
        boolean exchangeHoliday,

        double impliedVolatility,

        double availableMargin,
        double requiredMargin,

        int daysToExpiry,

        boolean alreadyRunning,
        boolean riskChecksPassed,

        double portfolioDelta,
        double maxAllowedDelta

) {}
```

This object contains everything a deployment rule might need.

Notice something important.

It contains **data**.

It contains **no business logic**.

---

# Building Specifications

Rather than creating dozens of tiny classes, we'll use static factory methods that produce specifications.

```java
public final class DeploymentSpecifications {

    private DeploymentSpecifications() {}

    public static Specification<StrategyDeploymentContext> marketOpen() {
        return StrategyDeploymentContext::marketOpen;
    }

    public static Specification<StrategyDeploymentContext> notExchangeHoliday() {
        return ctx -> !ctx.exchangeHoliday();
    }

    public static Specification<StrategyDeploymentContext> marginAvailable() {
        return ctx ->
                ctx.availableMargin()
                        >= ctx.requiredMargin();
    }

    public static Specification<StrategyDeploymentContext> ivAbove(double minimumIV) {
        return ctx ->
                ctx.impliedVolatility() >= minimumIV;
    }

    public static Specification<StrategyDeploymentContext> expiryWithin(int days) {
        return ctx ->
                ctx.daysToExpiry() <= days;
    }

    public static Specification<StrategyDeploymentContext> riskChecksPassed() {
        return StrategyDeploymentContext::riskChecksPassed;
    }

    public static Specification<StrategyDeploymentContext> notAlreadyRunning() {
        return ctx -> !ctx.alreadyRunning();
    }

    public static Specification<StrategyDeploymentContext> portfolioDeltaWithinLimits() {
        return ctx ->
                Math.abs(ctx.portfolioDelta())
                        <= ctx.maxAllowedDelta();
    }
}
```

Each method creates one reusable business rule.

Nothing more.

Nothing less.

---

# The Interesting Part

Now imagine the business requirement arrives.

> Deploy the strategy only when

- the market is open
- today is not an exchange holiday
- IV is above 18
- enough margin exists
- expiry is within five days
- portfolio risk checks pass
- strategy isn't already deployed
- portfolio delta remains within limits

Most systems immediately translate this into nested `if` statements.

Instead, we translate it directly into code.

```java
Specification<StrategyDeploymentContext> deploymentRule =

        DeploymentSpecifications.marketOpen()

        .and(DeploymentSpecifications.notExchangeHoliday())

        .and(DeploymentSpecifications.ivAbove(18))

        .and(DeploymentSpecifications.marginAvailable())

        .and(DeploymentSpecifications.expiryWithin(5))

        .and(DeploymentSpecifications.riskChecksPassed())

        .and(DeploymentSpecifications.notAlreadyRunning())

        .and(DeploymentSpecifications.portfolioDeltaWithinLimits());
```

Read that again.

The Java code reads almost exactly like the business requirement.

There are no temporary booleans.

No helper methods.

No nested branching.

Just a fluent composition of business rules.

---

# The Engine Doesn't Know the Rules

The deployment engine becomes almost embarrassingly simple.

```java
public class StrategyDeploymentEngine {

    private final Specification<StrategyDeploymentContext> deploymentRule;

    public StrategyDeploymentEngine(
            Specification<StrategyDeploymentContext> deploymentRule) {

        this.deploymentRule = deploymentRule;
    }

    public boolean shouldDeploy(
            StrategyDeploymentContext context) {

        return deploymentRule.isSatisfiedBy(context);
    }
}
```

Notice what happened.

The engine no longer knows

- what IV means,
- how margin works,
- what constitutes a holiday,
- or how risk checks are performed.

It knows only one abstraction.

```text
Specification
```

Everything else has been delegated.

---

# Changing Business Rules

Six months later the business changes.

> Deploy if IV exceeds 18 **or** portfolio delta exceeds 0.60.

Most systems now modify the deployment engine.

With specifications, we simply compose a different rule.

```java
Specification<StrategyDeploymentContext> volatilityTrigger =

        DeploymentSpecifications.ivAbove(18)

                .or(ctx ->
                        Math.abs(ctx.portfolioDelta()) > 0.60);
```

Then plug it back into the larger rule.

The engine remains untouched.

---

# Why Static Factory Methods?

Some implementations create one class per specification.

```text
MarginAvailableSpecification

IVAboveSpecification

MarketOpenSpecification

...
```

That works.

But for small, stateless predicates, dozens of tiny classes add ceremony without adding value.

Instead, expose specifications through factory methods.

```java
DeploymentSpecifications.ivAbove(18)
```

rather than

```java
new IVAboveSpecification(18)
```

If one specification eventually grows into fifty lines of logic or requires injected collaborators, it can internally return a dedicated implementation class later.

The public API never changes.

---

# Why This Pattern Scales

As systems grow, the challenge isn't writing business rules.

It's managing change.

The Specification Pattern gives us:

- Small, independently testable business rules.
- Reusable rules across workflows.
- Declarative composition instead of imperative branching.
- Stable engines that don't change when business policy changes.
- A fluent API that mirrors the language of the business.

Most importantly, it changes how we think.

Instead of asking

> *"How do I write this `if` statement?"*

we start asking

> *"What business concepts exist, and how can they be composed?"*

That's a much more scalable question.

---

# Final Thoughts

The Specification Pattern is often introduced as another Gang of Four pattern.

In practice, it's much more than that.

Combined with the Composite Pattern, it allows us to model business policy as a graph of reusable objects rather than a collection of branching statements.

The result is code that evolves with the business instead of fighting it.

And once you start seeing specifications, you'll notice them everywhere.

- `Predicate.and()`
- `Predicate.or()`
- Spring Security authorization managers
- JPA Criteria predicates
- Query builders
- Rules engines

They all share the same elegant idea.

**Simple rules and complex rules should look exactly the same to the code that consumes them.**
