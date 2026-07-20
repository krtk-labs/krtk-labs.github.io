# Understanding Java Lambda Target Types: Why `return facts -> true` Works

One of the most common misconceptions about Java lambdas is that they are somehow "objects" by themselves.

They're not.

In fact, a lambda expression **doesn't have a type of its own**.

This becomes obvious when you encounter code like this:

```java
default Condition and(Condition other) {
    return facts -> this.evaluate(facts) && other.evaluate(facts);
}
```

If you've only seen lambdas passed as method parameters, this can be confusing.

Where did this lambda come from?

How does Java know that it should become a `Condition`?

The answer lies in one of the most important concepts in Java's lambda implementation:

> **Target Typing**

---

## A Lambda Has No Type

Consider the following line:

```java
facts -> true;
```

Will this compile?

No.

The compiler complains because it has no idea what this lambda is supposed to represent.

Should it become:

- a `Condition`?
- a `Predicate<Facts>`?
- a `Function<Facts, Boolean>`?
- something else entirely?

A lambda expression is simply a piece of executable code.

Until Java knows **what interface it is implementing**, the lambda has no meaning.

---

## The Compiler Needs a Target Type

A lambda can only exist when Java can determine its **target type**.

A target type is simply the type that the compiler expects the expression to produce.

Let's look at the most common examples.

### 1. Method Parameters

Most developers first encounter lambdas like this:

```java
void process(Condition condition) {
    ...
}

process(facts -> true);
```

How does the compiler understand this lambda?

Because the parameter type is already known.

```text
process( ________ )

Parameter type = Condition
```

The compiler rewrites this conceptually as:

```java
Condition condition = facts -> true;
process(condition);
```

The parameter provides the target type.

---

### 2. Variable Assignment

The same thing happens during assignment.

```java
Condition condition = facts -> true;
```

Here the compiler sees:

```text
Condition  =  lambda
```

Since the left-hand side is a `Condition`, the lambda is converted into a `Condition`.

Again, the target type comes from the assignment.

---

### 3. Return Statements

Now let's return to our original example.

```java
default Condition and(Condition other) {
    return facts -> this.evaluate(facts)
                  && other.evaluate(facts);
}
```

At first glance, it almost looks magical.

Where did the `Condition` come from?

The compiler performs the exact same reasoning.

```text
return __________
```

It checks the method signature.

```java
default Condition and(...)
```

The method promises to return a `Condition`.

Therefore, the expression following `return` must also be a `Condition`.

The compiler interprets the lambda as:

```java
Condition result =
    facts -> this.evaluate(facts)
              && other.evaluate(facts);

return result;
```

Nothing magical is happening.

The **return type** is simply acting as the target type.

---

## Lambdas Always Need Context

One way to think about lambdas is this:

> A lambda never decides what it is.

The surrounding code decides.

Consider the following situations.

### Variable Assignment

```java
Condition c = facts -> true;
```

Target type:

```
Condition
```

---

### Method Parameter

```java
process(facts -> true);
```

Target type:

```
Condition
```

---

### Return Statement

```java
return facts -> true;
```

Target type:

```
Condition
```

---

### Explicit Cast

```java
Object obj = (Condition) (facts -> true);
```

Target type:

```
Condition
```

In every example, **the surrounding context tells Java which functional interface to create.**

---

## Which Method Does the Lambda Implement?

Now another question naturally arises.

Your interface looks like this:

```java
@FunctionalInterface
public interface Condition {

    boolean evaluate(Facts facts);

    default Condition and(Condition other) { ... }

    default Condition or(Condition other) { ... }

    default Condition negate() { ... }
}
```

There are four methods.

Which one does the lambda implement?

The answer is:

**Only the single abstract method.**

Java completely ignores the default methods during lambda conversion.

Since `Condition` has exactly one abstract method:

```java
boolean evaluate(Facts facts);
```

the compiler understands that

```java
facts -> true
```

is shorthand for

```java
new Condition() {

    @Override
    public boolean evaluate(Facts facts) {
        return true;
    }

};
```

Notice something important.

The compiler generates **only** the implementation of `evaluate()`.

It does **not** generate implementations for:

- `and()`
- `or()`
- `negate()`

Those methods are already implemented by the interface itself.

---

## Why Can I Immediately Call `and()`?

Suppose we write

```java
Condition adult =
    facts -> facts.get("age", Integer.class) >= 18;
```

Immediately afterwards we can write

```java
Condition eligible =
    adult.and(isEmployee);
```

But the lambda never implemented `and()`.

Why does this work?

Because after conversion, the lambda **is now a `Condition` object**.

Every `Condition` automatically inherits:

- `evaluate()`
- `and()`
- `or()`
- `negate()`

Exactly like a class inherits methods from its superclass.

The lambda only provides the missing piece:

```text
evaluate()
```

Everything else already exists.

---

## A Helpful Mental Model

Imagine every lambda being converted into an anonymous class.

This:

```java
facts -> facts.isValid()
```

is conceptually equivalent to:

```java
new Condition() {

    @Override
    public boolean evaluate(Facts facts) {
        return facts.isValid();
    }

}
```

The anonymous class automatically inherits the default methods from the interface.

The lambda simply provides a much shorter syntax.

---

## The Bigger Picture

Target typing is not limited to parameters.

Java uses it anywhere it can determine the expected type of an expression.

This includes:

- Variable assignments
- Method arguments
- Return statements
- Explicit casts
- Conditional (`?:`) expressions
- Generic method inference

Once you understand target typing, many pieces of Java's lambda syntax suddenly make sense.

Instead of asking:

> "How did Java know this lambda was a `Condition`?"

start asking:

> **"What type is Java expecting here?"**

That expected type is the target type, and it is the key that allows Java to convert a lambda expression into an implementation of a functional interface.
