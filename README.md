# Find Peak Element: Why Binary Search Works on an Unsorted Array, and a Template You Can Reuse for Mountain and Rotated Arrays

LeetCode 162 is medium difficulty for a reason that has nothing to do with typing code. The puzzle is that binary search normally needs a sorted array, and this array is not sorted. Yet the problem statement tells you flat out to hit O(log n), so a linear scan is off the table.

Once you see why the trick is legal, the code writes itself in about six lines. Once you don't, you'll write a nested loop, pass the small test cases, and get asked "what's the time complexity?" in an interview.

## What "find peak element" is actually asking

A peak element is strictly greater than both neighbors. The array is 0-indexed, there may be several peaks, and you're allowed to return the index of *any* one of them. The statement also hands you an unusual clause: imagine `nums[-1] = nums[n] = -∞`. Elements outside the array are considered smaller than anything inside it.

That clause is the whole game. It's why the array can never be peak-free: if you walk from the left edge and the values keep rising, the boundary at the right is -∞, so a descent has to happen somewhere, and the top of that descent is a peak.

Constraints from the original problem: length between 1 and 1000, values fit in a 32-bit signed int, and no two adjacent elements are equal (`nums[i] != nums[i + 1]`). That last one matters — it guarantees no flats, so every "am I still climbing?" comparison has a definite answer.

Input: `nums = [1,2,3,1]` → output 2, because `3` at index 2 beats both neighbors.

Input: `nums = [1,2,1,3,5,6,4]` → output 5 is valid (the `6`), and output 1 is equally valid (the `2`). Both are peaks. If your solution returns 5 and the expected answer says 1, you haven't failed anything — assuming you're returning an index, not a value.

## Why binary search is legal here

Binary search needs a monotonic property to bisect on. It does not need the array itself to be sorted.

The property here is local slope. Compare `nums[mid]` with `nums[mid + 1]` and you get one of two situations:

- `nums[mid] < nums[mid + 1]` — you're on an upward slope. Keep going right and you either keep climbing until you hit the -∞ boundary (making your current position eventually peak) or you turn downward (making that turning point a peak). Either way, a peak exists somewhere in `[mid + 1, right]`. The left half is safe to discard.
- `nums[mid] > nums[mid + 1]` — you're on a downward slope. Walk right and you're only descending, and the array ends in -∞. So a peak must exist at `mid` or to its left. Discard `[mid + 1, right]`.

Every comparison throws away half the remaining range and never discards the only peak available. That's O(log n) with O(1) extra space.

The mental shift that makes this stick: you are not searching for a *value*. You're searching for a *direction change*. That's why the array being unsorted doesn't break anything.

## Rewrite it as "find the first true"

This is where AlgoMonster's binary search material earns its keep, and it's the framing that transfers to other problems.

Build a boolean array where each index answers `nums[i] > nums[i + 1]`:

For `[1, 2, 1, 3, 5, 6, 4]`:


i:        0      1      2      3      4      5      6
nums[i]:  1      2      1      3      5      6      4
condition:false   true  false  false  false   true  (edge)


The pattern is `false, false, ..., true, true, ...` — never true followed by false. The first `true` is the peak index. Once you state it that way, you're solving the standard "first true in a sorted boolean array" problem, which has a fixed template you already know.

That reframing is the actual reusable skill. The same "find the boundary where the predicate flips" move solves:

- Find Minimum in Rotated Sorted Array
- Search in Rotated Sorted Array
- First Bad Version
- Peak Index in a Mountain Array

Same six lines, different predicate. That's also why AlgoMonster files this problem under implicit sorted arrays rather than under "array tricks" — the array isn't sorted by value, it's sorted by the predicate you impose on it. The platform's own problem page walks the identical argument, using `arr[i] > arr[i+1]` to build the false/true boundary.

> A useful habit: before writing binary search, write down the predicate and list a few indexes where it's false and a few where it's true. If the pattern isn't monotone, binary search is the wrong tool and no amount of pointer juggling will fix it.

## The reference implementation

The version below matches the template used across the pattern's lesson set, and it's what you want to be able to reproduce from memory:

python
def findPeakElement(nums):
    left, right = 0, len(nums) - 1
    while left < right:
        mid = (left + right) >> 1
        if nums[mid] > nums[mid + 1]:
            right = mid          # descending: peak at mid or to the left
        else:
            left = mid + 1       # ascending: peak strictly to the right
    return left


Three things to notice.

**The loop condition is `left < right`, not `left <= right`.** You converge onto a single index instead of tracking a `boundary` variable. Fewer moving parts, fewer off-by-one bugs. If you prefer the explicit "save the answer then keep searching" shape, that works too — it's the same algorithm with an extra variable.

**No bounds check on `nums[mid + 1]`.** Because `mid = (left + right) >> 1` with `left < right` always yields `mid < right <= len(nums) - 1`, so `mid + 1` is always a valid index. Adding a defensive `if mid + 1 < n` guard here breaks the logic rather than protecting it — an out-of-range read would silently jump to the `else` branch and send you right when you should go left.

**The single-element case falls out for free.** `left = right = 0`, the loop never runs, and `0` is returned. With the -∞ fiction, index 0 is legitimately a peak in a one-element array.

A JavaScript version, since the same job interview may not be in Python:

javascript
function findPeakElement(nums) {
  let left = 0, right = nums.length - 1;
  while (left < right) {
    const mid = (left + right) >> 1;
    if (nums[mid] > nums[mid + 1]) right = mid;
    else left = mid + 1;
  }
  return left;
}


## Walkthrough on `[1, 2, 1, 3, 5, 6, 4]`

Trace it once and the slope argument stops feeling abstract.

1. `left = 0, right = 6`, `mid = 3`. `nums[3] = 3 < nums[4] = 5` → climbing. `left = 4`.
2. `left = 4, right = 6`, `mid = 5`. `nums[5] = 6 > nums[6] = 4` → descending. `right = 5`.
3. `left = 4, right = 5`, `mid = 4`. `nums[4] = 5 < nums[5] = 6` → climbing. `left = 5`.
4. `left == right == 5`. Return 5.

Four comparisons for seven elements. Verify: `6 > 5` and `6 > 4`. Valid peak. The `2` at index 1 was also acceptable and got discarded along the way — that's fine, the problem only requires one.

## Edge cases and the mistakes that show up in interviews

**Returning a value instead of an index.** The problem asks for the index. Reading the question carelessly is the most common silent failure here.

**Assuming the answer must be the global maximum.** It doesn't have to be. In `[1, 2, 1, 3, 5, 6, 4]`, the `2` at index 1 is a legitimate answer. If a reviewer's test expects 5 and you return 1, both are correct against the stated problem — but if you reasoned "find the max with a scan," you've written O(n) and violated the requirement.

**Solving it with a linear scan.** It passes every test case. It also fails the explicit O(log n) requirement, and interviewers do check the complexity you claim. Write the scan only if you say out loud that you're starting with a baseline and then optimize.

**Overflow in languages without arbitrary-precision ints.** `(left + right) / 2` can overflow in Java or C++ at large indices. `left + (right - left) / 2` doesn't. In Python this is irrelevant, which is why Python solutions at big-tech interviews sometimes skip it and get dinged by reviewers who want the safe habit.

**Copying `mid == n - 1` guards from the mountain-array version into this one.** This trips people up, and it's visible in the public discussion threads on AlgoMonster's mountain-array problem: several readers point out that the `mid == alen - 1` term in that solution is redundant given the problem guarantees the peak is never the first or last element, and that they got correct output on custom inputs like `[7, 8, 9]` without it. Adding conditions that can't fire is a real code-review flag.

**Mixing `left < right` with `right = mid - 1`.** Pick one convention and stay in it. The converging version uses `right = mid` precisely because the narrowing has to keep `mid` in play. Accidentally writing `mid - 1` there produces an off-by-one that only shows on specific inputs.

## The variants worth drilling in the same session

These are the same predicate, different boundary rules, and interviewers rotate between them.

| Problem | Array shape | Peak/answer can be at the ends? | Initial range |
| --- | --- | --- | --- |
| Find Peak Element (162) | Arbitrary, no adjacent equals, ends treated as -∞ | Yes | `left = 0, right = n - 1` |
| Peak Index in a Mountain Array (852) | Strictly increases then strictly decreases | No — at least 3 elements, single peak in the middle | `left = 1, right = n - 2` |
| Peak of Mountain Array (AlgoMonster's version) | Strictly increases then strictly decreases | No | `left = 0, right = n - 1` with an explicit `mid == n - 1` term |

The difference between rows two and three is a good illustration of why you shouldn't memorize code. The 852-shaped problem tells you the peak is strictly interior, so you can safely shrink the search window to exclude both endpoints. AlgoMonster's framing keeps the full range and compensates with an extra boundary check — functionally fine, just carrying a condition the problem statement makes unnecessary.

Whichever you write, say the range logic out loud before coding. That's the part interviewers are evaluating, not the syntax.

## How to actually drill this pattern instead of memorizing the six lines

Knowing the answer to one problem gets you through one problem. The reason "find peak element" shows up on so many interview lists is that it's the cleanest test of whether you can construct a monotonic predicate on data that isn't sorted — and that skill is what rotates into rotated arrays, boundary searches, and allocation problems.

AlgoMonster's structure fits this problem specifically because it groups problems by pattern rather than by topic. Peak problems live under binary search over implicit sorted arrays, next to the "first true in a sorted boolean array" module that this solution reduces to. You see the template once and then hit five or six problems where the only thing that changes is the feasibility condition.

### Current plans and what they cost

Prices on this platform move with promotions, so the checkout page is the only authority — the list prices below are what a discount gets applied to.

| Plan | What's included | Price | Billing |
| --- | --- | --- | --- |
| Free access | Limited sample lessons and problems — enough to check whether the text-and-diagram teaching style works for you | $0 | No card required |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; real questions from major tech companies updated monthly; 3,000+ editorials and illustrations; member discount on live bootcamps | Promotional checkout has shown $69.99–$99/year ($5.84–$8.25/mo). List price $198/year | Billed annually, 12 months of access — [ start the annual plan](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates and permanent access to future content | Promotional checkout has shown $89–$189 one-time. List price $378 — [ check the current lifetime price](https://algo.monster?fpr=li69) | One-time payment |

A few buying notes that are worth knowing before you decide:

- There's no monthly option. It's annual or lifetime, and third-party reviews consistently confirm no team or enterprise tier either.
- The discount is already applied on the pricing page rather than requiring a code. Independent sources do list codes like `FAANG20` and `SCHELLEY` for extra percentages, but they're third-party claims about a moving target — check what the checkout actually charges you before assuming a code stacks.
- The yearly plan makes sense if you're interviewing in the next few months. Lifetime makes sense if you expect to interview again in two or three years. If you go in, do it for the pattern framework, not the problem count: the bank is small relative to LeetCode, and reviews across independent sites agree on that.
- Known limits, also consistent across reviews: everything is text plus diagrams (no video walkthroughs), and the system design and behavioral coverage is introductory rather than senior-level.

One way to spend a first week here: read the binary search over implicit sorted arrays module end to end, then solve 162, 852, and Find Minimum in Rotated Sorted Array without looking at the templates. If you can write the predicate for all three from a blank file, the pattern has stuck. If you can't, you've found the gap while it's still cheap to fix.

## FAQ

**Why not just find the maximum element?**
Finding the global max is O(n) and returns a valid peak, but the problem explicitly requires O(log n). It's also not what's being tested — the interviewer wants the slope-bisection reasoning.

**Does binary search really work on an unsorted array?**
Yes, when you can define a predicate that's monotone over the indexes. Here the predicate is `nums[i] > nums[i + 1]`, which is false up to the peak then true from the peak onward.

**What if there are two peaks?**
Return either. Any index `i` where `nums[i] > nums[i-1]` and `nums[i] > nums[i+1]` (with out-of-bounds treated as -∞) is accepted.

**What's the time and space complexity?**
O(log n) time and O(1) space. The search range halves on every iteration and only two pointers are stored.

**Can I use `left <= right` with a saved answer instead?**
Yes. Save the index whenever the descending condition holds, then set `right = mid - 1` and keep going. Same complexity, one extra variable, and it's the shape AlgoMonster's mountain-array solution uses. The converging `left < right` version is shorter and harder to get wrong once you trust it.
