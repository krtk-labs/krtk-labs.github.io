# Interval Problems - Constructing Overlap Logic

## Step 1: Never start by thinking about overlap.

Instead ask:

> **"When are two intervals definitely separate?"**

This is much easier to visualize.

```
Current
[---------]

                 [---------]
                 Next
```

The current interval ends before the next interval starts.

Mathematically:

```java
current.end < next.start
```

This is the **base condition**.

---

# Step 2: Read the problem statement carefully

Every interval problem differs only in one aspect:

> **Does touching count as overlapping?**

## Case 1: Touching is NOT overlapping

Example:

```
[1,3]
     [3,5]
```

These can coexist.

Examples:
- Maximum Meetings
- Activity Selection
- Non-overlapping Intervals (Minimum Removals)

So the intervals are separate when:

```java
current.end <= next.start
```

Equivalent form:

```java
current.start >= previous.end
```

---

## Case 2: Touching IS overlapping

Example:

```
[1,3]
     [3,5]
```

These should merge into

```
[1,5]
```

Examples:
- Merge Intervals
- Insert Interval

Now the intervals are separate only when:

```java
current.end < next.start
```

Equivalent form:

```java
current.start > previous.end
```

---

# Step 3: Overlap is simply the opposite

Never memorize overlap directly.

Compute it as:

```
overlap = NOT(non-overlap)
```

### Meetings

Non-overlap

```java
current.start >= previous.end
```

Overlap

```java
current.start < previous.end
```

---

### Merge Intervals

Non-overlap

```java
current.start > previous.end
```

Overlap

```java
current.start <= previous.end
```

---

# The Construction Process

Whenever solving an interval problem:

### Question 1

Are the intervals sorted?

- Yes → Single linear scan
- No → Sort first

---

### Question 2

When are two intervals completely separate?

Draw them.

```
Current
[------]

            [------]
            Next
```

Translate into code.

---

### Question 3

Does touching count as overlap?

If the problem says:

```
[1,3] and [3,4]
```

are **non-overlapping**

Use:

```java
current.end <= next.start
```

If the problem merges them

Use:

```java
current.end < next.start
```

---

### Question 4

Everything else is overlap.

No separate formula to memorize.

---

# Quick Cheat Sheet

| Problem | Touching Allowed? | Non-overlap Condition | Overlap Condition |
|----------|-------------------|-----------------------|-------------------|
| Maximum Meetings | ✅ Yes | `current.start >= prev.end` | `current.start < prev.end` |
| Activity Selection | ✅ Yes | `current.start >= prev.end` | `current.start < prev.end` |
| Non-overlapping Intervals | ✅ Yes | `current.start >= prev.end` | `current.start < prev.end` |
| Merge Intervals | ❌ No | `current.start > prev.end` | `current.start <= prev.end` |
| Insert Interval | ❌ No | `current.start > prev.end` | `current.start <= prev.end` |

---

# Mental Model

❌ Don't ask:

> "How do I detect overlap?"

✅ Ask:

> "How do I know these intervals are definitely separate?"

Once you have the **non-overlap** condition, the overlap condition is simply its negation.

---

# Golden Rule

> **Think in terms of separation, not overlap.**

The separation condition is intuitive to visualize.

The overlap condition is merely:

```text
NOT(separated)
```

This avoids memorizing four different comparison operators and instead derives the correct condition from the problem statement every time.
