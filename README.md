# Combination Sum II: Why Your Backtracking Returns Duplicates, and the Sort-and-Skip Fix (LeetCode 40)

A straight backtracking search solves Combination Sum II on the first example and embarrasses itself on the second. The code walks the array, builds up sums, hits the target, records the answer, backtracks. Everything looks right. Then the input contains `[2,5,2,1,2]` and the result comes back with the same combination twice, because the search picked the *first* `2` on one branch and the *second* `2` on another. Both are the same combination of values.

That's the entire problem. Not the recursion, not the pruning, not the complexity argument. Duplicate values in the input.

This piece walks through LeetCode 40 from the angle that actually trips people up: the difference between "used this index already" and "tried this value already at this level."

## What Combination Sum II actually asks

Given a list `candidates` that may contain repeated numbers, and a `target`, return every unique combination of values that sums to `target`. Three constraints do most of the work:

1. Each element may be used **once per combination** — position matters, so `candidates[i]` can't be reused on the next recursive call.
2. The input may contain duplicates, so `[1, 2, 7]` and `[2, 1, 7]` aren't different results — they're the same combination written differently.
3. The output must not contain duplicate combinations.

| Input | Output |
| --- | --- |
| `candidates = [10,1,2,7,6,1,5]`, `target = 8` | `[[1,1,6],[1,2,5],[1,7],[2,6]]` |
| `candidates = [2,5,2,1,2]`, `target = 5` | `[[1,2,2],[5]]` |

Note `[1,1,6]` in the first row. Two `1`s appear in the answer, which surprises people who assume "each number used once" means "each value used once." It means each *array position* is used at most once — and the input has two separate `1`s.

Constraints: `1 <= candidates.length <= 100`, `1 <= candidates[i] <= 50`, `1 <= target <= 30`. Difficulty is Medium, tagged Array and Backtracking. LeetCode's Chinese mirror reports roughly 703K accepted submissions out of 1.2M, which is a healthy acceptance rate for a problem whose only real trap is deduplication.

## Where the duplicate combinations come from

Take `candidates = [1, 2, 1, 7]` and `target = 8`. Before sorting, the recursion at the top level sees four children: index 0 (`1`), index 1 (`2`), index 2 (`1`), index 3 (`7`). Branches from index 0 and index 2 both start with the value `1`, and from that point forward the two subtrees contain identical value sets. You get `[1, 7]` twice.

The fix has two halves, and they address different things.

**Sorting** groups equal values next to each other so "have I seen this value before at this level?" becomes an adjacency check: `candidates[i] == candidates[i-1]`.

**Skipping** decides *when* to apply that check. And this is where the off-by-one bites.

python
if i > start and candidates[i] == candidates[i - 1]:
    continue


The condition `i > start`, not `i > 0`, is the whole trick. At the top of a recursion level, `start` is the first index this level is allowed to consider. If `candidates[i]` equals `candidates[i-1]` and `i` isn't the first index of this level, then the value at `i` was already offered as a choice at this same level — in an earlier loop iteration — and choosing it again would reproduce a subtree you've already explored.

Flip it to `i > 0` and `[1,1,6]` disappears from the first example. Those two `1`s sit at indices 1 and 2 in the sorted array `[1,1,2,5,6,7,10]`. When the search picks the `1` at index 1 and then loops to the `1` at index 2, `i > 0` is true and the values are equal, so the check throws the branch away. The subtlety: on that second call, the `1` at index 2 is the *first* candidate of the new level, and repeats of a value chosen at an *earlier* level are perfectly legal. That's the difference between a combination and a permutation, and it's the single most common bug in this problem.

## The solution

python
class Solution:
    def combinationSum2(self, candidates: list[int], target: int) -> list[list[int]]:
        candidates.sort()          # duplicates become adjacent
        res, path = [], []

        def backtrack(start: int, remaining: int) -> None:
            if remaining == 0:
                res.append(path[:])   # copy — never append the live list
                return
            for i in range(start, len(candidates)):
                # sorted, so everything after this index is >= candidates[i]
                if candidates[i] > remaining:
                    break
                # same value already tried at this level → identical subtree
                if i > start and candidates[i] == candidates[i - 1]:
                    continue
                path.append(candidates[i])
                backtrack(i + 1, remaining - candidates[i])  # i+1: one use per index
                path.pop()

        backtrack(0, target)
        return res


Three lines carry the entire problem.

- `candidates.sort()` — without it the adjacency check is meaningless.
- `continue` on the same-level repeat — kills duplicate combinations.
- `i + 1` in the recursive call — enforces "use each element once." Compare against LeetCode 39 (Combination Sum), where the same call is `i`, because there reuse is legal.

The `break` on `candidates[i] > remaining` is the pruning step, and it's only safe because the array is sorted: a value larger than what's left means every later value is larger too. On unsorted input a `break` would silently drop valid answers. It's an optimization, not a correctness requirement — the solution returns the right result without it, just slower.

### The alternative: a `used` array

Some editorials track a boolean array and check `used[i-1]` instead of comparing to `start`:

python
if i > 0 and candidates[i] == candidates[i - 1] and not used[i - 1]:
    continue


The logic is identical, restated as "the previous equal value hasn't been taken at this level." It costs one extra array and one extra bookkeeping step to maintain (`used[i] = True` before recursing, `False` after). The `i > start` version is easier to explain and easier to get right under interview pressure.

### Deriving it in an interview

If you're talking through this live, the productive sequence is: produce the naive recursion, show the duplicate output on a duplicated input, sort, notice that equal values are now adjacent siblings, decide to skip a sibling whose value matches its left neighbor. That narrative explains *why* the condition looks the way it does. Jumping straight to the final code and reciting "we skip duplicates" tends to invite a follow-up you can't answer.

## Complexity, and why the constraints matter

Sorting costs `O(n log n)`. The search itself is bounded by the number of distinct subsets of the array, so the worst case is `O(2^n)` time with recursion depth and temporary storage at `O(n)`, excluding the output list. AlgoMonster's editorial on this problem frames it the same way: pruning and duplicate-skipping cut the real number of recursive calls well below the bound, but the upper bound stays `O(2^n)`.

The stated constraints keep that bound harmless. With `target <= 30` and every candidate at least `1`, no valid path is longer than 30 elements, and the `remaining` pruning trims deep branches quickly. A 100-element input with a target of 30 is nowhere near the theoretical worst case.

## Combination Sum 39 vs 40 vs the rest

Most of the confusion around Combination Sum II is really confusion about its neighbors. Four problems share a skeleton and differ in two or three lines.

| Problem | Duplicates in input? | Reuse allowed? | Recursive call | Extra rule |
| --- | --- | --- | --- | --- |
| 39. Combination Sum | No | Yes | `dfs(i, ...)` | none |
| 40. Combination Sum II | Yes | No | `dfs(i + 1, ...)` | skip same value at same level |
| 216. Combination Sum III | N/A (digits 1–9) | No | `dfs(i + 1, ...)` | exactly `k` numbers, fixed range |
| 90. Subsets II | Yes | No | `dfs(i + 1, ...)` | same skip, but record every node |

Subsets II is the closest cousin and the best warm-up: same sort, same `i > start` skip, no target to hit — the result list collects the path at every level of the tree. If you can write Subsets II cleanly, Combination Sum II is that plus a `remaining` variable.

## Bugs worth checking before you submit

- **`i > 0` instead of `i > start`.** Loses `[1,1,6]`-style answers. The most common single-character failure in this problem.
- **`i` instead of `i + 1`** in the recursive call. That's Combination Sum's rule, not this one, and it produces combinations that reuse an index.
- **Not sorting.** The skip check compares neighbors; unsorted input makes it compare nothing meaningful.
- **`res.append(path)`** instead of `path[:]`. Every entry in the output aliases the same list, and they all end up empty after the final pops.
- **Deduping with a set at the end.** It passes, but it generates every duplicate subtree first and then converts tuples back to lists. Sorting up front and skipping siblings is the same amount of code with less wasted work.
- **Breaking on `remaining` without sorting.** Silently drops valid combinations instead of erroring.

The output order between combinations doesn't matter on LeetCode; within a combination, sorting the input means every answer comes out ascending, which is the convention most judges and interviewers expect.

## Practicing this pattern instead of re-reading this page

The reason Combination Sum II feels slippery is that it isn't one problem, it's one parameter change in a family of about six. Recognition is the skill being tested: see repeated values in an input array, reach for sort-plus-skip; see "each element once," reach for `i + 1`; see "any number of times," reach for `i`.

AlgoMonster organizes its curriculum around exactly that — pattern templates with decision flowcharts pointing at which pattern a problem belongs to, rather than a flat problem list. Its backtracking lessons cover the sibling-skip condition and the branch-pruning logic covered above, with diagrams, and the problems are drawn from real interview questions that the site says are updated monthly. The format is text and diagrams; there are no videos, which is either a feature or a dealbreaker depending on how you learn. Subscribers also get an AI assistant for explanations and a Speedrun mode for timed review sessions. Its own editorial for this problem is at the Liteproblems index under problem 40.

If you're grinding LeetCode 40, 39, 90, 216 and 77 in the same week, that's the point where a structured pattern reference starts saving time over scattered tabs. You can 👉 [work through AlgoMonster's backtracking pattern lessons here](https://algo.monster?fpr=li69) — the free tier exposes a slice of the content before you pay anything.

## What it costs

AlgoMonster's subscribe page lists two paid plans, plus a free tier with limited access to sample material. There's no monthly option, so the choice is annual versus one-time. Prices shown below are what the subscription page displayed at the time of writing; the site runs frequent discount periods, so checkout is the authority.

| Plan | What you get | Price | Billing period | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems | $0 | — | [start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; real company questions updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps | $8.25/mo, billed annually ($99/year); 3-day free trial | 12 months | [see the yearly plan](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates and, for qualified candidates, an internal referral to a top tech company | $189 | One-time | [compare the lifetime plan](https://algo.monster?fpr=li69) |

Two honest notes. First, the lifetime plan only pays off if you expect to interview again — a second job search within a couple of years makes the math lopsided in your favor, one interview next month does not. Second, third-party reviews are consistent about the gaps: no video walkthroughs, introductory-level system design, and no mock interviews. If your weak spot is performing under time pressure rather than recognizing patterns, this platform doesn't replace live practice.

## Questions that come up

**Can I skip the sorting and use a hash set to remove duplicates at the end?**
Yes, and it's a reasonable thing to say out loud in an interview before improving it. You generate every duplicate branch and pay for hashing each result. Sorting first costs `O(n log n)` and removes the duplicates before they're built.

**Does the order inside each combination matter?**
No. Any order is judged correct. Sorting the input makes the output deterministic and ascending, which is easier to eyeball when debugging.

**Why does `[1,1,6]` appear if each number is used only once?**
Because "once" refers to array indices, not values. `candidates` holds two separate `1` elements in that example, and a combination may include both.

**Is `O(2^n)` fast enough with up to 100 candidates?**
Yes, given the `target <= 30` bound and positive candidates. The pruning on `remaining` keeps the explored tree far smaller than the formal worst case.

**What's the difference between Combination Sum II and Subsets II?**
Subsets II has no target and records a copy of the path at every node of the recursion tree. The duplicate-skipping logic is the same line of code.

## The short version

Sort the array. Recurse with `i + 1`, because each index gets one use. Skip a candidate when it equals its left neighbor and isn't the first option at the current level. Prune when the remaining target goes below the current candidate. That's LeetCode 40.

Everything else — the `used` array variant, the set-based brute force, the complexity analysis — is scaffolding around those four lines. Get the `i > start` condition straight and the problem stops being about backtracking and starts being about typing.

If you want the pattern drilled properly with the surrounding family (39, 90, 216, 77) in one place, 👉 [AlgoMonster's pattern-based course](https://algo.monster?fpr=li69) puts all of them under one backtracking template with flowcharts for picking the right approach.
