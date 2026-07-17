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

A `Queue` promises a processing order — usually first-in-first-out, though not always. `PriorityQueue` is the exception worth flagging early: it only guarantees that the *head* is the smallest (or largest) element by some comparator, not that the whole collection is sorted. Iterating a `PriorityQueue` does not walk the elements in priority order — only repeatedly polling the head does. A `Deque` relaxes the "one end only" restriction and lets you push and pop from both ends, which is what makes it useful as a stack replacement, a sliding-window buffer, or the backbone of breadth-first and depth-first traversals.

A `Map` promises unique keys mapped to values, and almost every other collection in the framework can be understood as a specialization of this idea. A `HashSet`, for instance, is implemented internally as a `HashMap` where the values are ignored.

None of this is exotic. What is easy to miss is that the interface only tells you *what* operations exist — it says nothing about *how expensive* those operations are, whether the collection is safe under concurrent access, or whether iteration order is something you can rely on. That information lives one level down, in the implementation, and it is the level most developers stop paying attention to.

## The Six Dimensions That Actually Matter

Instead of memorizing which of the dozen `Map` implementations to use, it is more useful to evaluate any collection along six independent dimensions. Once you internalize these, most implementation choices become almost mechanical.

The first four are about coordination — whether the collection imposes any order and whether it can survive contact with more than one thread:

| Dimension | Question It Answers | Typical Values |
|---|---|---|
| **Ordering** | Does the collection guarantee iteration order? | Ordered, Sorted, Unordered |
| **Thread Safety** | Can multiple threads safely access it? | Thread-Safe, Not Thread-Safe |
| **Synchronization Strategy** | How is thread safety implemented? | None, Lock-Based, Lock-Free (CAS), Copy-On-Write |
| **Blocking Behavior** | Can operations wait for work? | Blocking, Non-Blocking |

But coordination is not the only thing a collection costs you. A fifth dimension sits underneath the other four and explains something they don't: raw operational speed.

Two words show up constantly in these numbers and are easy to conflate: **average** and **amortized**. Average-case complexity describes performance across a range of *inputs* — a `HashMap` get is O(1) on average because a good hash function spreads keys evenly across buckets, but a pathological set of colliding keys can degrade a single lookup toward O(n). Amortized complexity describes performance across a *sequence of operations* on the same structure — an `ArrayList` add is O(1) amortized because most calls just write into free array space at O(1), and the occasional expensive O(n) resize-and-copy is rare enough that, spread out over all the cheap calls since the last resize, its cost averages down to O(1) per call. Average is about the shape of the data; amortized is about an occasional expensive operation being smoothed out over many cheap ones. The two aren't mutually exclusive — a single operation can be described by either, or sometimes both, depending on which source of variability is actually at play.

| Dimension | Question It Answers | Typical Values |
|---|---|---|
| **Time Complexity** | How expensive is a get, put, insert, or remove? | O(1) average, O(log n), O(n) |

This is the dimension that explains why a collection with *no* coordination guarantees at all — no ordering, no thread safety, no locking — still earns a place in the framework. `HashMap` gives up every one of the first four properties on purpose, and what it buys with that trade is O(1) average-case lookups: no comparator to run, no tree to rebalance, no lock to acquire. An all-negative row in the earlier table isn't a collection with nothing to offer — it's the collection that opted out of every extra guarantee specifically to be the fastest plain key-value store available. That is also exactly when to reach for it: single-threaded code, no requirement on iteration order, and you just want the cheapest possible get and put.

Once you add time complexity into the picture, every other row in the table gets a clearer justification too. `TreeMap` accepts O(log n) operations because it refuses to give up sorted order — you are paying for the sort on every insert. `ConcurrentHashMap` is thread-safe without being blocking, using fine-grained, mostly lock-free synchronization so that reads rarely contend with writes, while staying close to `HashMap`'s O(1) average case. `ConcurrentSkipListMap` goes further still and combines thread safety with sorted navigation, which costs O(log n) rather than O(1) — a harder problem than it sounds, since it has to do all of this without ever taking a traditional lock — [I have written separately about how it pulls this off](./why-concurrent-skip-list-map.md). `LinkedBlockingQueue` adds a capability none of the others have: the ability for a consumer thread to simply wait until a producer has something for it, instead of spinning or polling, at the cost of that wait itself being unbounded time.

There is a sixth dimension, and it's the one most tutorials skip entirely, because it doesn't apply to `List`, `Set`, or `Map` at all — it only shows up once you look at queues. That's exactly why the collections most often left out of these comparisons are queue implementations.

| Dimension | Question It Answers | Typical Values |
|---|---|---|
| **Boundedness / Capacity** | What happens when the collection is full? | Unbounded, Bounded, Zero-capacity (rendezvous) |

Most `List`, `Set`, and `Map` implementations are unbounded by default — they grow until the JVM runs out of heap, and that's rarely the first thing that goes wrong in practice. Queues are different, because queues are usually the buffer between a fast producer and a slower consumer, and an unbounded queue in that position is a memory leak with a delay timer on it: nothing crashes until the producer outruns the consumer for long enough, and then everything crashes at once. `ArrayBlockingQueue` and `LinkedBlockingDeque` are bounded by construction — you supply a capacity, and once it's full, further inserts either block or fail, which is precisely the backpressure a producer-consumer pipeline needs to stay stable. `LinkedBlockingQueue` is only *optionally* bounded, and reaching for the unbounded constructor out of habit is a common way to accidentally remove that backpressure. `SynchronousQueue` takes this to its logical extreme: its capacity is zero. It holds nothing at all — every `put()` blocks until a consumer calls `take()` at that exact moment, making it less a queue and more a direct handoff point between two threads. `LinkedTransferQueue` sits at the other extreme, unbounded, but gives producers the option to wait until a consumer has actually received an element rather than just accepted it into a buffer.

Boundedness interacts with the coordination dimensions in a way that's easy to miss: two queues can be identically thread-safe and identically blocking, and still behave completely differently in production because one of them has a ceiling and the other doesn't.

With all six dimensions in hand, the next step is to see them applied to every implementation in the framework, not just a handful — that's what the following section does.

## Filling In the Framework

The framework has roughly thirty implementations across four families. Almost every one of them is explained by the six dimensions above; the few that aren't get a callout of their own in the next two sections.

**List implementations**

| Collection | Ordered | Thread-Safe | Synchronization | Bounded | Time Complexity |
|---|---|---|---|---|---|
| `ArrayList` | ✅ | ❌ | None | Unbounded | • `get(index)`: O(1)<br>• `add` at end: O(1) amortized<br>• `add`/`remove` in the middle: O(n) |
| `LinkedList` | ✅ | ❌ | None | Unbounded | • `addFirst`/`addLast`/`removeFirst`/`removeLast`: O(1)<br>• `get(index)`: O(n) |
| `Vector` | ✅ | ✅ | Lock-Based (legacy) — single lock shared by every method | Unbounded | • `get(index)`: O(1), lock acquired on every call<br>• `add`/`remove` in the middle: O(n) |
| `Stack` | ✅ | ✅ | Lock-Based (legacy) — inherited from `Vector`, single lock shared by every method | Unbounded | • `push`/`pop`/`peek`: O(1), lock acquired on every call |
| `CopyOnWriteArrayList` | ✅ | ✅ | Copy-On-Write | Unbounded | • `get` (read): O(1), no locking<br>• `add`/`remove`/`set` (write): O(n), full array copy on every write |

`Vector` and `Stack` score identically to `CopyOnWriteArrayList` on thread-safety, and yet nobody recommends them for a read-heavy concurrent workload — because the *synchronization strategy* column is the one that actually differentiates them. A single coarse lock around every method serializes all access, reads included; copy-on-write does not. Same "Thread-Safe: ✅," very different production behavior. (More on why `Vector` and `Stack` still exist at all in the legacy callout below.)

**Set implementations**

| Collection | Ordered | Thread-Safe | Synchronization | Bounded | Time Complexity |
|---|---|---|---|---|---|
| `HashSet` | ❌ | ❌ | None | Unbounded | • `add`/`remove`/`contains`: O(1) average |
| `LinkedHashSet` | ✅ | ❌ | None | Unbounded | • `add`/`remove`/`contains`: O(1) average |
| `TreeSet` | ✅ (Sorted) | ❌ | None | Unbounded | • `add`/`remove`/`contains`: O(log n)<br>• navigation (`floor`, `ceiling`, `first`, `last`): O(log n) |
| `ConcurrentSkipListSet` | ✅ (Sorted) | ✅ | Lock-Free — CAS on skip-list node pointers at each level (Fraser-style lock-free skip list) | Unbounded | • `add`/`remove`/`contains`: O(log n) expected<br>• navigation (`floor`, `ceiling`, `first`, `last`): O(log n) expected |
| `CopyOnWriteArraySet` | ✅ | ✅ | Copy-On-Write | Unbounded | • `contains`: O(n), linear scan<br>• `add`/`remove`: O(n), linear scan for uniqueness plus full array copy |
| `EnumSet` | ✅ (ordinal) | ❌ | None | Bounded (enum's fixed constant set) | • `add`/`remove`/`contains`: O(1), single bitwise operation |

`ConcurrentSkipListSet` and `CopyOnWriteArraySet` map onto the same dimensions as their `Map`/`List`-backed equivalents, because that's literally what they are underneath — a `ConcurrentSkipListSet` is a `ConcurrentSkipListMap` with the values discarded, the same relationship `HashSet` has to `HashMap`. `EnumSet` doesn't fit the pattern at all, which is exactly why it gets its own callout below.

**Queue & Deque implementations**

| Collection | Ordered | Thread-Safe | Synchronization | Blocking | Bounded | Time Complexity |
|---|---|---|---|---|---|---|
| `PriorityQueue` | Head-only (priority) | ❌ | None | ❌ | Unbounded | • `offer`/`poll` (heap insert/remove): O(log n)<br>• `peek`: O(1) |
| `ArrayDeque` | ✅ | ❌ | None | ❌ | Unbounded (resizable) | • `offer`/`poll`/`push`/`pop` at either end: O(1) amortized<br>• indexed access: not supported |
| `LinkedList` (as Deque) | ✅ | ❌ | None | ❌ | Unbounded | • `offer`/`poll`/`push`/`pop` at either end: O(1)<br>• `get(index)`: O(n) |
| `ConcurrentLinkedQueue` | FIFO | ✅ | Lock-Free — CAS on the tail node's `next` pointer (Michael-Scott non-blocking queue algorithm) | ❌ | Unbounded | • `offer`/`poll`: O(1) amortized |
| `ConcurrentLinkedDeque` | ✅ | ✅ | Lock-Free — CAS-based doubly-linked list, extending the Michael-Scott algorithm to both ends | ❌ | Unbounded | • `offer`/`poll`/`push`/`pop` at either end: O(1) amortized |
| `ArrayBlockingQueue` | FIFO | ✅ | Lock-Based — single lock shared by `put` and `take` | ✅ | Bounded (fixed at construction) | • `put`/`take`: O(1) plus unbounded wait when full/empty |
| `LinkedBlockingQueue` | FIFO | ✅ | Lock-Based — two-lock design, separate locks for `put` and `take` | ✅ | Optionally bounded | • `put`/`take`: O(1) plus unbounded wait when full/empty |
| `LinkedBlockingDeque` | ✅ | ✅ | Lock-Based — single lock shared by both ends (deque operations can touch head and tail together) | ✅ | Bounded | • `put`/`take` at either end: O(1) plus unbounded wait when full/empty |
| `PriorityBlockingQueue` | Head-only (priority) | ✅ | Lock-Based — single lock guarding the underlying heap array | ✅ (on `take`, not `put`) | Unbounded | • `put` (heap insert): O(log n)<br>• `take` (heap remove): O(log n) plus unbounded wait when empty |
| `DelayQueue` | Head-only (by delay expiry) | ✅ | Lock-Based — single lock guarding the underlying heap array | ✅ | Unbounded | • `put` (heap insert): O(log n)<br>• `take`: O(log n) plus unbounded wait until the head's delay expires |
| `SynchronousQueue` | N/A (holds nothing) | ✅ | Lock-Free — CAS-based dual stack/queue algorithm (Scherer-Scott-Lea) | ✅ | Zero-capacity (rendezvous) | • `put`/`take`: O(1) handoff plus unbounded wait for a matching thread |
| `LinkedTransferQueue` | FIFO | ✅ | Lock-Free — CAS-based dual-queue algorithm (nodes represent either data or a waiting consumer) | ✅ (optional, producer-side) | Unbounded | • `put`/`poll`: O(1) amortized<br>• `transfer`: O(1) amortized plus unbounded wait for a consumer to receive |

This is the family where nearly every dimension gets exercised at once, which is why it has almost twice as many implementations as any other. Note `PriorityQueue` and `PriorityBlockingQueue` are marked "head-only" rather than "sorted" — that's the correction from earlier: a priority queue guarantees the next element you poll is the smallest (or largest), not that a full iteration comes back in order. `DelayQueue` is a variation on the same idea, ordering by "time until this element's delay expires" instead of a comparator. And `SynchronousQueue` genuinely breaks the "Ordered" column, since a structure that never holds more than zero elements at rest doesn't have an ordering to speak of — it's included here specifically because it's the cleanest illustration of what "zero-capacity" as a boundedness value actually means. `SynchronousQueue` is also worth a specific correction: it's easy to assume anything this simple must be lock-based, but since Java 6 it's implemented as a CAS-based dual stack/queue with no traditional lock at all.

`ArrayDeque` and `LinkedList` used as a `Deque` show the same pattern one level up in this table: identical ordering, identical thread-safety (none), identical O(1) time complexity at both ends. The table alone gives no reason to prefer one. The real difference is exactly what Big-O hides — `LinkedList` allocates a new node object on every insertion and pays for pointer-chasing on every access, while `ArrayDeque` is backed by a contiguous resizable array with no per-element allocation and far better cache locality. Both are O(1) amortized at the ends; in practice `ArrayDeque` is consistently faster, which is exactly why the JDK's own documentation recommends it over `LinkedList` for stack and queue use.

`ArrayBlockingQueue` and `LinkedBlockingQueue` are worth a closer look too, because on paper they look almost identical — same ordering, same thread-safety, same O(1) time complexity, both blocking. What actually separates them doesn't show up as a difference in Big-O at all; it shows up in the Synchronization Strategy column once you look past the generic "Lock-Based" label. `ArrayBlockingQueue` guards both ends of the queue with a single lock, so a `put()` and a `take()` cannot proceed at the same time even when the queue is neither full nor empty — producers and consumers actively contend with each other on every operation. `LinkedBlockingQueue` uses two independent locks, one for the head and one for the tail, so a producer and a consumer can run concurrently without ever touching the same lock. Under light load the two are indistinguishable; under sustained concurrent producer-consumer traffic, `LinkedBlockingQueue`'s split-lock design gives it meaningfully higher throughput, which is exactly the kind of difference a same-looking Big-O row can hide. `LinkedBlockingQueue` also pays a cost `ArrayBlockingQueue` doesn't: because it's backed by linked nodes rather than a pre-allocated array, every `put()` allocates a new node, adding garbage-collection pressure that the array-backed queue avoids entirely.

**Map implementations**

| Collection | Ordered | Thread-Safe | Synchronization | Bounded | Time Complexity |
|---|---|---|---|---|---|
| `HashMap` | ❌ | ❌ | None | Unbounded | • `get`/`put`/`remove`/`containsKey`: O(1) average |
| `LinkedHashMap` | ✅ | ❌ | None | Unbounded | • `get`/`put`/`remove`/`containsKey`: O(1) average |
| `TreeMap` | ✅ (Sorted) | ❌ | None | Unbounded | • `get`/`put`/`remove`: O(log n)<br>• navigation (`floorKey`, `ceilingKey`, `firstKey`, `lastKey`): O(log n) |
| `Hashtable` | ❌ | ✅ | Lock-Based (legacy) — single lock shared by every method | Unbounded | • `get`/`put`/`remove`: O(1) average, lock acquired on every call |
| `ConcurrentHashMap` | ❌ | ✅ | CAS for uncontended inserts into an empty bin; a `synchronized` block scoped to that one bin on collision or resize — never one global lock | Unbounded | • `get`: O(1) average, typically no locking at all<br>• `put`/`remove`: O(1) average |
| `ConcurrentSkipListMap` | ✅ (Sorted) | ✅ | Lock-Free — CAS on skip-list node pointers at each level (Fraser-style lock-free skip list) | Unbounded | • `get`/`put`/`remove`: O(log n) expected<br>• navigation (`floorKey`, `ceilingKey`, `firstKey`, `lastKey`): O(log n) expected |
| `EnumMap` | ✅ (ordinal) | ❌ | None | Bounded (enum's fixed constant set) | • `get`/`put`/`remove`: O(1), direct array indexing by ordinal |
| `WeakHashMap` | ❌ | ❌ | None | Unbounded, but self-pruning | • `get`/`put`/`remove`: O(1) average |
| `IdentityHashMap` | ❌ | ❌ | None | Unbounded | • `get`/`put`/`remove`: O(1) average |

Look closely at `HashMap` and `IdentityHashMap`: every column matches, and that's not an oversight. The six dimensions genuinely cannot tell them apart, because the real difference — reference equality (`==`) instead of `.equals()` for deciding "same key" — isn't a performance or concurrency trade-off at all, it's a different definition of "same." See the equality-semantics reasoning in the Domain-Specialized Collections section below for why that distinction still matters despite being completely invisible in this table.

`Hashtable` slots in next to `ConcurrentHashMap` on thread-safety but, like `Vector`, gets there with one lock around everything — a direct ancestor of `Collections.synchronizedMap()`, not a competitor to the modern concurrent maps. `EnumMap` and `WeakHashMap`, on the other hand, don't lose to the six dimensions so much as answer a question the six dimensions don't ask at all. Both groups get their own reasoning below rather than being forced into this table's columns.

## Choosing a Collection in Practice

When you are actually designing a system and need to pick a collection, the six dimensions translate into six questions worth asking in order, because each answer narrows the field considerably.

Start with performance: what can this operation actually afford to cost? If you need the cheapest possible get and put and nothing else matters, you are already headed toward a hash-based structure. If you can tolerate O(log n) in exchange for a guarantee, sorted structures come into play. This question comes first because it sets a ceiling — no amount of ordering or concurrency reasoning matters if the base complexity of the structure was wrong for your workload to begin with.

Next, think about how the data needs to be organized. If you don't care about order at all, you have opened the door to the fastest possible implementations. If you need insertion order preserved, you have ruled out plain hash-based structures. If you need the data sorted at all times, you have committed to logarithmic-time operations whether you wanted to or not — this is where the performance question and the ordering question start pulling in opposite directions, and you have to decide which one wins.

After that, think about who touches the collection. A collection that only one thread ever sees does not need any synchronization at all, and adding it anyway is pure overhead. A collection with many readers and occasional writers is a very different problem from one with many concurrent writers, and the two call for different implementations — this is exactly the distinction that makes `CopyOnWriteArrayList` sensible for the first case and actively harmful for the second.

If you do need concurrency, decide how much contention you can tolerate. Lock-based synchronization is simple to reason about but serializes access under load. Lock-free structures built on compare-and-swap avoid that serialization but are harder to build correctly and are not automatically faster in every scenario. Copy-on-write structures make writes expensive so that reads can be essentially free, which only makes sense when reads vastly outnumber writes.

Ask whether your consumers should ever wait. A non-blocking queue returns immediately, empty or not, and leaves the waiting logic to you. A blocking queue lets a consumer thread simply park until a producer supplies something, which is precisely what makes it the natural backbone of producer-consumer pipelines.

Finally, if you landed on a queue, ask what should happen when it's full. This question only applies to queues, but when it applies, it matters as much as any of the others. An unbounded queue accepts work forever and pushes the failure mode downstream to "the JVM eventually runs out of memory," which is a much worse place to discover a capacity problem than at the point of insertion. A bounded queue forces you to decide, up front, whether a full queue should block the producer, reject the new element, or drop something — and that decision is usually the difference between a system that degrades gracefully under load and one that falls over silently until it doesn't.

Answering these six questions in order usually takes you from "there are forty implementations" to "there are one or two reasonable choices" — and at that point, the decision is easy.

## Why This Framing Matters More Than It Seems

It is tempting to treat this as a purely academic exercise, but the six-dimension model has direct consequences on production systems that are easy to learn the hard way.

- Reaching for `HashMap` in a multi-threaded context without external synchronization does not fail loudly — it corrupts silently, producing infinite loops or lost entries under load that only show up once traffic increases.
- Choosing `CopyOnWriteArrayList` for a write-heavy workload does not throw an exception either — it just gets progressively slower as every write copies the entire backing array, and the degradation is easy to misattribute to something else entirely.
- Assuming `LinkedHashMap` and `HashMap` are interchangeable because they share most of the same API ignores that one of them is silently paying a small constant cost on every operation to maintain an ordering guarantee you may not even be using.
- Defaulting to an unbounded `LinkedBlockingQueue` because it was the first constructor that compiled does not fail on day one either — it fails the day a downstream consumer slows down and the queue grows without limit until the process is killed for memory pressure.

"It works in my local tests" and "it holds up under concurrent production load" are very different bars, and the gap between them is almost always explained by one of these six dimensions being ignored rather than deliberately chosen.

## A Common Misconception

A mistake that trips up even experienced developers: assuming that "thread-safe" is a single, uniform property that either a collection has or doesn't. It isn't. `Collections.synchronizedMap()`, `ConcurrentHashMap`, and `CopyOnWriteArrayList` are all thread-safe, and they achieve it through three completely different mechanisms with three completely different performance profiles under contention.

A synchronized wrapper puts a single lock around every operation — simple, correct, and a bottleneck the moment more than a couple of threads want in at once. `ConcurrentHashMap` partitions its internal state so that unrelated operations rarely block each other at all. `CopyOnWriteArrayList` does not use locks for reads in the traditional sense — it hands out a stable snapshot of the array and only pays a synchronization cost when someone writes. Calling all three "thread-safe" and stopping there erases exactly the information you need to pick correctly.

## What "Ordered" Actually Means

When engineers say a collection is "ordered," they usually mean one of three genuinely different things, and conflating any two of them causes real bugs. Insertion order, as guaranteed by `LinkedHashMap` or `ArrayList`, means elements come back in the sequence you added them, and nothing more. Sort order, as guaranteed by `TreeMap` or `TreeSet`, means elements come back according to a comparator, regardless of when they were inserted — full iteration is guaranteed to be sorted. A `HashMap` guarantees neither, and the fact that it often *appears* to iterate in a stable order on a given JVM version is an implementation detail, not a contract — code that quietly depends on it is one JDK upgrade away from breaking.

The third meaning is priority order, and it's the one that gets conflated with sort order most often, because they sound identical. `PriorityQueue` guarantees only that `peek()` and `poll()` return the smallest (or largest) remaining element by some comparator — it says nothing about the order of the elements sitting behind the head. Internally it's a binary heap, and a heap is only partially ordered: a parent is guaranteed to be smaller than its children, but two sibling subtrees have no defined relationship to each other. If you iterate a `PriorityQueue` directly with a `for` loop instead of repeatedly calling `poll()`, you will not get elements back in priority order — you'll get them back in whatever order the heap's internal array happens to store them. This is a genuinely common bug: code that treats `PriorityQueue` as "a `TreeSet` you can iterate," when the only sorted-access path it actually offers is repeated removal from the head.

## Legacy Collections: Old Answers to a Question the Framework Later Solved Better

`Vector`, `Stack`, and `Hashtable` all predate the Java Collections Framework itself — they were part of Java 1.0, before `List`, `Set`, or `java.util.concurrent` existed, and were retrofitted to implement the modern interfaces later for compatibility. All three achieve thread safety the same way: a single lock, held for the duration of every method call, with no distinction between a read and a write. It's the same strategy `Collections.synchronizedMap()` and `Collections.synchronizedList()` apply today as a wrapper — except in these three classes it's welded into the implementation rather than opted into.

That single-lock strategy is also exactly why they've been superseded. `Vector` was displaced by `ArrayList` for single-threaded code (paying for a lock you don't need is pure waste) and by `CopyOnWriteArrayList` or a wrapped `ArrayList` for concurrent code (finer-grained or lock-free strategies outperform a single coarse lock under real contention). `Hashtable` was displaced by `ConcurrentHashMap` for the same reason. `Stack` is a particularly awkward case: it's technically a subclass of `Vector`, which means it inherits index-based access and insertion-in-the-middle methods that have no business being on a stack at all — the Javadoc itself recommends `ArrayDeque` instead, which implements the same LIFO behavior without the inherited baggage or the lock.

None of this means the three are unsafe to use — they still do exactly what they say. It means that in new code, whatever `Vector`, `Stack`, or `Hashtable` are doing for you, one of their modern replacements almost certainly does it faster, and you'd only reach for the originals today for legacy API compatibility, not because they won on any of the six dimensions.

## Domain-Specialized Collections: When the Key Space Itself Is the Optimization

A handful of implementations don't fit anywhere on the six dimensions, because they aren't answering "how do I balance ordering against speed" or "how many threads can touch this at once" — they're solving a narrower problem so completely that the whole dimension model becomes beside the point.

**Reference strength and garbage collection.** Every collection covered so far holds strong references to its contents, which means an object sitting inside a `HashMap` cannot be garbage-collected no matter how starved the rest of the program is for memory — the map itself is keeping it alive. `WeakHashMap` breaks that assumption on purpose: its keys are wrapped in `WeakReference`s, so the moment nothing outside the map still holds a strong reference to a key, the garbage collector is free to reclaim it, and the corresponding entry quietly disappears from the map on its own. This is the standard building block for caches and metadata registries that should never be the reason an object stays alive — you get automatic cleanup without writing any eviction logic yourself. None of the six dimensions capture this, because "how does this collection interact with the garbage collector" isn't a question ordering, concurrency, or complexity can answer.

**Equality semantics.** Every hash-based collection so far has silently assumed that "the same key" means `key1.equals(key2)`. `IdentityHashMap` throws that assumption out and uses reference equality (`==`) instead, so two distinct objects that are `.equals()`-equal but not the same object in memory are treated as two separate keys. This sounds like a bug generator until you hit the specific problems it exists for: serialization frameworks and object-graph traversal need to track "have I already visited this exact object" to detect cycles, and using `.equals()` for that would incorrectly collapse distinct-but-equal objects into one. This, too, sits outside the six dimensions entirely — it's not a trade-off between speed and ordering, it's a different definition of "same."

**Bit vectors and ordinal indexing.** `EnumSet` and `EnumMap` earn their place by exploiting something no general-purpose collection can assume: the complete set of possible keys is small, fixed, and known at compile time, because it's exactly the constants of one `enum` type. `EnumSet` is typically backed by a single `long` (or an array of them for larger enums) used as a bitmask, where membership of the *n*-th enum constant is just the *n*-th bit — insertion, removal, and membership checks become single bitwise operations instead of a hash lookup. `EnumMap` similarly backs itself with a plain array indexed by the enum constant's ordinal, turning every `get`/`put` into direct array indexing with no hashing at all. Both are meaningfully faster and more memory-compact than their `HashSet`/`HashMap` equivalents, but only because the problem was narrowed enough to make an array a valid substitute for a hash table in the first place — the trick doesn't generalize to any other kind of key.

The pattern across all three: none of them lose to `HashMap` or `ConcurrentHashMap` on the six dimensions. They're not competing on that axis at all — they've each identified a constraint (weak reachability, identity rather than equality, a small fixed key domain) specific enough that a completely different, simpler data structure becomes not just viable but strictly better.

## Footnote



The Java Collections Framework is a small set of interfaces — `List`, `Set`, `Queue`, `Deque`, and the interface-adjacent `Map` — layered with implementations that make different trade-offs across six dimensions: time complexity, ordering, thread safety, synchronization strategy, blocking behavior, and boundedness. Nearly every implementation decision in the framework is a direct consequence of choosing a point along these six axes — with two exceptions worth remembering: a handful of legacy classes that answer these questions in an outdated way, and a handful of domain-specialized ones that opt out of the questions entirely by narrowing the problem down to something a hash table was never built for.

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
- Blocking Queues and the Cost of Unbounded Growth
- ArrayDeque vs LinkedList
- Why WeakHashMap Doesn't Need Manual Eviction
- EnumSet and EnumMap: Collections as Bitmasks
- [`Sentinel Boundaries`](./sentinel_boundaries.md)
  

If you found this useful, share it with someone still importing `Vector` in 2026, and keep an eye out for the next piece in the series: **Ordered vs Sorted vs Unordered Collections**.
