# Skip Lists in Java: Why `ConcurrentSkipListMap` Exists, and When You Actually Need It

If you have written concurrent Java, you have used `ConcurrentHashMap`. It is fast, it is battle-tested, and it is the default answer whenever someone asks "how do I safely share a map across threads?"

But tucked away in `java.util.concurrent` sit two collections most developers never touch: `ConcurrentSkipListMap` and `ConcurrentSkipListSet`. Everyone knows they exist. Very few know why. Fewer still know when to reach for them over `ConcurrentHashMap`.

This article builds that intuition from first principles, and ends with a small trading engine where multiple threads hammer prices into a shared structure while other threads run live range queries against it - concurrently, without a single explicit lock.

## The Problem: Ordered Data Under Concurrent Writes

Picture a stock exchange. Thousands of price ticks arrive every second from multiple feeds:

```
RELIANCE -> 1495.20
TCS      -> 3421.10
INFY     -> 1587.80
HDFC     -> 1743.90
```

Several producer threads are writing prices concurrently. Meanwhile, other parts of the system need answers to questions like:

- What is the cheapest stock right now?
- What is the most expensive?
- Show me every stock trading between ₹1500 and ₹1600.
- What is the next stock priced just above ₹1550?

Notice the shape of these questions. None of them is "give me the value for this exact key." They are all about **order** - lowest, highest, next, previous, everything in a range. That distinction is the entire article.

## Why `ConcurrentHashMap` Falls Short

`ConcurrentHashMap` is built to answer exactly one kind of question extremely well:

```java
map.get("INFY"); // O(1) average
```

But ask it "give me everything between ₹1500 and ₹1600" and there is no shortcut. The map has no notion of order between its keys, so the only correct answer is to walk every entry and check each one - an O(n) scan, no matter how large the map gets. The same is true for finding the minimum, the maximum, or the next higher key. `ConcurrentHashMap` was never designed for navigation, only for retrieval.

## Why Not Just Use a `TreeMap`?

Java already ships a sorted map. A `TreeMap<Double, Stock>` keeps everything ordered automatically - minimum and maximum are cheap, range queries fall out of the API for free. So why not just use it?

Because `TreeMap` is not thread-safe. If one thread is inserting while another is iterating, the underlying red-black tree can be restructured mid-traversal. Best case, you get a `ConcurrentModificationException`. Worst case, you get a silently corrupted read.

The obvious fix is to wrap it:

```java
Collections.synchronizedSortedMap(new TreeMap<>());
```

This works, but it works by putting a single lock around the entire map. Every read and every write now competes for that one lock. One slow writer stalls every reader. As thread count goes up, throughput goes down - the classic shape of a globally synchronized data structure under contention.

## The Skip List Idea

A skip list keeps data sorted without a balanced tree at all, and that is the trick worth understanding.

Start with a plain sorted linked list:

```
1 → 2 → 3 → 4 → 5 → 6 → 7 → 8
```

Finding `8` means walking every node - O(n). Now add express lanes above it, the way a highway skips local exits:

```
Level 2:  1 ------------------> 5 ------------------> 8
Level 1:  1 --------> 3 --------> 5 --------> 7 --------> 8
Level 0:  1 → 2 → 3 → 4 → 5 → 6 → 7 → 8
```

A search for `8` starts at the top level, jumps as far as it can without overshooting, then drops down a level and repeats. Instead of visiting every node, it skips large chunks of the list. Average search time drops to O(log n).

The elegant part is how those levels get built: there is no balancing, no rotation, no recoloring. Each node, on insertion, flips a virtual coin. Heads, it also gets promoted to the level above. Heads again, promoted further. Most nodes stay at the bottom; a few climb to level 2; very few make it to level 5. Over many insertions, this random promotion produces a structure that behaves statistically like a balanced tree - without ever running a balancing algorithm.

That absence of rotations is precisely why skip lists concurrency-friendly. A red-black tree insert can trigger a cascade of rotations and recoloring that touches nodes far from the insertion point. A skip list insert only touches the nodes immediately around it at each level it participates in, which makes it far easier to reason about - and implement - without a single global lock.

## `ConcurrentSkipListMap` and `ConcurrentSkipListSet`

Java implements this idea directly:

- **`ConcurrentSkipListMap<K, V>`** - a thread-safe, sorted map with `O(log n)` get, put, and remove, and lock-free reads.
- **`ConcurrentSkipListSet<E>`** - the set equivalent; think of it as a concurrent, sorted `TreeSet`.

What makes them worth learning is the navigation API that comes for free once data is sorted:

```java
prices.firstEntry();                 // lowest price
prices.lastEntry();                  // highest price
prices.higherEntry(1500.0);          // next price above 1500
prices.lowerEntry(1500.0);           // next price below 1500
prices.tailMap(1500.0);              // everything >= 1500
prices.headMap(2000.0);              // everything < 2000
prices.subMap(1500.0, true, 1600.0, true); // everything in [1500, 1600]
```

Every one of these is `O(log n)` or an efficient view over the existing sorted structure - never a full scan.

## How Does `ConcurrentSkipListMap` Handle Concurrent Reads and Writes?

Understanding the layered structure explains why searches are fast. It doesn't yet explain how multiple threads can safely modify that structure at the same time without a global lock. The answer is a combination of lock-free algorithms, atomic operations, and the Java Memory Model.

### The Bottom Level Is the Source of Truth

A skip list has multiple levels, but the actual data lives only in the bottom-level linked list. Every level above it is just an index - a shortcut that helps a search skip over large sections of that bottom list faster.

```
Level 3 (Index)
Level 2 (Index)
Level 1 (Index)
Level 0 (Actual Data)
```

This separation matters more than it looks. Correctness depends only on the bottom list being right. The upper levels are performance optimizations, and they can be updated independently - even lazily - without ever putting correctness at risk.

### No Global Locks

Unlike a synchronized `TreeMap`, `ConcurrentSkipListMap` does not wrap the whole structure in one lock. It relies on Compare-And-Swap (CAS) to update pointers atomically, one pointer at a time.

Picture two threads trying to insert between the same two nodes:

```
10 -> 20
```

Both threads locate the insertion point and prepare their new node. The first thread performs a CAS that changes `10.next` from `20` to its new node, and it succeeds. When the second thread attempts the same CAS, it fails - `10.next` no longer points to `20`, it points to the node the first thread just inserted. Instead of blocking, the second thread simply re-reads the structure and retries the insertion at the now-correct position.

This is an optimistic strategy: assume no conflict, attempt the update, and only pay a retry cost when two threads genuinely collide. Unrelated updates elsewhere in the list proceed without ever touching each other's CAS operations.

### Why Readers Never Block

Every node stores its links in `volatile` fields, which guarantees that a pointer update made by one thread becomes visible to every other thread immediately, without needing a lock to force that visibility.

That means a reader traversing the list never has to wait for a writer. While one thread is mid-insert or mid-remove, a reader can keep walking the list concurrently. It might observe the structure just before the update or just after it - both are valid, consistent states - but it will never observe a half-updated, torn structure, because each pointer swap is a single atomic operation.

### Building the Index Levels

Once a new node is safely linked into the bottom list, the skip list gets around to building its upper index levels for that node. If a search happens to run before those index levels exist yet, it isn't wrong - it just falls through to the lower levels sooner and walks a few extra nodes before reaching the same correct answer.

The index is allowed to lag behind. The data is not.

### Why It Scales

Every operation synchronizes only on the specific pointers it touches, not on the map as a whole. Two threads inserting at opposite ends of the price range never contend with each other at all. Reads stay entirely lock-free. Writes only pay a retry when they genuinely collide with another write at the same spot.

This is the actual mechanism behind the trading engine below: three producer threads can hammer `put()` concurrently, a reader can call `firstEntry()` and `lastEntry()`, and a range query can call `subMap()` - all at the same time, on the same map - because none of them are waiting on one giant lock. CAS handles conflicting writes, `volatile` handles visibility, and the bottom-level list stays correct throughout.

## A Trading Engine, End to End

Here is a small market data engine that puts this to work. Three producer threads continuously push price updates into a shared `ConcurrentSkipListMap`. At the same time, one thread reports the best and worst prices, and another runs a live range query - all against the same structure, with no synchronized blocks anywhere in application code.

```java
import java.util.Map;
import java.util.Random;
import java.util.concurrent.*;

public class MarketDataEngine {

    private final ConcurrentSkipListMap<Double, String> orderBook =
            new ConcurrentSkipListMap<>();

    private final Random random = new Random();

    private final String[] stocks = {
            "INFY", "TCS", "HDFC", "RELIANCE", "ICICI", "SBIN"
    };

    public void start() {
        ExecutorService executor = Executors.newFixedThreadPool(6);

        // Three producer threads simulating market data feeds
        for (int i = 0; i < 3; i++) {
            executor.submit(() -> {
                while (true) {
                    double price = 1400 + random.nextInt(300);
                    String stock = stocks[random.nextInt(stocks.length)];
                    orderBook.put(price, stock);
                    sleep(300);
                }
            });
        }

        // Reader thread: best and worst price, live
        executor.submit(() -> {
            while (true) {
                System.out.println("Lowest Price : " + orderBook.firstEntry());
                System.out.println("Highest Price: " + orderBook.lastEntry());
                sleep(2000);
            }
        });

        // Range query thread: everything trading between 1500 and 1550
        executor.submit(() -> {
            while (true) {
                System.out.println("Stocks between 1500 and 1550");
                for (Map.Entry<Double, String> e :
                        orderBook.subMap(1500.0, true, 1550.0, true).entrySet()) {
                    System.out.println(e);
                }
                sleep(4000);
            }
        });
    }

    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) {
        new MarketDataEngine().start();
    }
}
```

Walk through what is actually happening. Three writer threads are hammering `put()` concurrently, each on its own schedule, each possibly touching overlapping price levels. Meanwhile, the reader thread calls `firstEntry()` and `lastEntry()` every two seconds, and the range-query thread calls `subMap()` every four seconds. None of these operations block each other behind a shared lock. There is no `synchronized` keyword, no manual copy of the map before iterating, no `ConcurrentModificationException` to catch. The skip list's internal structure is built specifically to let inserts, removes, and sorted traversals interleave safely.

Now imagine building the same engine on top of a `ConcurrentHashMap`. The `subMap` query becomes a full `for (entry : map.entrySet())` loop with a manual price-range check on every entry, every four seconds, forever. At 100 stocks that is fine. At 50,000 instruments across an exchange, that "just scan everything" query is now the most expensive line of code in your system - and it gets more expensive as the map grows, while the skip list's cost stays `O(log n)`.

## Performance at a Glance

| Operation   | `ConcurrentHashMap` | `ConcurrentSkipListMap`  |
| ----------- | -------------------: | -----------------------: |
| `get()`     | O(1) average          | O(log n)                 |
| `put()`     | O(1) average          | O(log n)                 |
| `remove()`  | O(1) average          | O(log n)                 |
| `firstKey()`| not supported         | O(log n)                 |
| `lastKey()` | not supported         | O(log n)                 |
| `higherKey()`/`lowerKey()` | not supported | O(log n)            |
| `subMap()`  | not supported (manual scan) | efficient sorted view |
| Iteration order | unordered         | sorted                   |

## When Should You Reach for It?

Use `ConcurrentSkipListMap` or `ConcurrentSkipListSet` when all three of these are true at once:

1. Multiple threads read and write the collection concurrently.
2. The data must stay sorted at all times.
3. Your queries are navigational - "next," "previous," "highest," "lowest," or "everything between X and Y" - rather than pure key lookups.

This shows up in order books, leaderboards, time-series event stores, delay queues ordered by execution time, expiring cache entries, and monitoring systems tracking metrics over time.

If your access pattern is purely "do you have this key," `ConcurrentHashMap` remains the right - and faster - default.

## Footnote

`ConcurrentHashMap` answers "do you have this key?" as fast as anything in the JDK. `ConcurrentSkipListMap` answers a different question - "what comes next?" - and it does so concurrently, without a global lock, by trading a balanced tree's rotations for a randomized ladder of express lanes over a sorted linked list. When your system needs both concurrency and ordered navigation at the same time, that trade is usually worth making.
