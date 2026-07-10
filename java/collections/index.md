---
layout: default
title: Collections
parent: Java
nav_order: 1
has_children: true
permalink: /java/collections/
---

# How Java Collections Really Work

If you have written any non-trivial Java, you have already formed an intuition about collections. You reach for `ArrayList` when you need a list, `HashMap` when you need a lookup table, and `ConcurrentHashMap` when someone mentions threads. The code compiles, the tests pass, and life goes on. But that intuition is usually shallow — it tells you *which* class to import, not *why* that class exists or what you gave up by choosing it over one of the dozen alternatives sitting right next to it in the same package.

This article tears open the Java Collections Framework and explains what is actually going on underneath the interface names — why so many implementations exist for what looks like the same job, why two thread-safe maps can behave completely differently under load, and what questions you should be asking before you type `new`.

## The Core Abstraction

The Java Collections Framework is not, at its heart, a pile of forty-odd classes. It is a small number of interfaces, each describing a contract, with implementations layered underneath that satisfy the contract in different ways. Once you see the interfaces clearly, the classes stop looking arbitrary.

```text
                 Iterable
                     │
                Collection
                     │
     ┌──────────┬──────────┬──────────┐
     │          │          │
    List       Set       Queue
                            │
                          Deque
```

`Map` sits outside this hierarchy entirely — it does not extend `Collection` — because a map is fundamentally a different kind of object: a set of key-value associations rather than a bag of elements. This is one of those framework decisions that looks inconsistent until you remember it was made this way on purpose. A `Map` cannot be iterated the same way a `List` can, so forcing it into the `Collection` interface would have meant lying about what it actually does.

Every concrete class you have ever used — `ArrayList`, `HashMap`, `LinkedBlockingQueue`, `TreeSet` — is an answer to the same underlying question: given this interface's contract, how do we implement it to optimize for a particular access pattern, ordering guarantee, or concurrency model?

## What the Interfaces Actually Promise

A `List` promises order and duplicates. Elements stay where you put them, indexed from zero, and nothing stops you from inserting the same value twice. `ArrayList` backs this with a resizable array, which makes index-based reads fast and shifts elements around on insertion or removal in the middle. `LinkedList` backs the same contract with a doubly linked list, trading fast indexed access for cheap insertion at either end. Same promise, different machinery, different trade-offs.

A `Set` promises uniqueness and nothing else by default. `HashSet` gives you no ordering guarantee at all in exchange for near-constant-time lookups. `LinkedHashSet` adds back insertion-order iteration at a small memory cost. `TreeSet` goes further and keeps elements sorted at all times, which costs you logarithmic-time operations instead of constant-time ones — the price of maintaining an ordering invariant on every insert.

A `Queue` promises a processing order, usually first-in-first-out, and exists specifically for producer-consumer patterns: task scheduling, event processing, message passing between threads. A `Deque` relaxes the "one end only" restriction and lets you push and pop from both ends, which is what makes it useful as a stack replacement, a sliding-window buffer, or the backbone of breadth-first and depth-first traversals.

A `Map` promises unique keys mapped to values, and almost every other collection in the framework can be understood as a specialization of this idea. A `HashSet`, for instance, is implemented internally as a `HashMap` where the values are ignored.

None of this is exotic. What is easy to miss is that the interface only tells you *what* operations exist — it says nothing about *how expensive* those operations are, whether the collection is safe under concurrent access, or whether iteration order is something you can rely on. That information lives one level down, in the implementation, and it is the level most developers stop paying attention to.

## The Four Questions That Actually Matter

Instead of memorizing which of the dozen `Map` implementations to use, it is more useful to evaluate any collection along four independent dimensions. Once you internalize these, most implementation choices become almost mechanical.

| Dimension | Question It Answers | Typical Values |
|---|---|---|
| **Ordering** | Does the collection guarantee iteration order? | Ordered, Sorted, Unordered |
| **Thread Safety** | Can multiple threads safely access it? | Thread-Safe, Not Thread-Safe |
| **Synchronization Strategy** | How is thread safety implemented? | None, Lock-Based, Lock-Free (CAS), Copy-On-Write |
| **Blocking Behavior** | Can operations wait for work? | Blocking, Non-Blocking |

These four dimensions explain nearly every implementation you will ever encounter. `HashMap` sacrifices ordering entirely because giving it up is what makes constant-time lookups possible. `TreeMap` accepts logarithmic-time operations because it refuses to give up sorted order. `ConcurrentHashMap` is thread-safe without being blocking, using fine-grained, mostly lock-free synchronization so that reads rarely contend with writes. `ConcurrentSkipListMap` goes a step further and combines thread safety with sorted navigation, which is a harder problem than it sounds — [I have written separately about how it pulls this off](./why-concurrent-skip-map.md). `LinkedBlockingQueue` adds a fifth capability on top of thread safety: the ability for a consumer thread to simply wait until a producer has something for it, instead of spinning or polling.

To make this concrete, here is how a handful of commonly used collections score across these four dimensions:

| Collection | Ordered | Thread-Safe | Synchronization | Blocking |
|---|---|---|---|---|
| `ArrayList` | ✅ | ❌ | None | ❌ |
| `HashMap` | ❌ | ❌ | None | ❌ |
| `LinkedHashMap` | ✅ | ❌ | None | ❌ |
| `TreeMap` | ✅ (Sorted) | ❌ | None | ❌ |
| `ConcurrentHashMap` | ❌ | ✅ | Mostly Lock-Free | ❌ |
| `ConcurrentSkipListMap` | ✅ (Sorted) | ✅ | Lock-Free | ❌ |
| `ArrayDeque` | ✅ | ❌ | None | ❌ |
| `ConcurrentLinkedQueue` | FIFO | ✅ | Lock-Free | ❌ |
| `LinkedBlockingQueue` | FIFO | ✅ | Lock-Based | ✅ |
| `CopyOnWriteArrayList` | ✅ | ✅ | Copy-On-Write | ❌ |

Read this table as a set of trade-offs, not a leaderboard. Nothing in this list is strictly better than anything else — `ConcurrentSkipListMap` is not a "better" `HashMap`, it is a different answer to a different question.

## Choosing a Collection in Practice

When you are actually designing a system and need to pick a collection, the four dimensions translate into four questions worth asking in order, because each answer narrows the field considerably.

Start with how the data needs to be organized. If you don't care about order at all, you have opened the door to the fastest possible implementations. If you need insertion order preserved, you have ruled out plain hash-based structures. If you need the data sorted at all times, you have committed to logarithmic-time operations whether you wanted to or not.

Next, think about who touches the collection. A collection that only one thread ever sees does not need any synchronization at all, and adding it anyway is pure overhead. A collection with many readers and occasional writers is a very different problem from one with many concurrent writers, and the two call for different implementations — this is exactly the distinction that makes `CopyOnWriteArrayList` sensible for the first case and actively harmful for the second.

If you do need concurrency, decide how much contention you can tolerate. Lock-based synchronization is simple to reason about but serializes access under load. Lock-free structures built on compare-and-swap avoid that serialization but are harder to build correctly and are not automatically faster in every scenario. Copy-on-write structures make writes expensive so that reads can be essentially free, which only makes sense when reads vastly outnumber writes.

Finally, ask whether your consumers should ever wait. A non-blocking queue returns immediately, empty or not, and leaves the waiting logic to you. A blocking queue lets a consumer thread simply park until a producer supplies something, which is precisely what makes it the natural backbone of producer-consumer pipelines.

Answering these four questions in order usually takes you from "there are forty implementations" to "there are one or two reasonable choices" — and at that point, the decision is easy.

## Why This Framing Matters More Than It Seems

It is tempting to treat this as a purely academic exercise, but the four-dimension model has direct consequences on production systems that are easy to learn the hard way.

- Reaching for `HashMap` in a multi-threaded context without external synchronization does not fail loudly — it corrupts silently, producing infinite loops or lost entries under load that only show up once traffic increases.
- Choosing `CopyOnWriteArrayList` for a write-heavy workload does not throw an exception either — it just gets progressively slower as every write copies the entire backing array, and the degradation is easy to misattribute to something else entirely.
- Assuming `LinkedHashMap` and `HashMap` are interchangeable because they share most of the same API ignores that one of them is silently paying a small constant cost on every operation to maintain an ordering guarantee you may not even be using.

"It works in my local tests" and "it holds up under concurrent production load" are very different bars, and the gap between them is almost always explained by one of these four dimensions being ignored rather than deliberately chosen.

## A Common Misconception

A mistake that trips up even experienced developers: assuming that "thread-safe" is a single, uniform property that either a collection has or doesn't. It isn't. `Collections.synchronizedMap()`, `ConcurrentHashMap`, and `CopyOnWriteArrayList` are all thread-safe, and they achieve it through three completely different mechanisms with three completely different performance profiles under contention.

A synchronized wrapper puts a single lock around every operation — simple, correct, and a bottleneck the moment more than a couple of threads want in at once. `ConcurrentHashMap` partitions its internal state so that unrelated operations rarely block each other at all. `CopyOnWriteArrayList` does not use locks for reads in the traditional sense — it hands out a stable snapshot of the array and only pays a synchronization cost when someone writes. Calling all three "thread-safe" and stopping there erases exactly the information you need to pick correctly.

## What "Ordered" Actually Means

Similarly, when engineers say a collection is "ordered," they usually mean one of two very different things, and conflating them causes real bugs. Insertion order, as guaranteed by `LinkedHashMap` or `ArrayList`, means elements come back in the sequence you added them, and nothing more. Sort order, as guaranteed by `TreeMap` or `TreeSet`, means elements come back according to a comparator, regardless of when they were inserted. A `HashMap` guarantees neither, and the fact that it often *appears* to iterate in a stable order on a given JVM version is an implementation detail, not a contract — code that quietly depends on it is one JDK upgrade away from breaking.

## Footnote

The Java Collections Framework is a small set of interfaces — `List`, `Set`, `Queue`, `Deque`, and the interface-adjacent `Map` — layered with implementations that make different trade-offs across four dimensions: ordering, thread safety, synchronization strategy, and blocking behavior. Nearly every implementation decision in the framework is a direct consequence of choosing a point along these four axes.

Understanding this — rather than memorizing which class to import for which situation — is the foundation for reasoning about collections you have never used before, including the ones that get added in future JDK releases.

---

## Where to Go Next

This article is the starting point for a series that works through the framework layer by layer:

**Foundations**
- Java Collections Framework Overview *(this article)*
- Ordered vs Sorted vs Unordered Collections
- Choosing the Right Collection

**Core Collections**
- Lists · Sets · Maps · Queues · Deques

**Concurrent Collections**
- Thread-Safe Collections
- Lock-Based vs Lock-Free Collections
- Blocking vs Non-Blocking Collections

**Deep Dives**
- Inside `ConcurrentHashMap`
- [Inside `ConcurrentSkipListMap`](./why-concurrent-skip-map.md)
- Copy-On-Write Collections
- Blocking Queues
- ArrayDeque vs LinkedList

If you found this useful, share it with someone still importing `Vector` in 2026, and keep an eye out for the next piece in the series: **Ordered vs Sorted vs Unordered Collections**.
