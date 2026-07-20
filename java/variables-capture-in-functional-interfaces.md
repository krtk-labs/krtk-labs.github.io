# Lambdas Aren't Magic: Understanding Variable Capture in Java Functional Interfaces

One of the most common questions developers have when they first start using Java lambdas is:

> *"Where is this variable coming from?"*

Consider the following code:

```java
public static Action freezeAccount(AccountService accountService) {

    return (facts, result) -> {

        Transaction tx = facts.get("transaction", Transaction.class);

        accountService.freeze(tx.accountId());

        result.annotate("ACCOUNT", "Account frozen");
    };
}
```

There are two variables inside the lambda:

* `facts`
* `accountService`

The first one is easy to miss because it isn't a parameter of `freezeAccount()`. The second one is even more interesting because the method has already returned by the time the lambda executes.

Understanding why both work gives you a much deeper understanding of lambdas, anonymous classes, and Java's closure mechanism.

---

# First, remember what an Action actually is

Suppose our functional interface looks like this.

```java
@FunctionalInterface
public interface Action {
    void execute(Facts facts, ExecutionResult result);
}
```

A lambda is **not** a new language construct.

It is simply a shorter way of creating an implementation of a functional interface.

When Java sees

```java
(facts, result) -> {
    ...
}
```

it understands it as

> "Create an object whose `execute()` method contains this code."

Conceptually, the compiler rewrites it into something similar to:

```java
return new Action() {

    @Override
    public void execute(Facts facts, ExecutionResult result) {

        Transaction tx = facts.get("transaction", Transaction.class);

        accountService.freeze(tx.accountId());

        result.annotate("ACCOUNT", "Account frozen");
    }
};
```

Once you look at it this way, the mystery starts to disappear.

---

# Where does `facts` come from?

Notice that the factory method has only one parameter.

```java
freezeAccount(AccountService accountService)
```

Yet the lambda contains another parameter.

```java
(facts, result)
```

These belong to **two completely different methods**.

The first method is the factory.

```java
freezeAccount(...)
```

The second method is the implementation of

```java
Action.execute(...)
```

The timeline looks like this.

```
freezeAccount(service)

        │
        ▼

returns an Action object

        │
        │
        ▼

Later...

action.execute(facts, result)
```

The rule engine eventually calls

```java
action.execute(facts, result);
```

Those arguments become the lambda parameters.

So `facts` is **not captured**.

It is supplied later by whoever invokes the Action.

---

# Then why is `accountService` still available?

This is where Java closures come in.

When the lambda is created,

```java
freezeAccount(accountService)
```

the lambda remembers the value of `accountService`.

This remembered value is called a **captured variable**.

Think of it like this.

```
Method call

accountService
      │
      ▼

Lambda created

captures

      │
      ▼

stores AccountService reference
inside the generated object
```

Even after `freezeAccount()` returns, the lambda still owns its copy of the reference.

---

# What the compiler effectively generates

Although you never write it yourself, the compiler behaves roughly as if it created a class like this.

```java
final class FreezeAccountAction implements Action {

    private final AccountService accountService;

    FreezeAccountAction(AccountService accountService) {
        this.accountService = accountService;
    }

    @Override
    public void execute(Facts facts, ExecutionResult result) {

        Transaction tx = facts.get("transaction", Transaction.class);

        accountService.freeze(tx.accountId());

        result.annotate("ACCOUNT", "Account frozen");
    }
}
```

Then your factory simply becomes

```java
public static Action freezeAccount(AccountService accountService) {
    return new FreezeAccountAction(accountService);
}
```

This is conceptually what Java is doing behind the scenes.

---

# Anonymous classes behave exactly the same

Replace the lambda with an anonymous class.

```java
public static Action freezeAccount(AccountService accountService) {

    return new Action() {

        @Override
        public void execute(Facts facts, ExecutionResult result) {

            Transaction tx = facts.get("transaction", Transaction.class);

            accountService.freeze(tx.accountId());

            result.annotate("ACCOUNT", "Account frozen");
        }
    };
}
```

The behavior is identical.

The anonymous class also captures `accountService`.

The syntax is different.

The mechanism is the same.

---

# Why must captured variables be effectively final?

Consider this example.

```java
AccountService service = new AccountService();

Action action = freezeAccount(service);

service = new MockAccountService();
```

Now ask yourself:

Which service should the Action use?

The original one?

Or the new one?

To avoid ambiguity, Java copies the value of captured local variables into the generated object.

Because it copies the value, local variables must be **final** or **effectively final**.

This rule makes closures simple, predictable, and thread-safe.

---

# Captured variables vs method parameters

These are often confused.

| Variable         | Comes from                               |
| ---------------- | ---------------------------------------- |
| `accountService` | Captured when the lambda is created      |
| `facts`          | Passed later when `execute()` is invoked |
| `result`         | Passed later when `execute()` is invoked |

A good way to remember this is:

> Captured variables belong to the **creation phase**.

> Lambda parameters belong to the **execution phase**.

---

# A useful mental model

Every lambda has two sources of information.

```
Creation Time
--------------------------
Captured variables

accountService
threshold
discountPercentage


Execution Time
--------------------------
Method parameters

facts
result
customer
order
```

One is remembered forever.

The other arrives every time the lambda executes.

---

# This is exactly how your rule engine works

When you define a rule

```java
Rule.builder(...)
    .then(Actions.freezeAccount(accountService))
    .build();
```

nothing is executed.

The Action merely remembers its configuration.

Later, when the rule engine fires the rule,

```java
action.execute(facts, executionResult);
```

the runtime state (`facts` and `executionResult`) is supplied.

The Action combines:

* the configuration it captured earlier, and
* the runtime data supplied by the engine.

This separation between **configuration** and **execution** is one of the reasons functional interfaces work so well for rule engines.

---

# Thinking like a Principal Engineer

Whenever you see a lambda, don't think of it as anonymous code.

Think of it as a tiny object.

Ask yourself three questions:

1. **What interface is this implementing?**
2. **What variables does it capture?**
3. **Who will eventually invoke it?**

Answering those three questions makes almost every lambda easy to understand.

Once you adopt this mental model, lambdas stop feeling magical. They're simply concise objects that remember a little bit of state and expose a single method.
