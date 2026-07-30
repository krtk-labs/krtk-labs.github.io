# The Candy Problem: Why One Pass Fails and Two Passes Fix It

You have a line of kids, each with a rating. You need to hand out candy such that every kid gets at least one, and any kid with a higher rating than a neighbor gets more candy than that neighbor. It sounds like a small, mechanical rule. It is not. This rule is *bidirectional* - it talks about the neighbor on the left and the neighbor on the right at the same time - and that single fact is what breaks the standard greedy instinct.

This article walks through exactly why a single left-to-right scan cannot solve this problem, what a "pass" actually means, and why the fix - two passes, merged with `max` - is not an arbitrary trick but a direct consequence of the constraint's shape.

## The Instinct That Fails

The natural first attempt looks like this: walk through the array left to right, and whenever a kid's rating is higher than the kid before them, give them one more candy than that previous kid.

Try it on `[1, 2, 1]`. Walking left to right:

- Kid 0: base case, 1 candy.
- Kid 1: rating `2 > 1`, so give them `previous + 1 = 2`.
- Kid 2: rating `1`, compared to kid 1's rating of `2` - not higher, so they fall back to the minimum, 1 candy.

Result: `[1, 2, 1]`. Check it against the rule: kid 1 has a higher rating than kid 0, and gets more candy. Good. But kid 1 *also* has a higher rating than kid 2, and kid 2 only has 1 candy while kid 1 has 2 - so kid 2 correctly has less. This one happens to work.

Now try `[1, 3, 2]` with the same idea, but notice the trap more carefully with `[3, 2, 1]`:

- Kid 0: base case, 1 candy.
- Kid 1: rating `2`, compared to `3` - not higher, so 1 candy.
- Kid 2: rating `1`, compared to `2` - not higher, so 1 candy.

Result: `[1, 1, 1]`. But the rule says kid 0 has a higher rating than kid 1, so kid 0 needs *more* candy than kid 1. It doesn't. The scan never had a mechanism to notice this, because it only ever compares a kid to the one *before* them, and kid 0 has no "before."

This is the core problem: **a single left-to-right scan can only enforce the rule in one direction.** It has no way to look backward and fix an earlier kid once it learns something new about a later kid.

## What a "Pass" Actually Means

Before fixing this, it's worth being precise about what a pass is, because the fix depends on a rule that is easy to state loosely and get wrong.

A pass is a single walk through the array, in some fixed direction, where at each index you assign a candy value using *only* values you have already finalized earlier in that same walk.

That last part is the entire ballgame. Say you're walking left to right and you're standing at index `i`. The only value you're allowed to trust is `candies[i-1]`, because that's the only index you've already visited and locked in. You are not allowed to reach forward to `candies[i+1]`, because you haven't computed it yet - it's still sitt at some default placeholder, not its true final value.

So a valid left-to-right pass has to look like this:

```
for i from 1 to n-1:
    if ratings[i] > ratings[i-1]:
        candies[i] = candies[i-1] + 1
    else:
        candies[i] = 1
```

Compare `ratings[i]` against `ratings[i-1]`, and assign to `candies[i]` using `candies[i-1]`. The index being *compared* and the index being *read from* are the same, and that index is always the one already behind you.

Here's the mistake that's easy to fall into: comparing `ratings[i]` to `ratings[i+1]` while still walking left to right, and assigning to `candies[i]` using `candies[i+1]`. On the surface this also "looks at both neighbors," but it silently reads a value - `candies[i+1]` - that hasn't been computed yet. You end up assigning `candies[i]` its final value based on a placeholder, and since you never revisit `i` again in this pass, that wrong value is now locked in forever.

Run this broken version on `[7, 3, 1, 1, 2, 4]` and you'll get `[2, 2, 1, 1, 1, 1]` - notice the "shape" is inverted. The bump sits at the start, where ratings are actually *decreasing* (7, 3, 1), which is exactly where a left-to-right pass should stay flat. That inversion is the fingerprint of reading from an index you haven't visited yet.

## Running the Two Correct Passes

Once the rule is "only read from what you've already visited," there are two clean passes available, one per direction.

**Left pass** - walk left to right, compare each kid to the one behind them:

```
left[i] = left[i-1] + 1   if ratings[i] > ratings[i-1]
left[i] = 1               otherwise
```

On `[7, 3, 1, 1, 2, 4]`:

| index | rating | comparison | left[i] |
|---|---|---|---|
| 0 | 7 | base case | 1 |
| 1 | 3 | `3 > 7`? no | 1 |
| 2 | 1 | `1 > 3`? no | 1 |
| 3 | 1 | `1 > 1`? no | 1 |
| 4 | 2 | `2 > 1`? yes | 2 |
| 5 | 4 | `4 > 2`? yes | 3 |

`left = [1, 1, 1, 1, 2, 3]`. Notice the rise happens at the *end* of the array, exactly where ratings are actually climbing (1, 2, 4). That's the tell that this pass is correct.

**Right pass** - walk right to left, compare each kid to the one ahead of them:

```
right[i] = right[i+1] + 1   if ratings[i] > ratings[i+1]
right[i] = 1                 otherwise
```

| index | rating | comparison | right[i] |
|---|---|---|---|
| 5 | 4 | base case | 1 |
| 4 | 2 | `2 > 4`? no | 1 |
| 3 | 1 | `1 > 2`? no | 1 |
| 2 | 1 | `1 > 1`? no | 1 |
| 1 | 3 | `3 > 1`? yes | 2 |
| 0 | 7 | `7 > 3`? yes | 3 |

`right = [3, 2, 1, 1, 1, 1]`. Here the rise happens at the *start*, exactly where ratings are actually falling (7, 3, 1). Also correct, by the mirror-image logic.

Each pass, on its own, only enforces half the rule. `left` guarantees "if you beat your left neighbor's rating, you beat their candy count." `right` guarantees the same thing for the right neighbor. Neither one, by itself, satisfies the full problem.

## Merging the Two Passes

The final answer is `candies[i] = max(left[i], right[i])`:

| index | left[i] | right[i] | candies[i] |
|---|---|---|---|
| 0 | 1 | 3 | 3 |
| 1 | 1 | 2 | 2 |
| 2 | 1 | 1 | 1 |
| 3 | 1 | 1 | 1 |
| 4 | 2 | 1 | 2 |
| 5 | 3 | 1 | 3 |

`candies = [3, 2, 1, 1, 2, 3]`. Check it by hand against the original ratings `[7, 3, 1, 1, 2, 4]`: 7 > 3 > 1 is a strictly falling run, and 3 > 2 > 1 mirrors it correctly. 1, 2, 4 is a strictly rising run, and 1, 2, 3 mirrors it correctly. Every neighbor condition holds.

## Why Max, and Not Min

This is the part that deserves slowing down, because "just take the max" without knowing why is the kind of thing that falls apart the moment the problem is tweaked slightly.

Think of `left[i]` and `right[i]` as two separate *demands* on kid `i`'s candy count:

- `left[i]` says: "I need at least this many candies to beat my left neighbor, if my rating is higher than theirs."
- `right[i]` says: "I need at least this many candies to beat my right neighbor, if my rating is higher than theirs."

Both demands have to hold **at the same time** - the problem requires the left condition and the right condition to both be true simultaneously, not one or the other. If a kid needs at least 3 candies to satisfy the left demand, and at least 2 to satisfy the right demand, giving them only 2 candies breaks the left demand. Giving them 3 satisfies both, because 3 already clears the smaller bar of 2. So the number that satisfies two simultaneous "at least this much" demands is whichever demand is bigger - the `max`.

If you used `min` instead, you'd be satisfying only the weaker of the two demands, and silently breaking the stronger one. Try it by hand on `[1, 2, 2]`:

- `left = [1, 2, 1]` (kid 1 beats kid 0's rating, kid 2 does not beat kid 1's rating since they're tied)
- `right = [1, 1, 1]` (no kid beats the one to their right)
- `min(left, right) = [1, 1, 1]`

But the rule says kid 1's rating (2) is higher than kid 0's rating (1), so kid 1 needs more candy than kid 0. With `min`, both have 1 candy. The rule is broken. Swap in `max` and you correctly get `[1, 2, 1]`.

## Why the Merge Doesn't Break Anything

There's a subtlety worth addressing directly: when you take `max(left[i], right[i])`, could that somehow damage the guarantee that `left` gave you? Concretely - if `ratings[i] > ratings[i-1]`, we know `left[i] > left[i-1]`. But after merging, is it still true that `candies[i] > candies[i-1]`?

Here's the reasoning, without the algebra getting in the way. Since `candies[i] = max(left[i], right[i])`, we know `candies[i]` is at least as big as `left[i]`, which is already bigger than `left[i-1]`. So `candies[i]` clears `left[i-1]` easily. The only way the merge could go wrong is if `candies[i-1]` jumped up *above* `left[i-1]` - meaning `right[i-1]` was the bigger of the two at position `i-1`.

But here's the fact that saves us: `right[i-1]` being large means kid `i-1` beats kid `i`'s rating (a falling step going into `i`). We started by assuming the opposite - `ratings[i] > ratings[i-1]`, a *rising* step. A single step can't be both rising and falling at once. So whenever `left[i] > left[i-1]` actually matters (rating rising into `i`), `right[i-1]` is guaranteed to just be sitting at its minimum value of 1, not something inflated. That means `candies[i-1]` can't have snuck above `left[i-1]` in this case, and the inequality `candies[i] > candies[i-1]` survives the merge cleanly. The same argument runs in mirror image for the right pass and falling steps.

The short version: each pass's guarantee only "matters" exactly where the merge can't interfere with it, because the two passes are triggered by opposite conditions (rising vs. falling) that can never both fire on the same step.

## The Pattern to Remember

This problem is one instance of a shape that shows up constantly once you know to look for it:

> A constraint depends on *both* sides of an element (both neighbors, both past and future, both before and after). A single directional scan can only see one side cleanly. Split it into two one-directional passes - one for each side - then merge with `max` if both are lower-bound demands, or `min` if both are upper-bound demands.

Same shape, different clothes:

- **Trapping Rain Water** - water trapped at index `i` depends on the tallest wall to the left *and* the tallest wall to the right. Build `leftMax[]` and `rightMax[]` separately, then merge with `min` (the water level is capped by the *shorter* of the two walls).
- **Product of Array Except Self** - the product at index `i` needs everything to the left multiplied together *and* everything to the right multiplied together. Build `prefixProduct[]` and `suffixProduct[]`, then merge by multiplying them.
- **Candy** - the candy count at index `i` needs to beat the left neighbor *and* the right neighbor when applicable. Build `left[]` and `right[]`, merge with `max`.

The moment you notice a rule that references both sides of a position, that's the signal. Don't try to force it into one pass. Split it, solve each half separately where the logic is clean and provably correct, then combine with whichever operator matches what "both constraints holding at once" actually means for that problem.
