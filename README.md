# 0 1 knapsack problem: build the DP table, cut memory to one array, and spot the subset-sum variants in interviews

Most people meet this problem in one of three ways: a greedy ratio solution that looks reasonable and is wrong, a brute force over 2^n subsets that times out, or a recursion that works on the sample input and dies on the third test case. The 0 1 knapsack problem is usually where dynamic programming stops being a buzzword and starts being an actual tool.

It's also one of the most heavily reused ideas in coding interviews. The classic "maximize value under a capacity" framing shows up directly, but the same recurrence is hiding inside subset-sum, target-sum, last-stone-weight, and coin-problem questions. Learn it once, properly, and you get five or six later problems for free.

## What the "0/1" in the name actually means

You're given `n` items. Item `i` has a weight `w[i]` and a value `v[i]`. You have a knapsack that can carry at most `W` units of weight. Pick a subset of items so the total weight doesn't exceed `W` and the total value is as large as possible.

The "0/1" refers to the decision variable for each item: it's either in the bag (1) or out (0). No fractions. You can't cut an item in half, and you can't take the same item twice. That single constraint is what makes the problem hard.

Why hard? Because brute force means checking all 2^n subsets, and because the greedy shortcut doesn't work here. In the *fractional* knapsack — where you're allowed to take 40% of an item — sorting by value-to-weight ratio and taking greedily is optimal. Remove that freedom and greedy breaks immediately.

A small example of greedy failing:

| Item | Weight | Value | Value/weight |
| --- | --- | --- | --- |
| 1 | 1 | 3 | 3.0 |
| 2 | 2 | 5 | 2.5 |
| 3 | 3 | 6 | 2.0 |

Capacity `W = 4`. Greedy takes item 1 (weight 1, value 3), then item 2 (weight 2, value 5), then can't fit item 3 — total value 8, weight 3. The actual optimum is items 1 and 3: weight 4, value 9. Greedy picked the right first item and the wrong second one, and that's the whole problem in miniature.

## The take-or-skip recurrence

The standard fix is to define a state with two dimensions:

`dp[i][w]` = the maximum value you can get using only the first `i` items, with a capacity of `w`.

For each item you consider, exactly two things can happen:

- **Skip it.** The answer is whatever the first `i-1` items could do with the same capacity: `dp[i-1][w]`.
- **Take it** (only if `w[i] <= w`). You gain `v[i]` and lose `w[i]` units of capacity: `v[i] + dp[i-1][w - w[i]]`.

So:

text
dp[i][w] = dp[i-1][w]                                        if w[i] > w
dp[i][w] = max(dp[i-1][w], v[i] + dp[i-1][w - w[i]])         otherwise
dp[0][w] = 0   for all w
dp[i][0] = 0   for all i


Two things worth noticing before you write any code. First, the "take" branch reads row `i-1`, not row `i` — that's the mechanism that prevents reusing an item. Second, the answer lives at `dp[n][W]`, the bottom-right corner.

One definitional trap: don't define the subproblem as "the best subset of the first `i` items" alone. That version fails because the optimal solution for `i` items with capacity `W` doesn't necessarily extend the optimal solution for `i-1` items — the better global answer may need a *different* selection at a smaller remaining capacity. Capacity has to be part of the state.

## Filling the table: four items, one capacity

Take items `(weight, value)` of `(1,1)`, `(3,4)`, `(4,5)`, `(5,7)` with `W = 7`. The optimum is items 2 and 3: weight 7, value 9. Note that item 4 has the highest individual value and still gets left out.

Here's a 1D array (indexed by capacity, `0` through `7`) traced item by item, which is how you'd actually run it in code:

text
start:                    [0, 0, 0, 0, 0, 0, 0, 0]
after item (w=1, v=1):    [0, 1, 1, 1, 1, 1, 1, 1]
after item (w=3, v=4):    [0, 1, 1, 4, 5, 5, 5, 5]
after item (w=4, v=5):    [0, 1, 1, 4, 5, 6, 6, 9]
after item (w=5, v=7):    [0, 1, 1, 4, 5, 7, 8, 9]


`dp[7] = 9` at the end. Every cell was computed exactly once and each cell only ever depended on values from before the current item was processed. That last property is what lets you delete the first dimension.

## Shrinking O(n·W) space to one array

Instead of a 2D table, keep a single array `dp` of length `W+1` where `dp[w]` holds the best value for capacity `w` using the items processed so far. The update for each item is:

text
for w from W down to w[i]:
    dp[w] = max(dp[w], v[i] + dp[w - w[i]])


The loop runs **backward**, and this is the single most common trip-up. When you compute `dp[w]`, the value you read from `dp[w - w[i]]` must still be the result from *before* this item was considered. Going right-to-left guarantees that, because the lower indices haven't been overwritten yet in this pass.

Go left-to-right and `dp[w - w[i]]` may already include the current item, so you'd be adding item `i` more than once — which silently turns a 0/1 knapsack into an unbounded knapsack. Same code, one reversed range, different problem.

python
def knapsack(weights, values, W):
    dp = [0] * (W + 1)
    for wi, vi in zip(weights, values):
        for w in range(W, wi - 1, -1):   # backward: each item used at most once
            dp[w] = max(dp[w], vi + dp[w - wi])
    return dp[W]


Time complexity is **O(n·W)**, space is **O(W)**. The trade-off: with the 1D array you lose the ability to walk back through the table and reconstruct *which* items were chosen. If the question asks for the actual subset, keep the 2D table or store a parent pointer.

Also worth stating out loud in an interview: `O(n·W)` is **pseudo-polynomial**. `W` is part of the input's numeric value, not its length, so the runtime grows with the *size* of `W`, not the number of digits used to write it. Knapsack is NP-hard in general; the DP is efficient only when the capacity is modest — which is exactly why interview constraints usually say something like `1 <= capacity <= 1000` or `<= 10^4`.

## Where the 0/1 recurrence hides in other problems

Most knapsack questions you'll get aren't retail-flavored. They're the same recurrence wearing different clothes, and the thing that changes is what the DP cell stores.

| Variant | Item reuse | Inner loop | Cell holds | Typical problems |
| --- | --- | --- | --- | --- |
| Subset-sum reachability | once | backward | boolean | Partition Equal Subset Sum (LC 416), Last Stone Weight II (LC 1049) |
| Count subsets / ways | once | backward | count, `dp[0] = 1` | Target Sum (LC 494) |
| Unbounded, min items | unlimited | forward | min, `dp[0] = 0` | Coin Change (LC 322), Perfect Squares (LC 279) |
| Unbounded, count combinations | unlimited | forward, items outer | count | Coin Change II (LC 518) |
| Unbounded, count permutations | unlimited | forward, amount outer | count | Combination Sum IV (LC 377) |
| Two-dimensional capacity | once | backward on both dims | max | Ones and Zeroes (LC 474) |

The row that catches people out most often is the loop *nesting* for counting problems. If each combination should be counted once — `{1,2}` and `{2,1}` being the same thing — items go in the outer loop and the target in the inner loop. If order matters and they're distinct, swap them. The array is identical; only the nesting changes.

## Two reductions worth memorizing

**Partition Equal Subset Sum (LC 416).** Split the array into two subsets with equal sums. Total the array; if it's odd, return false immediately. Otherwise the target is `total / 2`, and the question becomes "does any subset sum to exactly half?" That's boolean 0/1 knapsack with `dp[0] = True` and `dp[s] = dp[s] or dp[s - x]` running backward.

**Target Sum (LC 494).** Assign `+` or `-` to each number so the result equals `target`. Let `P` be the set with plus signs and `N` the ones with minus signs. Then `sum(P) - sum(N) = target`, and since `sum(P) + sum(N) = total`, you get `sum(P) = (target + total) / 2`. So it's not a DP over signs at all — it's a count of subsets summing to one specific value, with `dp[0] = 1` (one way to form sum 0: pick nothing) and `dp[s] += dp[s - num]` running backward.

If `(target + total)` is odd or negative, the answer is 0. Catching that early saves you a table allocation and a couple of loops, and interviewers notice.

## Complete, bounded, and mixed knapsack

Once the basic version clicks, the follow-ups are mostly about how many times an item can be used:

- **Complete (unbounded) knapsack** — unlimited copies of each item. Identical recurrence, inner loop runs forward.
- **Bounded (multiple) knapsack** — item `i` can be taken up to `k[i]` times. The naive approach adds a third loop over `k`, which costs O(n·W²) unless you handle the counts cleverly.
- **Mixed knapsack** — some items takeable once, some `k` times, some unlimited. All three are processed with their respective loop directions.
- **Binary decomposition** — a bounded item that can be taken up to `k` times is equivalent to 0/1 items with multiplicities `1, 2, 4, ...`. Splitting `k` this way turns into roughly `log k` items, and then the plain 0/1 code handles it.

That decomposition trick is genuinely useful and worth having in your notes: "unbounded or bounded" problems can often be pushed back into the 0/1 shape rather than requiring new code.

## How to recognize the pattern fast

The signals that should make you stop and reach for a DP array:

- You're choosing a subset from a list, and each element is used at most once.
- There's a budget, capacity, target sum, or weight limit acting as the constraint.
- The objective is maximize, minimize, count, or a yes/no reachability question.
- Constraints are small in *value* (sums up to a few thousand) but not in count — that gap is the tell that a pseudo-polynomial DP is intended.
- Someone says "either take it or don't" about every element.

And the signals that you've gone the wrong way: ratios, sorting by value density, or a recursion with no repeated state.

## Mistakes that cost people the interview

- **Forward inner loop on a 0/1 problem.** Turns it into unbounded knapsack. Check the direction every single time.
- **Sizing the array by item count.** The array is indexed by capacity or target sum, not by index in the input list.
- **Wrong nesting for counting problems.** Combinations and permutations differ only in whether items or the target sits in the outer loop.
- **Missing the `dp[0]` base case.** `dp[0] = 0` for maximization, `dp[0] = True` for reachability, `dp[0] = 1` for counting. Getting this wrong produces answers that are off by one in ways that are hard to debug live.
- **Forgetting the feasibility check.** Odd sums for partition problems, targets larger than the total, negative remaining capacity — handle them before the loop.
- **Overflowing.** Values summing past 2^31 for large `n` is a real risk in languages without arbitrary-precision integers.
- **Not stating the complexity.** `O(n·W)` time and `O(W)` space, plus the pseudo-polynomial caveat. Interviewers ask about that almost every time, and "it's just O(nW)" without the caveat reads as memorization.

## Practicing the pattern instead of memorizing one solution

Solving 0/1 knapsack once and moving on is why the same problem resurfaces three weeks later in interview form and looks unfamiliar. The useful mental model isn't "here's the knapsack code" — it's "here are the six questions I ask about any budget-constrained subset problem, and here's the loop direction each answer implies."

That's a pattern-recognition skill, which means it needs repetition across variants rather than one perfectly understood example. AlgoMonster approaches interview prep exactly this way: it organizes its curriculum around reusable patterns with decision flowcharts for "which technique does this problem need?", and its dynamic programming track ends in a dedicated knapsack group — nine problems in the knapsack section specifically, sitting after the prefix, grid, two-sequence, interval, and non-constant-transition groups, so you arrive at it already fluent in DP state design. If you want to see how the progression is laid out before paying anything, the free tier covers a subset of the content: 👉 [try the free sample lessons first](https://algo.monster?fpr=li69).

For a topic like this, what matters more than the problem list is the explanation quality. AlgoMonster's DP material is text-and-diagram based, with editorials and illustrations rather than video, plus an AI assistant and a Speedrun mode for timed practice. The knapsack intro editorial covers the 0/1 and unbounded cases side by side, which is precisely where the forward-versus-backward loop distinction gets made concrete. 👉 [see the knapsack lessons and the rest of the DP track](https://algo.monster?fpr=li69)

### Current plans

AlgoMonster's checkout lists three options. Prices are promotional and move around, so treat the numbers below as the current range rather than a permanent list price — the checkout page is the only source worth trusting when they differ.

| Plan | What's included | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Sample lessons and a subset of patterns and problems — enough to judge whether the text-plus-diagram teaching style works for you | $0 | — | [Start on the free plan](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons, and problems; unlimited AI assistant; unlimited Speedrun; real questions from top tech companies updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps; Foundation and Build-an-LLM courses included | Listed at $198/year, currently shown at $99/year ($8.25/mo billed annually); some promo links land on a $69.99/year variant | 12 months of full access, renews annually | [Compare the yearly plan](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan, plus lifetime updates and all future content additions | Listed at $378, currently shown at $189 one-time; some promo links land on an $89 variant | One-time payment | [Check the lifetime plan](https://algo.monster?fpr=li69) |

Note that Quant Interview Prep is a specialized course developed with external partners and is sold separately. There's no monthly plan in the checkout — it's free, yearly, or lifetime. Third-party review sites quote a much wider range for lifetime access (some list $459 and yearly prices from $119 to $299), which is a good reminder that these figures are promo-dependent and worth verifying at checkout rather than trusting a blog post from six months ago.

### Which one makes sense

If you're prepping for a specific interview cycle that ends in a few months, the yearly plan is the cheaper commitment and you'll have finished the DP track long before it expires. The lifetime plan makes sense if you expect to do this again — engineers who switch jobs every two or three years do tend to revisit patterns each time, and $189 once beats renewing twice. If you're only here because knapsack specifically is blocking you, the free tier plus a solid written explanation of the loop direction may be enough on its own.

The honest limitation: AlgoMonster is text and diagrams. If you learn algorithms by watching someone talk through a whiteboard, this format won't click, and the free tier will tell you that within twenty minutes. It's also built for self-directed individuals — there's no team or enterprise plan.

## Quick answers

**Is 0/1 knapsack actually asked in interviews?** The direct retail version shows up occasionally. The recurrence behind it shows up constantly, usually disguised as subset-sum, partition, target-sum, or a coin problem. Treat it as one pattern with many skins.

**Do I need to memorize the table?** No. You need to be able to derive the recurrence from "take it or skip it" on the spot, and to know that the inner loop runs backward for 0/1 and forward for unbounded.

**Why not just use recursion with memoization?** That works and costs the same asymptotically. Top-down fills only the reachable states and is easier to derive under pressure; bottom-up with a 1D array uses less memory and is easier to write quickly once the recurrence is clear. Interviewers are happy with either if you explain the state.

**What if capacity is huge, like 10^9?** Then the DP is out. You're into approximation territory — a 2-approximation exists that compares "the best greedy prefix" against "the single item with the highest value," which is worth knowing as a fallback answer when the interviewer moves the goalposts.
