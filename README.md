# k closest points to origin: drop the sqrt, use a size-k heap, and defend the tradeoff out loud

You can solve this problem in four lines with a sort, and that answer passes every hidden test LeetCode has for it. It's also the answer that gets you a "can we do better?" and then silence for twenty seconds.

So the useful thing to learn here isn't the four lines. It's the reason a size-k heap exists, why the heap is a *max*-heap when you want the *closest* points, and how to say all of that out loud before you type.

## The problem, stated plainly

Given an array `points` where `points[i] = [xi, yi]` represents a point on the X-Y plane, and an integer `k`, return the k closest points to the origin `(0, 0)`. Distance is Euclidean. Any order in the answer is accepted, and the answer is guaranteed unique apart from ordering.

Two examples cover almost everything you need to reason about:

- `points = [[1,3],[-2,2]], k = 1` → `[[-2,2]]`. Distance to `(1,3)` is √10, to `(-2,2)` is √8. √8 is smaller, so `(-2,2)` wins.
- `points = [[3,3],[5,-1],[-2,4]], k = 2` → `[[3,3],[-2,4]]`, and `[[-2,4],[3,3]]` is equally correct.

Constraints: `1 <= k <= points.length <= 10^4`, with each coordinate between `-10^4` and `10^4`. Older copies of the statement used `10^3` and coordinate bounds of `100`; the shape of the problem didn't change.

## The first thing to notice: the square root is a trap

Distance from a point to the origin is `sqrt(x² + y²)`. The square root is monotonically increasing for non-negative numbers, which means sorting by `sqrt(x² + y²)` produces exactly the same order as sorting by `x² + y²`.

So compute `d² = x² + y²` and never call `sqrt`. Three reasons this matters in an interview:

1. It's cheaper. Integer multiplication instead of a floating-point square root on every comparison.
2. It sidesteps floating-point comparison problems. `sqrt` on large integers can round two genuinely different distances to the same `double`, and now your comparator thinks they tie.
3. Interviewers notice. Saying "I'll compare squared distances, which preserves the ordering" is a two-second sentence that signals you understand what you're doing rather than pattern-matching a template.

The squared distance is the key for every approach below, including the heap.

## Approach 1: sort everything, take the first k

python
class Solution:
    def kClosest(self, points: List[List[int]], k: int) -> List[List[int]]:
        points.sort(key=lambda p: p[0] * p[0] + p[1] * p[1])
        return points[:k]


Time is `O(n log n)` because you sorted all `n` points to find `k` of them. Space depends on the sort implementation; Python's Timsort allocates a temporary buffer in the worst case, so `O(n)`.

Is that acceptable? For `n ≤ 10^4`, yes, comfortably. This is not a problem where the sort-based answer times out. If you write this in an interview and stop there, you've written a correct solution.

What you've also done is more work than the problem asks for. You computed a full ordering of all `n` points when you only needed the smallest `k` of them. When `k` is small relative to `n`, somewhere between 99% of your effort was wasted.

## Approach 2: keep only k points, in a max-heap

The idea: walk through the points once, maintaining a container that holds the `k` closest points seen so far, and nothing else. When a new point is closer than the worst point in that container, it replaces it.

The container needs exactly one fast operation: "give me the worst thing you're holding." That's a max-heap rooted on squared distance.

Here's the part people trip over. To find the k *closest*, you use a *max*-heap, not a min-heap. The root of a max-heap holds the farthest of the `k` points you're keeping, and that root is your eviction bar. A new point only earns a place if it's closer than that bar. A min-heap would put the closest point at the root, which tells you nothing about who to kick out.

Trace it on `points = [[3,3],[5,-1],[-2,4]], k = 2`, using squared distances 18, 26, and 20:

- Push `(3,3)` with `d² = 18`. Heap: one item. Root is `(3,3)`.
- Push `(5,-1)` with `d² = 26`. Heap has `k` items now. Root is the farther of the two, `(5,-1)`.
- `(-2,4)` has `d² = 20`. That's less than the root's 26, so it evicts `(5,-1)`. Heap now holds `(3,3)` and `(-2,4)`.

That's your answer, and you never looked at the sorted order of anything.

Python only ships a min-heap, so store the negated distance. A tuple of `(-d², x, y)` sorts so the largest `d²` has the smallest first element, which puts it at the root of `heapq`'s min-heap. The negation is doing exactly one job: turning your min-heap into a max-heap.

python
import heapq
from typing import List

class Solution:
    def kClosest(self, points: List[List[int]], k: int) -> List[List[int]]:
        heap = []  # size-k max-heap, simulated by negating the key
        for x, y in points:
            d2 = x * x + y * y
            if len(heap) < k:
                heapq.heappush(heap, (-d2, x, y))
            else:
                heapq.heappushpop(heap, (-d2, x, y))
        return [[x, y] for _, x, y in heap]


A few details worth knowing:

- `heappushpop` pushes then pops the smallest element, in a single `O(log k)` pass. It's cheaper than a separate push followed by a conditional pop, and it keeps the heap at exactly `k` items. When the new point is farther than everything in the heap, `heappushpop` simply discards the new point, which is the behavior you want.
- The tuple comparison falls back to `x` and then `y` when two points have identical squared distances. Since the answer is order-independent, that tie-breaking is harmless. Don't overthink it.
- The heap comes out in heap order, not sorted order. You don't need to sort it. If the interviewer wants the points sorted by distance, that's an extra `O(k log k)` on top, which is still cheap.

Complexity: `O(n log k)` time and `O(k)` space. Each of the `n` points triggers at most one heap operation on a heap of size `k`. When `k` is small, `log k` is small, and you're close to linear. When `k` approaches `n`, this degrades to `O(n log n)` and the sort-based solution is just as good with less code.

Here's the same logic in Java, where you don't have to negate anything because `PriorityQueue` takes a comparator:

java
class Solution {
    public int[][] kClosest(int[][] points, int k) {
        PriorityQueue<int[]> maxHeap = new PriorityQueue<>(
            (a, b) -> (b[0] * b[0] + b[1] * b[1]) - (a[0] * a[0] + a[1] * a[1])
        );
        for (int[] p : points) {
            maxHeap.offer(p);
            if (maxHeap.size() > k) maxHeap.poll();
        }
        int[][] res = new int[k][];
        for (int i = 0; i < k; i++) res[i] = maxHeap.poll();
        return res;
    }
}


The subtracting comparator works because with the given bounds the difference never exceeds `2 × 10^8`, which fits in a 32-bit `int`. Widen the bounds and you'd need `long`, or a comparator using `Long.compare` on squared distances.

## Approach 3: quickselect, if you want the average-case win

There's a third answer that's worth one sentence in an interview and rarely worth implementing unless asked: quickselect.

You partition the array around a pivot by squared distance, exactly like quicksort, then recurse only into the side that contains index `k`. You discard the other half entirely. Average time is `O(n)`, because the work shrinks geometrically instead of being repeated `log n` times. Worst case is `O(n²)`, when pivot choices keep splitting off one element at a time.

The tradeoff is honest and easy to state: quickselect has the better average asymptotic, and the heap has the better guarantee plus no risk of blowing up on adversarial input. Interviewers tend to accept either, but they will ask you what happens in the worst case, so know the answer before you offer the approach.

Python's `heapq.nsmallest(k, points, key=...)` is worth mentioning as a one-liner, and worth noting that it internally does the same `O(n log k)` heap work you just wrote by hand.

## Picking between them

| Approach | Time | Extra space | Reason to choose it |
| --- | --- | --- | --- |
| Sort by squared distance | `O(n log n)` | `O(n)` sort buffer | Small `n`, clearest code, least to get wrong |
| Size-k max-heap | `O(n log k)` | `O(k)` | `k` much smaller than `n`, or input arrives as a stream |
| Quickselect | `O(n)` average, `O(n²)` worst | `O(1)` with in-place partitioning | You want the best average speed and accept the worst case |

If you only remember one row, remember the second one. It's the answer the problem is shaped to elicit, and it's the version that survives every follow-up.

## The follow-ups that actually decide the interview

Getting the heap out is half the round. The other half is what happens next.

**"k equals n."** The heap fills to capacity and never evicts anything, so the answer is every point. Nothing special in the code, but say it out loud so the interviewer knows you checked.

**"The points arrive one at a time and you can't store them all."** This is the question the heap was built for. A streaming solution keeps `O(k)` memory and answers at any moment, because the heap always describes the best `k` points seen so far. The sort-based solution can't do this at all, and quickselect needs random access to the whole array. Having this answer ready turns a follow-up into a score.

**"What if the coordinates were up to `10^9`?"** Squared distance is `2 × 10^18`, which overflows a 32-bit integer and is uncomfortably close to the edge of a signed 64-bit one at `9.2 × 10^18` for a single point but fine per point. In Java or C++ this means `long` or `int64_t`. In Python, integers are arbitrary precision and you have nothing to do. Mentioning the overflow boundary unprompted is worth more than a clever implementation.

**"Can you break ties?"** You can't, and you don't need to. The problem guarantees a unique answer except for ordering, so a tie in distance that affects the result can't happen. Don't burn interview minutes on a tie-breaking policy.

**"Return the points, not the distances."** Obvious in the problem statement, easy to fumble halfway through coding under pressure. Re-read the return type before you finish.

**"Why is the root of your heap the farthest point?"** Because you negated the distance to convert Python's min-heap into a max-heap. Say that sentence. Don't leave the interviewer guessing whether it was a deliberate move or a lucky one.

## This isn't one problem, it's the top-k pattern

The reason this question shows up constantly is that it's the cleanest instance of a family. Any prompt with the words *k largest*, *k smallest*, *k closest*, *k most frequent*, or *top k* is asking for the same machinery, and the deciding question is whether you need all of them sorted or just the boundary.

Problems built on the identical skeleton:

- **Kth Largest Element in an Array** (LeetCode 215) — a size-k min-heap, or quickselect.
- **Top K Frequent Elements** (347) — same heap, with a count key from a hash map.
- **Kth Largest Element in a Stream** (703) — the streaming version, which is exactly why the heap approach matters.
- **Find K Pairs with Smallest Sums** (373) — heap over pairs, not scalars.
- **Kth Smallest Element in a BST** (230) — in-order traversal or a heap, depending on the follow-ups.
- **Sort Characters by Frequency** (451) — bucket sort or the same top-k machinery.

Learn one of these properly and the others become variations on where the key comes from and whether the input is static or streaming. That's the actual return on studying patterns instead of individual solutions: the second problem costs you a third of the time.

## Where a pattern library fits into this

If the heap version above felt like a set of tricks you memorized, the fix isn't doing 200 more problems. It's seeing the top-k family laid out together once, in order, with the decision logic spelled out: does the input fit in memory, do you need the k best or the kth best, is `k` far smaller than `n`.

[AlgoMonster](https://algo.monster?fpr=li69) organizes its material that way. It's an interview prep platform built by ex-Google engineers around roughly 48 named patterns, each with a conceptual explanation, a reusable code template, and practice problems attached to it. Heap and priority queue is one of those patterns, which is how K Closest Points, Kth Largest Element, and Kth Largest in a Stream end up in the same chapter instead of scattered across a problem list. There's a step-by-step breakdown of the problem on 👉 [AlgoMonster's walkthrough of K Closest Points to Origin](https://algo.monster/liteproblems/973?fpr=li69) that goes through the sort-based version with the reasoning traced out.

Two things to know about the format before you buy anything. It's entirely text and diagrams. There is no video. The platform leans on interactive illustrations and flowcharts instead, with a decision flowchart for "which pattern does this problem need" that's meant to be the thing you keep in your head during a real screen. Reviews consistently describe the curriculum as designed to be worked through in about 4 to 8 weeks at an hour or two a day, with a Speedrun mode for timed review and an AI assistant that explains concepts and debugs submitted code.

Two limitations worth knowing, because they come up in every serious write-up: system design coverage is introductory, not something you'd rely on for a senior-level loop, and there's no live mock interview or human feedback. Engineers targeting L5+ rounds usually pair it with a dedicated system design resource and a person who'll watch them code.

## AlgoMonster plans and current pricing

AlgoMonster runs promotions more or less continuously, and the pricing you see in reviews lags behind the checkout page. The prices below are the promotional numbers currently displayed on AlgoMonster's subscribe page, alongside the list prices crossed out next to them. Treat the checkout page as the source of truth, because these move.

| Plan | What you get | Price | Billing | Get access |
| --- | --- | --- | --- | --- |
| Free | Sample patterns, lessons, and problems to test the teaching style | $0 | No billing | [Start on the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro Yearly | Every pattern, lesson, and problem; unlimited AI assistant; unlimited Speedrun; real questions from top tech companies updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps; 12 months of access | $99/year, about $8.25/month billed annually (list price $198) | Billed annually, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in the annual plan plus lifetime updates on all current and future content | $189 (list price $378) | One-time payment | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

A couple of notes that saved me from a misleading table. AlgoMonster doesn't currently advertise a monthly plan on that page, and multiple 2026 reviews state plainly that monthly billing isn't offered; an `$18.99/month` tier has appeared on older versions of the page, so if a monthly option is what you want, confirm it at checkout rather than trusting a screenshot. The subscription also bundles two extra courses at no additional cost, a Foundation Course covering data structures and algorithms from the ground up, and a Build an LLM course that goes into tokenization, attention, and training rather than stopping at API calls. Some partner-developed courses, such as Quant Interview Prep, are sold separately.

The math that matters for this article: if you're using it to fix a specific gap like "I don't reliably recognize when a heap is the answer," a year at $99 is a small number next to the cost of failing a screen. If you expect to interview repeatedly over the next several years, the one-time $189 pays for itself faster than two annual renewals.

## What to actually say in the interview

Here's the narration that turns this problem from a coin flip into a routine.

Restate the problem in one sentence: return the `k` points with the smallest Euclidean distance to the origin, any order. Then immediately say that you'll compare squared distances instead of distances, because the square root preserves ordering. That one sentence buys credibility for the rest of the round.

State the brute force before you write anything: sort all points by squared distance and take the first `k`, at `O(n log n)`. Then improve it: keep a size-`k` max-heap of the closest points seen so far, evicting the root whenever a closer point arrives, for `O(n log k)` time and `O(k)` extra space.

Then code it. Then walk through the second example out loud, tracking the heap contents and the root after each insertion. Then name the follow-ups yourself: streaming input favors the heap, `k = n` needs no special handling, huge coordinates force 64-bit arithmetic, and quickselect has a better average with a worse guarantee.

That sequence covers the pattern, the reasoning, and the failure modes. The four-line sort becomes what it should be: the baseline you improved on, not the answer you settled for.
