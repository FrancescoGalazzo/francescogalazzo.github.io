---
layout: post
title: "Workshop Selection Using Dynamic Programming in Java"
date: 2026-11-18
categories: [Java, algorithm]
tags: [java, algorithms, dynamic-programming, greedy, project]
featured: false
related_posts: false
---

## Introduction

Every conference poses the same hidden optimization problem: the schedule is full of attractive workshops, several of them overlap, and you can only be in one room at a time. Each session has a different appeal — some are must-see, some are filler — so the goal is not to attend the *most* sessions, but to walk away with the *most total value*.

This post walks through a complete Java console application I built for my algorithms course that solves exactly this problem. It is the classic **weighted interval scheduling** problem, and the solution is a textbook application of **dynamic programming**: sort by finish time, compute predecessors, fill a DP table, and reconstruct the optimal schedule. Along the way we will see why the obvious greedy strategy fails, why the recurrence is correct, how the schedule is reconstructed from the DP table, and how the implementation was verified against brute force and benchmarked up to 5,000 workshops.

***

## The Problem

The input is a CSV file of workshops. Each workshop has an id, a fixed start and finish time (stored as minutes after midnight, so 09:00 is 540), and a preference value:

```text
id,start,finish,value
A,540,600,4
B,600,660,4
C,540,660,10
```

The task: select a subset of workshops such that **no two selected workshops overlap** and the **sum of values is maximized**. Two simplifying assumptions make the model clean: back-to-back sessions are allowed (a workshop ending at 10:00 and one starting at 10:00 do not conflict — zero travel time), and values are nonnegative and additive.

Why not just try every subset? Because there are `2^n` of them. Twenty workshops already means a million combinations; a thousand workshops means a number with about 300 digits. We need something smarter.

***

## Why Greedy Is Not Enough

The natural first instinct is a greedy rule: *always take the workshop that finishes earliest*. Intuitively it makes sense — finishing early keeps the rest of the day open. And in fact, if every workshop had the same value, this rule is provably optimal: it maximizes the *count* of sessions you can attend.

But our objective is total **value**, not count, and that changes everything. Look at the three workshops above:

| Workshop | Time       | Value |
|:---------|:-----------|------:|
| A        | 09:00–10:00 | 4    |
| B        | 10:00–11:00 | 4    |
| C        | 09:00–11:00 | 10   |

The greedy rule takes A first (it finishes earliest), then B (compatible with A), and skips C because it overlaps both. Total: **8**. But the optimal answer is C alone: **10**. The greedy choice of A — locally sensible, finishes first — irrevocably blocks the single most valuable workshop. No simple greedy rule can repair this, because a locally good choice can foreclose the global optimum. This is precisely the situation where dynamic programming earns its keep: instead of committing to one branch, it evaluates both options (take it or skip it) for every workshop, reusing subproblem solutions so the exploration never explodes.

***

## The Algorithm

The algorithm has four steps. Everything below refers to workshops sorted by finish time, with 1-based indices.

### Step 1 — Sort by finish time

Sort the workshops by finish time (breaking ties by id so results are reproducible). This ordering is the keystone: after sorting, all workshops *compatible* with workshop j — those that finish before j starts — form a contiguous prefix at the front of the list. That contiguity is what makes the recurrence possible.

### Step 2 — The predecessor array p[j]

For each workshop j, define:

```
p[j] = the largest i < j such that finish[i] <= start[j] (or 0 if none)
```

In plain words: *if I attend workshop j, what is the latest workshop I could have attended just before it?* Because the list is sorted by finish time, scanning backwards from j−1 and stopping at the first compatible workshop automatically yields the largest such i.

The payoff of this definition is subtle and elegant: since compatible workshops form a prefix, "the best I can do with anything compatible with j" is exactly "the best I can do with the first p[j] workshops" — which is a subproblem the DP table already solves. p[j] is the bridge that connects the "include j" case to a solved subproblem.

### Step 3 — The DP table

Let OPT[j] denote the maximum total value achievable using only the first j workshops. The base case is OPT[0] = 0 (the empty schedule). For j ≥ 1, every schedule for the first j workshops either contains workshop j or it does not:

```
OPT[j] = max( OPT[j-1],  v[j] + OPT[p[j]] )
```

- **Exclude j:** the schedule lives entirely in the first j−1 workshops, worth at most OPT[j−1].
- **Include j:** we gain v[j], and everything else must finish before j starts — so the rest is a schedule over the prefix ending at p[j], worth at most OPT[p[j]].

Why is taking the maximum of these two cases correct? By induction. When we compute OPT[j], both OPT[j−1] and OPT[p[j]] refer to smaller prefixes and are already optimal. Any optimal schedule falls into one of the two cases, so its value is bounded by the corresponding case — and both case values are achievable. Maximum of the two equals the optimum.

### Step 4 — Reconstructing the schedule

OPT[n] gives the best *value*, but not *which* workshops achieve it. During the fill we record a boolean take[j] (true when including j strictly beat skipping it), then walk backwards from j = n:

- if take[j]: workshop j belongs to the answer; jump to j = p[j] (everything between p[j] and j is incompatible with j anyway);
- otherwise: workshop j is not used; step to j − 1.

The collected workshops are reversed at the end. One subtlety worth emphasizing: the answer is this **single backtracking path**, not "every j with take[j] = true". The take[] array stores decisions from branches the final path never traverses, and blindly printing all of them produces a wrong schedule.

***

## A Worked Example

Here is a four-workshop example, small enough to trace by hand but rich enough to exercise every branch of the algorithm:

| ID | Start | Finish | Value |
|----|-------|--------|-------|
| A  | 09:00 | 10:00  | 6     |
| B  | 10:00 | 11:00  | 2     |
| C  | 10:00 | 12:00  | 8     |
| D  | 11:30 | 12:30  | 5     |

Sorted by finish time the order is A, B, C, D. Computing predecessors:

- **p[1] = 0** — nothing exists before A.
- **p[2] = 1** — A finishes at 10:00 exactly when B starts: compatible (adjacency allowed).
- **p[3] = 1** — B finishes at 11:00, after C starts at 10:00; A is the latest compatible.
- **p[4] = 2** — C finishes at 12:00, after D starts at 11:30; B is the latest compatible.

Now the DP table:

| j | ID | Select = v[j] + OPT[p[j]] | Skip = OPT[j−1] | take[j] | OPT[j] |
|---|----|----------------------------|-----------------|---------|--------|
| 1 | A  | 6 + OPT[0] = 6             | 0               | true    | 6      |
| 2 | B  | 2 + OPT[1] = 8             | 6               | true    | 8      |
| 3 | C  | 8 + OPT[1] = 14            | 8               | true    | 14     |
| 4 | D  | 5 + OPT[2] = 13            | 14              | **false** | **14** |

OPT = [0, 6, 8, 14, 14]. The best value is **14**, and notice that at j = 4 skipping D wins — the algorithm compared both branches instead of committing to one, exactly what the greedy cannot do.

Reconstruction walks backwards: j = 4, take is false → skip D, go to j = 3. j = 3, take is true → **C is selected**, jump to p[3] = 1. j = 1, take is true → **A is selected**, jump to p[1] = 0. Done. The optimal schedule is **A + C = 14**.

And here is the subtlety from the previous section made concrete: take[2] is *true* (including B beat skipping it when only the first two workshops were considered), yet B is not in the final answer — the winning path went through C, which excludes B. Following the single backtracking path, with its jumps via p[j], is what keeps the reconstruction honest. On this same input the greedy takes A + B + D for a total of 13 — one point of value lost to a rule that cannot look ahead.

***

## The Implementation

The project is deliberately split into seven small files, one responsibility each — which also makes each piece explainable in isolation:

| File | Responsibility |
|------|----------------|
| `Workshop.java` | The data model: id, start, finish, value, and the overlap test |
| `WorkshopCsv.java` | Read and validate the CSV; every bad row fails loudly with a line number |
| `WeightedIntervalScheduler.java` | The algorithm: sort, predecessors, DP table, reconstruction, trace mode |
| `GreedyBaseline.java` | The earliest-finish greedy, for comparison |
| `ExhaustiveSearch.java` | Brute force over all `2^n` subsets — the ground truth oracle (n ≤ 20) |
| `ScheduleGenerator.java` | Seeded, reproducible synthetic schedules |
| `Main.java` | The CLI: solve, verify, selftest, benchmark, generate |

The core of the solver is short. The predecessor computation is a backward scan:

```java
private static int[] computePredecessors(List<Workshop> sorted) {
    int n = sorted.size();
    int[] p = new int[n + 1];                 // p[0] = 0 sentinel

    for (int j = 1; j <= n; j++) {
        int startJ = sorted.get(j - 1).start;

        // Scan backwards from j-1 down to 1.
        for (int i = j - 1; i >= 1; i--) {
            if (sorted.get(i - 1).finish <= startJ) {
                p[j] = i;                     // largest compatible i
                break;                        // stop at the FIRST hit
            }
        }
    }
    return p;
}
```

And the DP fill is a direct translation of the recurrence:

```java
for (int j = 1; j <= n; j++) {
    Workshop w = sorted.get(j - 1);

    int include = w.value + opt[p[j]];   // case 2: attend workshop j
    int exclude = opt[j - 1];            // case 1: skip workshop j

    take[j] = include > exclude;         // strict '>' per the pseudocode
    opt[j] = Math.max(include, exclude);
}
```

A `--trace` flag prints the full DP table (j, p[j], skip, select, OPT[j]) so the walkthrough above can be reproduced live with:

```bash
java -cp out Main solve data/example_walkthrough.csv --trace
```

***

## Verification: Trust, but Check

How do you know the DP is actually correct, and not just plausible? Three layers:

1. **Brute-force oracle.** For small inputs (n ≤ 20) the program enumerates all `2^n` subsets, keeps the conflict-free ones, and returns the true optimum. On every tested instance — the edge-case CSVs and five seeded random instances — the DP value equals the exhaustive value. This is the strongest check available: not "the answer looks right" but "the answer is provably the best of everything".

2. **Structural checks.** For every solution the tests also verify that the selected schedule is genuinely conflict-free (pairwise overlap test) and that the reported value equals the sum of the selected values — the reconstruction cannot invent value.

3. **Edge cases.** Empty input, a single workshop, no overlaps at all, everything overlapping, ties in finish times, zero values, back-to-back sessions, and an invalid CSV row (start after finish) that must be rejected with a clear error. The selftest runs all of it in one command:

```bash
java -cp out Main selftest data
# SELFTEST RESULT: 14 passed, 0 failed
```

The synthetic generator deserves a mention here because of **reproducibility**: it uses an explicitly seeded `java.util.Random`, so the same parameters always regenerate the identical file. Every experiment below can be replayed by anyone, bit for bit.

***

## Benchmark Results

The benchmark generates instances with a fixed overlap parameter of 0.6 (start times compressed into roughly the first three hours of the day, so conflicts are frequent but the problem is not trivial), for sizes 100, 500, 1,000, and 5,000 workshops, with three fixed seeds each. Runtime is the median of five runs after a warm-up, to shield the measurement from JIT compilation and garbage-collection noise:

```text
n       seed   DP value   DP ms     Greedy ms  Greedy val  Gap
100     7      82         0.226     0.094      60          22
500     7      106        0.688     0.199      70          36
1000    7      113        0.668     0.228      57          56
5000    7      119        12.160    1.347      84          35
```

Two observations fall out of this table:

- **The value gap is real and systematic.** The DP beats the greedy on every seed and every size, often by 30–50% of the greedy's value. The "obvious" rule leaves a lot of preference points on the table.
- **The runtime grows quadratically, as the theory predicts.** From n = 500 to n = 5,000 (ten times the workshops) the DP time grows roughly thirty-fold — the signature of the O(n²) predecessor scan, which at n = 5,000 performs about 12.5 million cheap comparisons, still finishing in ~12 ms.

***

## Complexity

| Operation | Cost |
|-----------|------|
| Sort by finish time | O(n log n) |
| Compute all predecessors (backward scan) | O(n²) |
| Fill DP table + reconstruct schedule | O(n) |
| **Total time** | **O(n²)** |
| **Memory** | **O(n)** (the p, OPT, and take arrays) |

The dominant term is the predecessor scan. Because finish times are sorted, each p[j] query could instead be answered with a binary search over the finish array, bringing the total down to O(n log n). I kept the O(n²) scan deliberately: it matches the project proposal, it is trivially explainable line by line, and at these input sizes the difference is milliseconds. The comparison worth remembering is not O(n²) versus O(n log n) — it is O(n²) versus the O(2^n) of brute force, which is the difference between 12 milliseconds and a number of centuries no one should ever compute.

***

## Conclusion

Weighted interval scheduling is one of those problems where the journey is the lesson. The greedy rule fails for a reason that generalizes far beyond scheduling: local optimality is not global optimality when choices constrain the future. The fix — dynamic programming — works because sorting by finish time turns "everything compatible with j" into a contiguous prefix, p[j] turns that prefix into an address, and the recurrence OPT[j] = max(skip, take) reduces an exponential search to n comparisons of two already-solved subproblems.

The full source (seven Java files, ten sample CSVs, and a README with a class-demo script) is compact enough to read in one sitting — and if you want to watch the DP think, run the solver with `--trace` and follow the table row by row. It is a rare pleasure in algorithms: an optimal answer you can watch being built, one decision at a time.
