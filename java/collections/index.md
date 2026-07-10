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

## The Five Questions That Actually Matter

Instead of memorizing which of the dozen `Map` implementations to use, it is more useful to evaluate any collection along five independent dimensions. Once you internalize these, most implementation choices become almost mechanical.

The first four are about coordination — whether the collection imposes any order and whether it can survive contact with more than one thread:

| Dimension | Question It Answers | Typical Values |
|---|---|---|
| **Ordering** | Does the collection guarantee iteration order? | Ordered, Sorted, Unordered |
| **Thread Safety** | Can multiple threads safely access it? | Thread-Safe, Not Thread-Safe |
| **Synchronization Strategy** | How is thread safety implemented? | None, Lock-Based, Lock-Free (CAS), Copy-On-Write |
| **Blocking Behavior** | Can operations wait for work? | Blocking, Non-Blocking |

But coordination is not the only thing a collection costs you. A fifth dimension sits underneath the other four and explains something they don't: raw operational speed.

| Dimension | Question It Answers | Typical Values |
|---|---|---|
| **Time Complexity** | How expensive is a get, put, insert, or remove? | O(1) average, O(log n), O(n) |

This is the dimension that explains why a collection with *no* coordination guarantees at all — no ordering, no thread safety, no locking — still earns a place in the framework. `HashMap` gives up every one of the first four properties on purpose, and what it buys with that trade is O(1) average-case lookups: no comparator to run, no tree to rebalance, no lock to acquire. An all-negative row in the earlier table isn't a collection with nothing to offer — it's the collection that opted out of every extra guarantee specifically to be the fastest plain key-value store available. That is also exactly when to reach for it: single-threaded code, no requirement on iteration order, and you just want the cheapest possible get and put.

Once you add time complexity into the picture, every other row in the table gets a clearer justification too. `TreeMap` accepts O(log n) operations because it refuses to give up sorted order — you are paying for the sort on every insert. `ConcurrentHashMap` is thread-safe without being blocking, using fine-grained, mostly lock-free synchronization so that reads rarely contend with writes, while staying close to `HashMap`'s O(1) average case. `ConcurrentSkipListMap` goes further still and combines thread safety with sorted navigation, which costs O(log n) rather than O(1) — a harder problem than it sounds, since it has to do all of this without ever taking a traditional lock — [I have written separately about how it pulls this off](./why-concurrent-skip-list-map.md). `LinkedBlockingQueue` adds a capability none of the others have: the ability for a consumer thread to simply wait until a producer has something for it, instead of spinning or polling, at the cost of that wait itself being unbounded time.

To make this concrete, here is how a handful of commonly used collections score across all five dimensions:

| Collection | Ordered | Thread-Safe | Synchronization | Blocking | Time Complexity (get/put) |
|---|---|---|---|---|---|
| `ArrayList` | ✅ | ❌ | None | ❌ | O(1) get, O(n) mid-insert |
| `HashMap` | ❌ | ❌ | None | ❌ | O(1) average |
| `LinkedHashMap` | ✅ | ❌ | None | ❌ | O(1) average |
| `TreeMap` | ✅ (Sorted) | ❌ | None | ❌ | O(log n) |
| `ConcurrentHashMap` | ❌ | ✅ | Mostly Lock-Free | ❌ | O(1) average |
| `ConcurrentSkipListMap` | ✅ (Sorted) | ✅ | Lock-Free | ❌ | O(log n) |
| `ArrayDeque` | ✅ | ❌ | None | ❌ | O(1) amortized, both ends |
| `ConcurrentLinkedQueue` | FIFO | ✅ | Lock-Free | ❌ | O(1) amortized |
| `LinkedBlockingQueue` | FIFO | ✅ | Lock-Based | ✅ | O(1) + wait time |
| `CopyOnWriteArrayList` | ✅ | ✅ | Copy-On-Write | ❌ | O(1) read, O(n) write |

Read this table as a set of trade-offs, not a leaderboard. Nothing in this list is strictly better than anything else — `ConcurrentSkipListMap` is not a "better" `HashMap`, it is a different answer to a different question, and it pays for that answer in both concurrency overhead and time complexity that plain `HashMap` never has to touch.

## Choosing a Collection in Practice

When you are actually designing a system and need to pick a collection, the five dimensions translate into five questions worth asking in order, because each answer narrows the field considerably.

Start with performance: what can this operation actually afford to cost? If you need the cheapest possible get and put and nothing else matters, you are already headed toward a hash-based structure. If you can tolerate O(log n) in exchange for a guarantee, sorted structures come into play. This question comes first because it sets a ceiling — no amount of ordering or concurrency reasoning matters if the base complexity of the structure was wrong for your workload to begin with.

Next, think about how the data needs to be organized. If you don't care about order at all, you have opened the door to the fastest possible implementations. If you need insertion order preserved, you have ruled out plain hash-based structures. If you need the data sorted at all times, you have committed to logarithmic-time operations whether you wanted to or not — this is where the performance question and the ordering question start pulling in opposite directions, and you have to decide which one wins.

After that, think about who touches the collection. A collection that only one thread ever sees does not need any synchronization at all, and adding it anyway is pure overhead. A collection with many readers and occasional writers is a very different problem from one with many concurrent writers, and the two call for different implementations — this is exactly the distinction that makes `CopyOnWriteArrayList` sensible for the first case and actively harmful for the second.

If you do need concurrency, decide how much contention you can tolerate. Lock-based synchronization is simple to reason about but serializes access under load. Lock-free structures built on compare-and-swap avoid that serialization but are harder to build correctly and are not automatically faster in every scenario. Copy-on-write structures make writes expensive so that reads can be essentially free, which only makes sense when reads vastly outnumber writes.

Finally, ask whether your consumers should ever wait. A non-blocking queue returns immediately, empty or not, and leaves the waiting logic to you. A blocking queue lets a consumer thread simply park until a producer supplies something, which is precisely what makes it the natural backbone of producer-consumer pipelines.

Answering these five questions in order usually takes you from "there are forty implementations" to "there are one or two reasonable choices" — and at that point, the decision is easy.

## Why This Framing Matters More Than It Seems

It is tempting to treat this as a purely academic exercise, but the four-dimension model has direct consequences on production systems that are easy to learn the hard way.

- Reaching for `HashMap` in a multi-threaded context without external synchronization does not fail loudly — it corrupts silently, producing infinite loops or lost entries under load that only show up once traffic increases.
- Choosing `CopyOnWriteArrayList` for a write-heavy workload does not throw an exception either — it just gets progressively slower as every write copies the entire backing array, and the degradation is easy to misattribute to something else entirely.
- Assuming `LinkedHashMap` and `HashMap` are interchangeable because they share most of the same API ignores that one of them is silently paying a small constant cost on every operation to maintain an ordering guarantee you may not even be using.

"It works in my local tests" and "it holds up under concurrent production load" are very different bars, and the gap between them is almost always explained by one of these five dimensions being ignored rather than deliberately chosen.

## A Common Misconception

A mistake that trips up even experienced developers: assuming that "thread-safe" is a single, uniform property that either a collection has or doesn't. It isn't. `Collections.synchronizedMap()`, `ConcurrentHashMap`, and `CopyOnWriteArrayList` are all thread-safe, and they achieve it through three completely different mechanisms with three completely different performance profiles under contention.

A synchronized wrapper puts a single lock around every operation — simple, correct, and a bottleneck the moment more than a couple of threads want in at once. `ConcurrentHashMap` partitions its internal state so that unrelated operations rarely block each other at all. `CopyOnWriteArrayList` does not use locks for reads in the traditional sense — it hands out a stable snapshot of the array and only pays a synchronization cost when someone writes. Calling all three "thread-safe" and stopping there erases exactly the information you need to pick correctly.

## What "Ordered" Actually Means

Similarly, when engineers say a collection is "ordered," they usually mean one of two very different things, and conflating them causes real bugs. Insertion order, as guaranteed by `LinkedHashMap` or `ArrayList`, means elements come back in the sequence you added them, and nothing more. Sort order, as guaranteed by `TreeMap` or `TreeSet`, means elements come back according to a comparator, regardless of when they were inserted. A `HashMap` guarantees neither, and the fact that it often *appears* to iterate in a stable order on a given JVM version is an implementation detail, not a contract — code that quietly depends on it is one JDK upgrade away from breaking.

## Footnote

The Java Collections Framework is a small set of interfaces — `List`, `Set`, `Queue`, `Deque`, and the interface-adjacent `Map` — layered with implementations that make different trade-offs across five dimensions: time complexity, ordering, thread safety, synchronization strategy, and blocking behavior. Nearly every implementation decision in the framework is a direct consequence of choosing a point along these five axes.

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
- [Inside `ConcurrentSkipListMap`](./why-concurrent-skip-list-map.md)
- Copy-On-Write Collections
- Blocking Queues
- ArrayDeque vs LinkedList

If you found this useful, share it with someone still importing `Vector` in 2026, and keep an eye out for the next piece in the series: **Ordered vs Sorted vs Unordered Collections**.
