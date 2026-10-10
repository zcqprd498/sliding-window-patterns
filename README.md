# sliding window problems: how to tell fixed from variable windows, write the shrink condition once, and work the questions that actually get asked

You can recite the template. `left`, `right`, a running sum, a `while` loop that shrinks. Then a new problem shows up, it says "longest substring with at most two distinct characters," and your hands go back to writing two nested loops.

The code was never the problem. Recognition is. Solving sliding window problems reliably comes down to three decisions you make before typing anything: is the window size given, is the condition monotonic, and does the shrink go in a `while` or an `if`. Get those three right and most of these problems collapse into the same fifteen lines.

Everything below is organized around those decisions, with the specific LeetCode questions where each one shows up.

## Two questions that decide the whole approach

Answer these before writing code.

**1. Is the window size given?** If the problem hands you a `k`, that's a fixed-size window. Compute the first window, then slide one step at a time, adding the element entering and subtracting the one leaving.

**2. If not: is the condition monotonic?** Meaning, if a window violates the condition, does every larger window containing it also violate it? If yes, you can expand and shrink with two pointers. If no, sliding window is the wrong tool and you should stop trying to force it.

Then filter by the wording. Two signals have to line up:

- The problem asks about a **contiguous** range: subarray or substring. "Subsequence" and "subset" are the exit sign. Those are not contiguous, and a moving window cannot represent them.
- The answer is a longest, shortest, maximum, minimum, or a count of valid windows. Sometimes the giveaways are phrased as constraints instead: "at most K," "flip at most K," "no more than two distinct."

|  | Fixed-size window | Variable-size window |
| --- | --- | --- |
| Window size | Given as `k` | Grows and shrinks by condition |
| Pointer movement | Both pointers advance together | `right` always advances, `left` advances only when the window breaks |
| Typical phrasing | "subarray of size k", "every window of length k" | "longest", "shortest", "at most k distinct" |
| Answer update | Once per slide | Inside or after the shrink loop, depending on whether you want min or max |
| Time | O(n) | O(n) |

## The case where the problem looks right and isn't

Sliding window problems get misapplied most often on sum-based questions with negative numbers.

Take **LC 560, Subarray Sum Equals K**. Contiguous subarray, asks for a count, checks every box on the recognition list. Sliding window still fails, and it fails silently. The whole argument for shrinking depends on the sum moving in one direction as the window grows. With negatives, adding an element can make the sum smaller, so "sum is too big, shrink" is no longer a safe rule. The standard solution is prefix sums with a hash map.

Compare **LC 209, Minimum Size Subarray Sum**. Same on the surface, and sliding window works perfectly, because the constraints guarantee positive values. Growing the window can only increase the sum, so the shrink condition stays honest.

This is worth internalizing: check the constraints for negatives before you commit. It costs five seconds and saves twenty minutes of debugging a solution that cannot work.

## Fixed-size windows: four lines and a running state

The fixed-size template is short enough to keep in your head:

python
def fixed_window(nums, k):
    window = sum(nums[:k])
    best = window
    for right in range(k, len(nums)):
        window += nums[right] - nums[right - k]   # add incoming, drop outgoing
        best = max(best, window)
    return best


The detail that matters is `nums[right - k]` rather than a separately tracked `left` pointer. The left edge of the window is defined by where the right edge is, so you do not need to maintain it. Fewer variables, fewer off-by-one mistakes.

Start with **LC 643, Maximum Average Subarray I**. It is the plainest possible version of this pattern, and if you can write it from memory you understand the mechanics.

Once a fixed window also needs frequency tracking, the shape changes slightly. **LC 567, Permutation in String** and **LC 438, Find All Anagrams in a String** both ask whether one string's character counts match a sliding window of a fixed length. You build a count for the target, maintain a count for the window, and track how many characters are currently satisfied. Every slide updates exactly two characters. That two-updates-per-step discipline is what keeps it O(n) instead of O(n·k).

## Variable-size windows: the shrink condition is the entire problem

With variable windows, the boilerplate stops mattering and two choices do all the work.

python
def variable_window(nums, target):
    left = 0
    state = 0                      # sum, count, frequency map
    best = float('inf')
    for right in range(len(nums)):
        state += nums[right]       # expand
        while state >= target:     # shrink while the window is valid
            best = min(best, right - left + 1)
            state -= nums[left]
            left += 1
    return 0 if best == float('inf') else best


**Decision one: `while` or `if`.**

If the answer you want is a *longest* window, you can usually replace the inner `while` with a single `if`. When the window becomes invalid, move `left` forward by exactly one instead of shrinking back to validity. The window's size then never decreases; it just slides. The answer at the end is the window length.

This works because you only ever need the size of the largest valid window seen so far. The window keeping its size is a record, not a claim that the whole window is valid. It is the standard trick for **LC 424, Longest Repeating Character Replacement**, **LC 1004, Max Consecutive Ones III**, and **LC 1493, Longest Subarray of 1's After Deleting One Element**. Stick with `while` when the answer depends on the actual contents rather than just the length.

**Decision two: where you update the answer.**

- Minimum-length problems: update **inside** the shrink loop, because that is the only place the window is valid. **LC 209** is the reference case.
- Maximum-length problems: update **after** the shrink loop exits, for the same reason.

Getting this backwards is the most common source of a solution that passes the examples and fails on the hidden tests. If your answer is off by one on a min-length problem, this is almost always why.

## Two variants that catch people who learned the basics and stopped

**"Exactly K" is not monotonic.** *LC 992, Subarrays with K Different Integers* asks for exactly K distinct integers. "At most K" grows and shrinks predictably. "Exactly K" does not, because a window can become invalid in both directions. The clean fix is to write a helper that counts subarrays with **at most** K distinct values, then subtract:


exactly(K) = atMost(K) - atMost(K - 1)


The same trick handles "exactly K odd numbers" style questions. Once you see it, a whole family of problems that looked like they needed something exotic becomes two calls to code you already wrote.

**Sliding window maximum needs a deque, not two pointers.** *LC 239, Sliding Window Maximum* asks for the max of every window of size k. The two-pointer approach gives you nothing here, because removing the leftmost element can invalidate the maximum and there is no way to recover the runner-up. The fix is a monotonic deque holding **indices** in decreasing order of value:

- Before inserting `right`, pop from the back while the back element's value is less than or equal to `nums[right]`. Those elements can never be the maximum of a future window.
- Pop from the front when the front index has fallen out of the window.
- The front of the deque is the maximum of the current window.

Every index enters and leaves the deque once, so it is O(n) with O(k) space. This one is worth writing out longhand once. It is a different mental model from the rest of the family, and interviewers like it precisely because it separates people who know the pattern from people who know the pattern's easiest version.

## Bugs worth checking for by name

These are not hypothetical. They are the specific things that turn a correct-looking submission into a wrong answer.

- **Updating `best` before verifying the window is valid.** Order of operations inside the loop is load-bearing.
- **Missing the early guard in LC 713** (`Subarray Product Less Than K`). If `k <= 1`, no subarray qualifies, and the shrink loop will otherwise run forever with a product of 1.
- **Comparing dictionary sizes in LC 76** (`Minimum Window Substring`). `len(window)` counts distinct characters present, not characters whose required count is satisfied. Track a separate `formed` counter that increments when a character's count hits its target exactly.
- **Decrementing a frequency map without deleting zero-count keys.** Then `len(map)` reports characters that are no longer in the window, and your window validity check lies.
- **Letting `left` overtake `right`.** The window becomes empty or negative and `right - left + 1` turns into nonsense. Usually this shows up as an answer of 0 or a negative length.

## A practice order that builds recognition instead of volume

Do these roughly in order. Each rung adds exactly one new idea.

1. **643, Maximum Average Subarray I** (Easy) — fixed window, nothing else
2. **219, Contains Duplicate II** (Easy) — fixed window plus a set
3. **567, Permutation in String** (Medium) — fixed window plus frequency comparison
4. **438, Find All Anagrams in a String** (Medium) — same skill, one more moving part
5. **209, Minimum Size Subarray Sum** (Medium) — first variable window, first `while`, min-length update placement
6. **3, Longest Substring Without Repeating Characters** (Medium) — variable window plus a set
7. **424, Longest Repeating Character Replacement** (Medium) — window plus a max-frequency state
8. **1004, Max Consecutive Ones III** (Medium) — same shape as 424 with simpler state
9. **1493, Longest Subarray of 1's After Deleting One Element** (Medium) — off-by-one trap
10. **713, Subarray Product Less Than K** (Medium) — counting valid windows instead of finding one
11. **340, Longest Substring with At Most K Distinct Characters** — the generic "at most K" version
12. **992, Subarrays with K Different Integers** (Hard) — the atMost subtraction
13. **76, Minimum Window Substring** (Hard) — the `formed` counter problem
14. **239, Sliding Window Maximum** (Hard) — the deque

That is roughly a week of focused work, and it covers the variant space. You do not need fifty sliding window problems. You need these fourteen and the ability to say which template each one is.

Sliding window appears in AlgoMonster's curated practice lists — Monster 50 and Blind 75 both include entries tagged as introducing, extending, and hardening the pattern, and those lists are **free to practice** once you create an account. 👉 [Practice the sliding window problems in AlgoMonster's curated lists](https://algo.monster?fpr=li69)

## Why this is O(n) when it looks like O(n²)

The nested loop makes people nervous. It shouldn't.

`right` advances exactly n times across the entire run. `left` only ever moves forward, and it can move at most n times total, because it never resets backwards and never exceeds `right`. So the inner loop's total work across all outer iterations is bounded by n, not by n per outer iteration. Every element enters the window once and leaves once. Total pointer movement: at most 2n.

Space is O(1) for sum-based problems, and O(k) or O(alphabet size) when you keep a frequency map. For lowercase English letters that is effectively O(1) too.

## Where a structured pattern course helps and where it doesn't

If you have read this far and recognized your own failure mode, the gap is usually pattern selection rather than implementation. That is the specific problem AlgoMonster is built around, and it is worth being precise about what it does and doesn't give you for sliding window problems.

**What it offers.** Lessons are organized by pattern, so sliding window is taught as a named thing with a template rather than a trick you infer from solutions. The site's pricing page currently lists 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, written by Google engineers and competitive programmers. The diagram-heavy format suits this particular topic, since watching `left` and `right` move is much easier to absorb visually than in prose.

**The feature aimed directly at the recognition problem.** Speedrun is a set of rapid drills that ask you to identify the technique rather than write the full solution, and it's included without limits on Pro. Identifying "this is a sliding window with a shrink while-loop" in three seconds is the actual interview skill. Writing the loop is the easy part.

**Honest limits.** Everything is text and diagrams. There are no video walkthroughs, so if you learn best by watching someone code, NeetCode or AlgoExpert will fit you better. The problem count is in the hundreds, not thousands, which is enough to learn the patterns but not enough for volume training before a Google loop. Most candidates who get value from it end up pairing it with LeetCode for reps. It is also scoped to DSA. System design coverage is introductory, and there is no behavioral prep.

## What AlgoMonster costs right now

The pricing page shows three tiers. Prices on subscription sites move around with promotions, so confirm what you see at checkout before paying.

| Plan | Price | Billing | What you get | Link |
| --- | --- | --- | --- | --- |
| Free | $0 | — | Limited access to a subset of patterns and problems; the Monster 50 and Blind 75 practice lists are usable | [Create a free account and try the practice lists](https://algo.monster?fpr=li69) |
| Pro Yearly | $5.84/mo, billed annually at $69.99/year | Recurring, cancel anytime | All patterns, lessons and problems; unlimited AI Assistant; unlimited Speedrun; real questions from top tech companies updated monthly; full access for 12 months | [Get Pro Yearly access](https://algo.monster?fpr=li69) |
| Pro Lifetime | $89 | One-time payment | Everything in the annual plan plus lifetime updates, no renewal | [Get Pro Lifetime access](https://algo.monster?fpr=li69) |

The free tier is a real evaluation path, not a demo. You can work the curated lists and judge whether the text-and-diagram style clicks before spending anything. What you cannot do is read the full pattern lessons behind the paywall.

One practical note: if you are three weeks from an interview and your sliding window recognition is the bottleneck, the yearly plan is the low-risk move. If you expect to interview again in two or three years, the lifetime plan costs about ten dollars more than the annual one, which makes the decision fairly simple at current pricing.

## Questions that come up

**How often is sliding window actually asked?** It is one of the most frequently tested patterns in array and string rounds, and it sits in the first tier of most curated interview lists, including AlgoMonster's Monster 50 and Blind 75. The reason is practical: it separates candidates who can produce an O(n) solution from candidates who reach for nested loops.

**Can I use sliding window with negative numbers?** Not for sum-based conditions. The shrink logic depends on the window's state moving in one direction as it grows. With negatives, that assumption breaks and the algorithm can return a wrong answer without any error. Use prefix sums with a hash map for those.

**Do I need to memorize the templates?** Memorize the shape, not the characters. Two decisions carry the difficulty: whether the shrink is a `while` or an `if`, and whether the answer update goes before or after the shrink loop. If you can justify both for a given problem, you can write the code.

**Is the free tier enough to learn this pattern?** Enough to practice. The curated free problem lists include sliding window entries, and that is genuinely useful for reps. The explanations, templates, and decision flowcharts are part of the paid content, so if you are stuck on *why* a shrink condition works, the free tier will not get you there.

**Should I do all fourteen problems above, or jump to the hards?** Do them in order. The hard problems in this list are hard because they stack two ideas, and both of the underlying ideas come from the easier rungs. Jumping to LC 76 before you are comfortable with LC 438 mostly teaches you that you can read a solution and still not produce it.

Start with LC 643. If you can write it in three minutes without looking anything up, move to 209 and spend your time on the shrink condition. That is where the pattern actually lives, and it is the part that transfers to the problem you have not seen yet.
