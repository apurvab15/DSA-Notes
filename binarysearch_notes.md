# Binary Search — Study Notes

---

# Part 0: The Framework

## Binary search is NOT "find x in a sorted array"

**Binary search = find the boundary of a monotonic predicate.**

You have a search space (indices, or answer values) where some predicate `feasible(x)` looks like:

```
x:           1    2    3    4    5    6    7    8
feasible:    F    F    F    T    T    T    T    T
                            ^
                            binary search finds THIS flip point
```

"Find target in sorted array" is just the special case where the predicate is `arr[i] >= target`.

**The organizing question for every problem in this file:**

> *"What is my predicate, and why is it monotonic?"*

If you can answer that, the code writes itself from one template. If you can't, binary search doesn't apply (yet — maybe the problem needs remodeling).

## The One Template (lower-bound style)

Find the **smallest** `x` in `[lo, hi]` where `feasible(x)` is true:

```java
int lo = /* smallest candidate */, hi = /* largest candidate */;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;   // floors, biases LEFT
    if (feasible(mid)) {
        hi = mid;        // mid works — keep it as a candidate
    } else {
        lo = mid + 1;    // mid fails — discard it and everything left of it
    }
}
return lo;   // lo == hi == the boundary
// If the answer might not exist, verify feasible(lo) before returning.
```

**Why this template never infinite-loops:** `mid` floors toward `lo`, so `mid < hi` always. The `hi = mid` branch strictly shrinks the range; the `lo = mid + 1` branch strictly shrinks it too. Guaranteed progress.

**The invariant (say this out loud in interviews):**
- Everything left of `lo` is confirmed **infeasible**.
- Everything from `hi` onward is... not yet ruled out (and `hi` itself may be confirmed feasible).
- Loop ends when the unknown region collapses to one point: the boundary.

## The Flipped Template (upper-bound / maximize style)

Find the **largest** `x` where `feasible(x)` is true (predicate goes T T T F F F):

```java
while (lo < hi) {
    int mid = lo + (hi - lo + 1) / 2;   // +1 → CEILS, biases RIGHT
    if (feasible(mid)) {
        lo = mid;        // mid works — keep it
    } else {
        hi = mid - 1;    // mid fails — discard it and everything right
    }
}
return lo;
```

**Critical pairing rule (memorize this — it kills 90% of infinite loops):**

| If your branches are... | mid must... |
|---|---|
| `hi = mid` / `lo = mid + 1` | floor: `lo + (hi - lo) / 2` |
| `lo = mid` / `hi = mid - 1` | ceil: `lo + (hi - lo + 1) / 2` |

The rule in one sentence: **whichever pointer can be set to `mid` itself, `mid` must be biased AWAY from that pointer.** Otherwise a 2-element range can loop forever.

## The Three Templates: Lower Bound, Upper Bound, Exact Match

### 1. Lower bound — first index with `nums[i] >= target`

```java
int lowerBound(int[] nums, int target) {
    int lo = 0, hi = nums.length;              // hi = n (can return "past the end")
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] >= target) hi = mid;     // >= : mid might BE the answer, keep it
        else lo = mid + 1;
    }
    return lo;    // first i with nums[i] >= target; n if none
}
```

`[2,4,4,4,7]`, target 4 → **1** (first 4).

### 2. Upper bound — first index with `nums[i] > target`

```java
int upperBound(int[] nums, int target) {
    int lo = 0, hi = nums.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] > target) hi = mid;      // > : strictly greater
        else lo = mid + 1;
    }
    return lo;    // first i with nums[i] > target; n if none
}
```

`[2,4,4,4,7]`, target 4 → **4** (one past the last 4).

### 3. Exact match — index of `target`, or −1

```java
int find(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;          // hi = n-1: searching real elements
    while (lo <= hi) {                         // <= : range can shrink to empty
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;   // early exit
        else if (nums[mid] < target) lo = mid + 1;
        else hi = mid - 1;                     // mid checked, safe to discard fully
    }
    return -1;
}
```

### The nuanced differences

**Lower vs upper bound: ONE character.** `>=` vs `>`. Both find the first true index of a predicate:
- lower bound: "at least target" → `>=`
- upper bound: "past target" → `>`

**Bounds vs exact match: three changes that always travel together.**

| | Lower/upper bound | Exact match |
|---|---|---|
| `hi` starts at | `n` | `n − 1` |
| loop condition | `lo < hi` | `lo <= hi` |
| after checking mid | `hi = mid` (mid may be the answer) | `hi = mid − 1` (mid fully ruled out) |
| loop ends by | converging on the boundary | returning, or range going empty |

*Why they differ:* in boundary-finding, a true `mid` might itself be the answer — you can never discard it, so `hi = mid`, which forces `lo < hi` to terminate. In exact match, the `==` check fully resolves mid — discard it from both sides (`mid ± 1`), and `lo <= hi` lets the range go empty. And `hi = n` vs `n − 1`: bounds must be able to say "nothing qualifies" as the value `n`; exact match says it with `−1`.

**Never splice templates.** `lo <= hi` with `hi = mid` → infinite loop. `lo < hi` with `hi = mid − 1` → can skip the boundary. Each column above is an all-or-nothing package.

### Identities worth memorizing (sorted array with duplicates)

```
count of target      = upperBound(t) − lowerBound(t)
first occurrence     = lowerBound(t)        (verify nums[i] == t)
last occurrence      = upperBound(t) − 1    (verify)
floor  (last  <= t)  = upperBound(t) − 1
ceiling(first >= t)  = lowerBound(t)
upperBound(t)        = lowerBound(t + 1)    (for integers)
```

The last identity means you only need to OWN one template: lower bound. Derive upper bound as `lowerBound(t + 1)`, and exact match as `lowerBound(t)` + an equality check.

## Tips (framework-level)

- **Only ONE template.** Do not juggle `lo <= hi` vs `lo < hi` per problem. Use `while (lo < hi)` boundary-finding everywhere possible; the only place `lo <= hi` appears in these notes is find-exact-match style (Rotated Search) and partition search (Median), where the loop can terminate by *returning*, not by converging.
- `lo + (hi - lo) / 2` not `(lo + hi) / 2` — overflow. In Java this matters for value-space searches where lo, hi can be near `Integer.MAX_VALUE`.
- **Watch for long overflow inside `feasible`** — sums of workloads, `m * k` counts, etc. This is the #1 silent bug in BS-on-answer problems.
- Dry-run discipline: test the template on a **2-element range** where the answer is each of the two elements. That's the minimal example that catches every off-by-one and infinite loop.
- Search space size `n` → O(log n) iterations. For value-space searches, `n` = (hi − lo), so complexity is O(cost(feasible) · log(range)).


---

# Decision Cheat Sheet (read this the morning of)

Organized by the questions you ask **in order**. The two lines that make it automatic: **draw the boundary + say the predicate.** Everything else is lookup.

## Step 1 — Draw the boundary. What am I searching?

Write the YYYNN / NNNYYY line and mark the arrow *before* touching the loop.

| I want... | Predicate shape | Answer is | Return |
|---|---|---|---|
| first thing that qualifies | `N N N Y Y` | first Y | `lo` |
| last thing that qualifies | `Y Y Y N N` | last Y | **reframe** ↓ |
| exact element | — | equality hit | `mid` on match, else `-1` |

**The reframe rule (most important line here).** Never search "last Y" directly — it forces the flipped/ceil template that keeps causing infinite loops. Convert it:

```
last Y of P     ≡     (first Y of NOT-P)  −  1
floor(t)        =     lowerBound(t+1) − 1
ceiling(t)      =     lowerBound(t)
last True       =     (first False) − 1
```

Every maximize / floor / predecessor problem becomes a **first-Y** problem → the one default template. If you're reaching for ceil-mid, you skipped this step.

## Step 2 — Pick the template (only two you ever write)

**Default — lower bound (first Y):**
```java
int lo = 0, hi = n;                    // hi = n: "no Y" lands at n
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;      // floor
    if (predicate(mid)) hi = mid;      // Y: keep, boundary here-or-left
    else lo = mid + 1;                 // N: discard, boundary strictly right
}
return lo;
```

**Exact match (element must be found & returned early)** — only for Rotated Search & Median-style:
```java
int lo = 0, hi = n - 1;                // hi = n-1: "-1" is the not-found answer
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;      // floor
    if (found) return mid;
    else if (goLeft) hi = mid - 1;
    else lo = mid + 1;
}
return -1;
```

The flipped/ceil template exists but Step 1's reframe removes the need — don't write it.

## Step 3 — The pairing rule (anti-splice guard)

Only these packages are legal. Anything else infinite-loops or skips the seam:

| loop | updates | mid |
|---|---|---|
| `lo < hi` | `hi = mid` / `lo = mid + 1` | **floor** |
| `lo < hi` | `lo = mid` / `hi = mid - 1` | **ceil** |
| `lo <= hi` | `hi = mid - 1` / `lo = mid + 1` | floor |

One rule regenerates the table: **the pointer that can hold `mid` forces `mid` to bias away from it.** `lo = mid` → ceil. `hi = mid` → floor.

## Step 4 — Set `hi` by asking "what if nothing qualifies?"

| the search... | `hi` starts at | because "none" returns |
|---|---|---|
| lower/upper bound | `n` | `n` — past the end must be sayable |
| exact match | `n - 1` | `-1` |
| answer-space (guaranteed to exist) | max candidate value | n/a |

*Why it matters:* the boundary you're searching may not be inside the array (all-N → answer is `n`, past the end). `hi = n` gives that answer a home. `hi = n-1` makes the all-N case wrong or stuck.

## Step 5 — Which search SPACE? (names the whole pattern)

| Signal in the problem | Space | Predicate |
|---|---|---|
| "first ≥ / insert position / count of x" | indices | `a[i] >= t` |
| rotated / one seam | indices | "which half is sorted?" invariant |
| peak / single element | indices (maybe **derived**: even positions, blocks) | slope / parity |
| "min speed/capacity such that..." | **answer values** | greedy check `<= budget` |
| "max length/gap such that..." | answer values → **reframe to min** | greedy check `>= k` |
| "kth smallest" over huge/implicit set | value range | `countLE(x) >= k` |
| median / kth of two sorted | **cut positions** | two-sided border invariant |
| floor/predecessor in a design | indices / TreeMap | `ts[i] <= t` → reframe |

## Pre-flight — run before saying "done" (2 seconds)

1. **Predicate aloud, WITH comparator:** "at least k," "at most m," "strictly greater." (`<` vs `<=` is the #1 recurring bug.)
2. **2-element dry run**, answer = each element. Catches every infinite loop and off-by-one.
3. **Not-found:** does the empty / all-N case return the right sentinel?
4. **BS-on-answer only:** does `feasible(hi)` trivially pass? Is `lo` on the invariant's side (**max**, not min)? Is the accumulator `long`?

## Pattern-specific landmines

- **Rotated w/ dups (81):** equal borders kill the discard guarantee → `lo++, hi--`; worst case **O(n)** — *volunteer this.*
- **Real-valued space (gas stations):** no `mid ± 1`; loop on `hi - lo > eps`, use `lo = mid`.
- **Median:** name all four borders on their own lines; every real comparison is cross-array; search the **smaller** array.
- **Answer-space min bound:** `lo = max(element)` is often **correctness**, not tightness — the greedy check silently lies below it.
---

# Tier 0: Template Mechanics

---

## Binary Search (LC 704)

**Description:** Sorted array of distinct ints, return index of `target` or −1.

**Example:** `nums = [-1,0,3,5,9,12], target = 9 → 4`

**Brute force:** Linear scan, O(n).

**Intuition:** Predicate = `nums[i] >= target`. It's monotonic because the array is sorted: once values reach `target`, they stay ≥ it. Find the boundary (lower bound), then check whether the element *at* the boundary actually equals target.

Concretely for target 9: predicate over `[-1,0,3,5,9,12]` is `F F F F T T`. Lower bound lands on index 4, `nums[4] == 9` ✓.

**Predicate:** `nums[i] >= target`, search space = indices `[0, n]`.

**Time:** O(log n)
**Space:** O(1)

```java
public int search(int[] nums, int target) {
    int lo = 0, hi = nums.length;           // hi = n, NOT n-1: "not found
                                            // anywhere" must be representable
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] >= target) hi = mid;
        else lo = mid + 1;
    }
    return (lo < nums.length && nums[lo] == target) ? lo : -1;
}
```

**Tips:**
- `hi` starts at `n`, not `n − 1`. Lower bound must be able to return `n` ("everything is smaller than target"). Forgetting this is the classic bug.
- The final existence check (`lo < n && nums[lo] == target`) is the pattern for every "might not exist" search.

**Trick:** In Java, `Arrays.binarySearch(nums, target)` returns `-(insertionPoint) - 1` when absent — so `insertionPoint = -(result + 1)`. Knowing this saves you writing lower bound when the interviewer allows library calls.

---

## Search Insert Position (LC 35)

**Description:** Sorted distinct array; return index of `target`, or the index where it *would* be inserted.

**Example:** `nums = [1,3,5,6], target = 2 → 1`

**Brute force:** Scan for first `nums[i] >= target`, O(n).

**Intuition:** This problem IS lower bound, stated in plain English. "Where would it be inserted" = "first index whose value is ≥ target". No existence check needed — the boundary itself is the answer either way.

**Predicate:** `nums[i] >= target`, space = `[0, n]`.

**Time:** O(log n)
**Space:** O(1)

```java
public int searchInsert(int[] nums, int target) {
    int lo = 0, hi = nums.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] >= target) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}
```

**Tips:** When someone says "insertion point", "first element ≥ x", "smallest index such that..." — it's lower bound verbatim.

**Trick:** `TreeMap.ceilingKey(x)` / `TreeSet.ceiling(x)` are lower bound over dynamic data — same concept when the collection mutates.

---

## Find First and Last Position of Element in Sorted Array (LC 34)

**Description:** Sorted array with duplicates; return `[firstIndex, lastIndex]` of `target`, or `[-1,-1]`.

**Example:** `nums = [5,7,7,8,8,10], target = 8 → [3,4]`

**Brute force:** Scan once for first, once for last, O(n).

**Intuition:** Two lower bounds:
- First occurrence = lower bound of `target` (first index with value ≥ target).
- Last occurrence = **lower bound of `target + 1`, minus 1** (one before the first element strictly greater).

Concretely: `[5,7,7,8,8,10]`, target 8. Lower bound of 8 → index 3. Lower bound of 9 → index 5, minus 1 → 4. Answer `[3,4]`.

This "reuse lower bound with target+1" move means you write ONE helper and never write a separate upper-bound loop.

**Predicate:** `nums[i] >= t` for two values of `t`.

**Time:** O(log n)
**Space:** O(1)

```java
public int[] searchRange(int[] nums, int target) {
    int first = lowerBound(nums, target);
    if (first == nums.length || nums[first] != target) return new int[]{-1, -1};
    int last = lowerBound(nums, target + 1) - 1;
    return new int[]{first, last};
}

private int lowerBound(int[] nums, int target) {
    int lo = 0, hi = nums.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] >= target) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}
```

**Tips:**
- Count of occurrences of `x` in a sorted array = `lowerBound(x+1) - lowerBound(x)`. Comes up as a follow-up constantly.
- Existence check happens once, after the *first* search — if first fails, skip the second.

**Trick:** For non-integer or overflow-prone types where `target + 1` isn't safe, write upper bound as lower bound with predicate `nums[i] > target` (strict) instead.

---

## First Bad Version (LC 278)

**Description:** Versions `1..n`; some version went bad and all later versions are bad. API `isBadVersion(v)`. Find the first bad one, minimizing API calls.

**Example:** `n = 5, first bad = 4`: `G G G B B → 4`

**Brute force:** Call `isBadVersion` on 1, 2, 3, ... — O(n) calls.

**Intuition:** The problem hands you the predicate directly — `isBadVersion` IS `feasible`, and the problem statement even guarantees monotonicity ("all versions after a bad version are also bad"). This is the purest possible predicate-boundary problem; the interview value is narrating it that way.

**Predicate:** `isBadVersion(v)`, space = `[1, n]`.

**Time:** O(log n) API calls
**Space:** O(1)

```java
public int firstBadVersion(int n) {
    int lo = 1, hi = n;      // answer guaranteed to exist, so hi = n (not n+1)
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (isBadVersion(mid)) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}
```

**Tips:** When the answer is *guaranteed* to exist in `[1, n]`, `hi` starts at `n` and no post-check is needed. Compare with LC 704 where `hi = n` exists precisely because "no answer" is possible. Knowing *why* the ranges differ is the mastery signal.

**Trick:** "Minimize API calls" is interviewer-speak for "the feasibility check is expensive" — the exact mindset for Tier 2, where `feasible` costs O(n).

---
# Tier 1: Broken Monotonicity in Index Space

The array isn't globally sorted, but **some invariant survives** that tells you which half to discard. The skill here is *finding the surviving invariant*.

---

## Search in Rotated Sorted Array (LC 33)

**Description:** Sorted distinct array rotated at an unknown pivot (e.g. `[0,1,2,4,5,6,7]` → `[4,5,6,7,0,1,2]`). Find `target`'s index or −1, in O(log n).

**Example:** `nums = [4,5,6,7,0,1,2], target = 0 → 4`

**Brute force:** Linear scan O(n) — or find the pivot with one binary search, then a normal binary search in the correct half (two passes, still O(log n) but more code).

**Intuition:** Global sortedness is broken, but the surviving invariant is:

> **At any `mid`, at least ONE of the two halves is fully sorted.**

Why: there's only one "break point" (the rotation seam), and `mid` splits the array into two halves — the seam can live in at most one of them. The other half is clean.

So the algorithm is: identify the sorted half (one comparison: `nums[lo] <= nums[mid]`?), then ask "is target inside the sorted half's range?" — a question you can answer *because* it's sorted. If yes, go there; if no, the target must be in the messy half.

Concretely, `[4,5,6,7,0,1,2]`, target 0, mid = 3 (value 7):
- `nums[0]=4 <= nums[3]=7` → left half `[4,5,6,7]` is sorted.
- Is 0 in `[4, 7)`? No → discard left, search `[0,1,2]`. ✓

**Predicate:** none directly — this is "discard a half by invariant" rather than pure boundary-finding, hence the `lo <= hi` find-exact loop (we can exit early by returning).

**Time:** O(log n)
**Space:** O(1)

```java
public int search(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;

        if (nums[lo] <= nums[mid]) {                    // left half sorted
            if (nums[lo] <= target && target < nums[mid])
                hi = mid - 1;                           // target in sorted left
            else
                lo = mid + 1;
        } else {                                        // right half sorted
            if (nums[mid] < target && target <= nums[hi])
                lo = mid + 1;                           // target in sorted right
            else
                hi = mid - 1;
        }
    }
    return -1;
}
```

**Tips:**
- `nums[lo] <= nums[mid]` — the `<=` matters. When `lo == mid` (2-element range), the left "half" is one element, which is trivially sorted; using `<` misroutes this case.
- The range checks are half-open on the mid side (`target < nums[mid]`) because `mid` was already checked for equality.
- Dry-run on the 2-element cases `[3,1]` target 1, and `[1,3]` target 3 — these catch every variant of the boundary bug.

**Trick (follow-up: LC 81, duplicates allowed):** With duplicates, `nums[lo] == nums[mid] == nums[hi]` tells you *nothing* about which half is sorted (e.g. `[1,1,1,0,1]`). Handle it by shrinking: `lo++, hi--`. Worst case degrades to O(n) — say this proactively; it's the entire point of the follow-up.

---

## Find Minimum in Rotated Sorted Array (LC 153)

**Description:** Rotated sorted distinct array; return the minimum element.

**Example:** `nums = [3,4,5,1,2] → 1`

**Brute force:** Linear scan, O(n).

**Intuition:** Back to a clean predicate! Compare everything against the **last element**:

> `feasible(i) = nums[i] <= nums[n-1]`

Elements before the seam are all > `nums[n-1]`; elements from the minimum onward are all ≤ `nums[n-1]`. So the predicate is `F F F T T` and the boundary is exactly the minimum.

Concretely `[3,4,5,1,2]` vs last element 2: `F F F T T` → boundary at index 3, value 1. ✓

If the array wasn't rotated at all (`[1,2,3,4,5]`): all `T`, boundary = index 0. Handled for free — no special case.

**Predicate:** `nums[i] <= nums[n-1]`, space = `[0, n-1]` (answer guaranteed to exist).

**Time:** O(log n)
**Space:** O(1)

```java
public int findMin(int[] nums) {
    int lo = 0, hi = nums.length - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] <= nums[nums.length - 1]) hi = mid;   // in the "low" run
        else lo = mid + 1;                                  // still in the "high" run
    }
    return nums[lo];
}
```

**Tips:**
- Comparing against `nums[hi]` (shrinking) also works and is the common writeup; comparing against the *fixed* `nums[n-1]` is easier to reason about as a static predicate. Pick one and stick with it.
- Comparing against `nums[lo]` does NOT work cleanly (the not-rotated case breaks) — if you catch yourself doing that, stop and re-anchor.

**Trick:** Once you have the min's index `p`, LC 33 can be solved as: binary search in `[0, p-1]` or `[p, n-1]` (pick by comparing target with `nums[n-1]`). Good to mention as the "two clean searches" alternative.

---

## Find Peak Element (LC 162)

**Description:** Array where `nums[i] != nums[i+1]`; find ANY peak (element greater than both neighbors). `nums[-1] = nums[n] = -∞`. O(log n) required.

**Example:** `nums = [1,2,3,1] → 2` (value 3)

**Brute force:** Scan for first `i` with `nums[i] > nums[i+1]`, O(n).

**Intuition:** The array has NO sortedness at all — and binary search still works. This is the problem that proves the predicate framing is more fundamental than "sorted".

> `feasible(i) = nums[i] > nums[i+1]`  ("the downhill has started")

Why is the boundary a peak? Walk it: if `nums[mid] < nums[mid+1]` you're on an uphill slope — since the array ends at −∞, that slope MUST crest somewhere to the right. So a peak is guaranteed right of mid. If `nums[mid] > nums[mid+1]`, mid could itself be the peak (or the peak is left) — keep it.

Concretely `[1,2,3,1]`: predicate at i = 0,1,2 is `F F T` → boundary index 2. ✓

**Predicate:** `nums[i] > nums[i+1]`, space = `[0, n-2]` — and if all F, the answer is `n-1` (array strictly rising into the last element). Setting `hi = n-1` handles that automatically.

**Time:** O(log n)
**Space:** O(1)

```java
public int findPeakElement(int[] nums) {
    int lo = 0, hi = nums.length - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] > nums[mid + 1]) hi = mid;   // downhill: peak is mid or left
        else lo = mid + 1;                          // uphill: peak strictly right
    }
    return lo;
}
```

**Tips:**
- `mid + 1` never overflows the array: `mid < hi <= n-1` because mid floors.
- Interview narration gold: *"We're not searching sorted data — we're searching for the boundary between 'still climbing' and 'started descending', and the −∞ walls guarantee that boundary exists."*

**Trick:** Peak Index in a Mountain Array (LC 852) is this exact code. Find in Mountain Array (LC 1095) = find peak, then one ascending search left of it + one descending search right of it.

---

## Single Element in a Sorted Array (LC 540)

**Description:** Sorted array where every element appears exactly twice except one; find it in O(log n).

**Example:** `nums = [1,1,2,3,3,4,4,8,8] → 2`

**Brute force:** XOR everything (O(n)) or scan pairs (O(n)).

**Intuition:** The surviving invariant is **parity of pair alignment**:

- *Before* the single element, pairs start at EVEN indices: `(0,1), (2,3), ...` — so `nums[even] == nums[even+1]`.
- *After* it, everything shifts by one: pairs start at ODD indices — so `nums[even] != nums[even+1]`.

> `feasible(evenIndex i) = nums[i] != nums[i+1]`  ("the misalignment has started")

The boundary (first misaligned even index) is exactly where the single element sits.

Concretely `[1,1,2,3,3,4,4,8,8]`, checking even indices 0,2,4,6: pairs `(1,1) (2,3) (3,4) (4,8)` → `F T T T` → boundary at even index 2 → answer `nums[2] = 2`. ✓

**Predicate:** over even indices only: `nums[i] != nums[i+1]`.

**Time:** O(log n)
**Space:** O(1)

```java
public int singleNonDuplicate(int[] nums) {
    int lo = 0, hi = nums.length - 1;        // hi is always even (odd length)
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (mid % 2 == 1) mid--;             // snap mid to even
        if (nums[mid] != nums[mid + 1]) hi = mid;      // misaligned: single is here or left
        else lo = mid + 2;                              // aligned: single strictly right
    }
    return nums[lo];
}
```

**Tips:**
- Snapping `mid` to even keeps the search space "array of even positions" — cleaner than juggling both parities in your head.
- Note `lo = mid + 2` (next even index), not `+1`. Dry-run on `[1,1,2]` and `[2,1,1]`.

**Trick:** The transferable idea is **"binary search over a derived index space"** — here, even positions. Same move appears in problems that binary search over pair-starts, block-starts, etc. Ask: *"what's the smallest unit where my invariant is clean?"*

---
# Tier 2: Binary Search on the Answer

**The Google heavy-hitter.** New search space: not indices — the **range of possible answers**. `feasible(x)` is a greedy O(n) check: "could the answer be as good as x?"

**Recognition signals:**
- "Minimize the maximum ..." / "maximize the minimum ..."
- "Smallest speed/capacity/threshold such that a constraint holds"
- Answer lives in a numeric range, and making the candidate MORE generous can only help (→ monotonic!)

**Why feasibility is monotonic (the one-line proof you volunteer):** any schedule that works with capacity `c` still works with capacity `c+1` — extra slack never hurts. So `feasible` is `F F F T T T`.

**The recipe:**
1. Answer range `[lo, hi]` — lo = tightest imaginable answer, hi = loosest.
2. `feasible(mid)` — usually a greedy single pass.
3. Lower-bound template. Total: **O(n · log(range))**.

---

## Koko Eating Bananas (LC 875)

**Description:** Piles of bananas; each hour Koko eats up to `k` from ONE pile. Find the minimum `k` to finish all piles within `h` hours.

**Example:** `piles = [3,6,7,11], h = 8 → 4`

**Brute force:** Try k = 1, 2, 3, ... until one works: O(max(piles) · n).

**Intuition:** The brute force scans `F F F T T T ...` linearly — the answer space is monotonic (faster eating always finishes no later), so binary search the flip point instead.

- `hours(k) = Σ ceil(pile / k)` — hours needed at speed k.
- `feasible(k) = hours(k) <= h`.

Concretely, `[3,6,7,11], h=8`:
- k=4: 1+2+2+3 = 8 ✓   k=3: 1+2+3+4 = 10 ✗
- Predicate over k=1..11: `F F F T T T T T T T T` → boundary k=4. ✓

Range: lo = 1 (can't eat 0), hi = max(piles) (faster than the biggest pile is wasted — 1 hour per pile is the floor).

**Predicate:** `feasible(k) = Σ ceil(pileᵢ / k) <= h`, space = `[1, max(piles)]`.

**Time:** O(n · log(max(piles)))
**Space:** O(1)

```java
public int minEatingSpeed(int[] piles, int h) {
    int lo = 1, hi = 0;
    for (int p : piles) hi = Math.max(hi, p);

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (feasible(piles, mid, h)) hi = mid;
        else lo = mid + 1;
    }
    return lo;    // answer guaranteed: k = max(piles) always works (n <= h given)
}

private boolean feasible(int[] piles, int k, int h) {
    long hours = 0;                          // long: up to 1e4 piles * 1e9 each
    for (int p : piles) {
        hours += (p + k - 1) / k;            // ceil division, no doubles
        if (hours > h) return false;         // early exit
    }
    return hours <= h;
}
```

**Tips:**
- **Integer ceil: `(p + k - 1) / k`** — never `Math.ceil((double) p / k)` (precision risk at large values, and slower).
- Accumulate hours in `long`. This is THE overflow spot.
- Early-exit inside `feasible` once `hours > h` — free constant-factor win, nice to mention.

**Trick:** Every Tier 2 problem is this code with a different `feasible`. Practice writing the shell from memory in <60 seconds, so all interview thought goes into the feasibility check.

---

## Capacity To Ship Packages Within D Days (LC 1011)

**Description:** Weights must ship IN ORDER; a ship carries at most `capacity` per day. Find the minimum capacity to ship within `days` days.

**Example:** `weights = [1,2,3,4,5,6,7,8,9,10], days = 5 → 15`

**Brute force:** Try each capacity from max(w) upward, simulate: O(sum(w) · n).

**Intuition:** Same skeleton as Koko; the new piece is the greedy feasibility check for an *ordered partition*:

> `feasible(cap)`: sweep left to right, pack greedily into the current day until adding the next weight would exceed `cap`, then open a new day. Count days ≤ `days`?

Greedy is optimal here because packing as much as possible per day can never increase the day count (exchange argument — worth stating, not proving).

Range: lo = **max(weights)** (a day must fit the heaviest single package), hi = sum(weights) (everything in one day).

> **CORRECTION — `lo = max(weights)` is load-bearing for CORRECTNESS, not just tightness.** The greedy check silently lies below that bound: `weights=[7,2,5,10,8]`, cap=9 → the walk produces `[7,2] [5] [10] [8]` = 4 days, but `[10]` is a day of load 10 under a cap of 9. The check never notices — it just opens a new day and dumps the 10 in. So a too-low `lo` can make an impossible capacity report as feasible. (Koko is different: its check is a real formula, `Σ ceil(pile/k)`, valid for every k ≥ 1. Don't generalize from it.)

Concretely, cap = 15 on the example: `[1..5][6,7][8][9][10]` = 5 days ✓; cap = 14: `[1..4][5,6][7][8][9][10]` = 6 days ✗.

**Predicate:** `feasible(cap) = greedyDays(cap) <= days`, space = `[max(w), sum(w)]`.

**Time:** O(n · log(sum(w)))
**Space:** O(1)

```java
public int shipWithinDays(int[] weights, int days) {
    int lo = 0, hi = 0;
    for (int w : weights) { lo = Math.max(lo, w); hi += w; }

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (daysNeeded(weights, mid) <= days) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}

private int daysNeeded(int[] weights, int cap) {
    int days = 1, load = 0;
    for (int w : weights) {
        if (load + w > cap) { days++; load = 0; }   // open a new day
        load += w;
    }
    return days;
}
```

**Tips:**
- `days` starts at 1, not 0 — the first package already occupies a day. Dry-run `weights=[1], days=1`.
- Keep the "open a new day" reset (`load = 0`) and the "add weight" (`load += w`) as **separate lines** — fusing them into one expression is exactly the fuse-things-that-want-to-be-separate trap.

**Trick:** "In order / contiguous" is what makes greedy valid. If packages could be reordered, this becomes bin-packing-like (different problem). Flag the ordering assumption out loud.

---

## Split Array Largest Sum (LC 410)

**Description:** Split `nums` into `k` non-empty contiguous subarrays, minimizing the largest subarray sum.

**Example:** `nums = [7,2,5,10,8], k = 2 → 18` (split `[7,2,5] [10,8]`)

**Brute force / alternative:** Interval-style DP — `dp[i][j]` = best answer for first `i` elements in `j` parts, trying every last-split point: **O(k · n²) time, O(k · n) space**. Correct, but dominated below.

**Intuition:** Flip the question. Instead of "what's the minimum largest-sum?", ask the *decision* version: "CAN we split into ≤ k parts where every part sums ≤ cap?" — and that is literally `daysNeeded(cap) <= k` from the shipping problem. Same greedy, same monotonicity (bigger cap → fewer parts needed).

**This problem = Capacity To Ship with the story stripped off.** Recognizing that in ~30 seconds is the transferable skill.

Concretely, `[7,2,5,10,8]`, cap=18: `[7,2,5][10,8]` → 2 parts ✓; cap=17: `[7,2,5][10][8]` → 3 parts ✗. Boundary: 18.

**Predicate:** `feasible(cap) = greedyParts(cap) <= k`, space = `[max(nums), sum(nums)]`.

**Time:** O(n · log(sum))
**Space:** O(1)

```java
public int splitArray(int[] nums, int k) {
    long lo = 0, hi = 0;
    for (int x : nums) { lo = Math.max(lo, x); hi += x; }   // long: sums can overflow int

    while (lo < hi) {
        long mid = lo + (hi - lo) / 2;
        if (partsNeeded(nums, mid) <= k) hi = mid;
        else lo = mid + 1;
    }
    return (int) lo;
}

private int partsNeeded(int[] nums, long cap) {
    int parts = 1;
    long sum = 0;
    for (int x : nums) {
        if (sum + x > cap) { parts++; sum = 0; }
        sum += x;
    }
    return parts;
}
```

**Tips:**
- `long` for lo/hi/sum — total sum can exceed int range depending on constraints.
- **Volunteer the DP comparison unprompted** (phone-screen protocol): "There's an O(k·n²) DP over split points, but binary-search-on-answer gets O(n log sum) — the DP is worth mentioning because it works even when greedy feasibility doesn't."

**Trick:** The family — Split Array (parts ≤ k), Ship Packages (days ≤ D), Book Allocation, Painter's Partition — are ALL the same problem: *minimize the max of a contiguous partition*. One `partsNeeded` to rule them all.

---

## Minimum Number of Days to Make m Bouquets (LC 1482)

**Description:** `bloomDay[i]` = day flower `i` blooms. A bouquet needs `k` ADJACENT bloomed flowers. Find the minimum day to make `m` bouquets, or −1.

**Example:** `bloomDay = [1,10,3,10,2], m = 3, k = 1 → 3`

**Brute force:** Simulate each candidate day: O(max(bloomDay) · n).

**Intuition:** Waiting longer only blooms MORE flowers → any bouquet set achievable by day `d` is achievable by day `d+1` → monotone → binary search days.

`feasible(day)`: one pass counting a **run of consecutive bloomed flowers**; every time the run hits `k`, bank a bouquet and reset the run. An unbloomed flower resets the run to 0.

Concretely `[1,10,3,10,2], m=3, k=1`, day=3: bloomed mask `[T,F,T,F,T]` → runs of length 1 at positions 0,2,4 → 3 bouquets ✓. Day=2: `[T,F,F,F,T]` → 2 bouquets ✗.

**Predicate:** `feasible(day) = bouquets(day) >= m`, space = `[min(bloomDay), max(bloomDay)]`.

**Time:** O(n · log(max(bloomDay)))
**Space:** O(1)

```java
public int minDays(int[] bloomDay, int m, int k) {
    if ((long) m * k > bloomDay.length) return -1;   // long! m,k up to 1e6 each

    int lo = Integer.MAX_VALUE, hi = 0;
    for (int d : bloomDay) { lo = Math.min(lo, d); hi = Math.max(hi, d); }

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (bouquets(bloomDay, mid, k) >= m) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}

private int bouquets(int[] bloomDay, int day, int k) {
    int made = 0, run = 0;
    for (int d : bloomDay) {
        if (d <= day) {
            run++;
            if (run == k) { made++; run = 0; }   // bank a bouquet, reset run
        } else {
            run = 0;                              // gap breaks adjacency
        }
    }
    return made;
}
```

**Tips:**
- The `-1` impossibility check MUST use `(long) m * k` — the classic silent int overflow.
- The run/bank/reset structure is three separate concerns (extend run, bank bouquet, break run). Keep them as three visible branches.

**Trick:** "Adjacent/consecutive + threshold" feasibility checks recur (e.g., Maximum Beauty variants). The run-counter is your reusable sub-pattern.

---

## Cutting Ribbons (LC 1891) — the MAXIMIZE direction

**Description:** Ribbons of given lengths; cut them into pieces of equal length `L` (leftover discarded). Find the max `L` such that you get at least `k` pieces, or 0.

**Example:** `ribbons = [9,7,5], k = 4 → 4` (pieces: 9→2, 7→1, 5→1)

**Brute force:** Try each L from max down: O(max · n).

**Intuition:** Now the predicate runs `T T T F F F` — small lengths give many pieces (feasible), large lengths give few. We want the LAST true → flipped template with **ceiling mid**.

- `pieces(L) = Σ floor(ribbonᵢ / L)`
- `feasible(L) = pieces(L) >= k` — monotone DECREASING in L.

Concretely `[9,7,5], k=4`: L=4 → 2+1+1=4 ✓; L=5 → 1+1+1=3 ✗. Last true: 4.

**Predicate:** `feasible(L) = Σ ⌊rᵢ/L⌋ >= k`, space = `[1, max(ribbons)]`, maximize.

**Time:** O(n · log(max))
**Space:** O(1)

```java
public int maxLength(int[] ribbons, int k) {
    int lo = 1, hi = 0;
    for (int r : ribbons) hi = Math.max(hi, r);

    // possible that even L=1 fails -> answer 0
    if (pieces(ribbons, 1) < k) return 0;

    while (lo < hi) {
        int mid = lo + (hi - lo + 1) / 2;    // CEIL because lo = mid below
        if (pieces(ribbons, mid) >= k) lo = mid;    // works: try longer
        else hi = mid - 1;                          // too long: shrink
    }
    return lo;
}

private long pieces(int[] ribbons, int L) {
    long count = 0;
    for (int r : ribbons) count += r / L;
    return count;
}
```

**Tips:**
- **The ceil-mid pairing rule from Part 0 fires here.** `lo = mid` requires `mid = lo + (hi - lo + 1) / 2`, or `[4,5]` loops forever. Dry-run a 2-element range every time you write the maximize variant.
- Alternative that avoids the flipped template entirely: search for the *minimize* boundary of the negated predicate (`pieces(L) < k`) and return `boundary - 1`. Pick whichever you can write without thinking.

**Trick:** Maximize-the-minimum problems (Magnetic Force Between Two Balls, Divide Chocolate) are this same flipped shape: `feasible(gap/sweetness)` = greedy check, keep the largest true. If you can flip Koko → Ribbons fluently, that whole family is unlocked.

**The Tier 2 map:**

| minimize the max (floor mid) | maximize the min (ceil mid) |
|---|---|
| Koko, Ship Packages, Split Array, Bouquets (min day) | Cutting Ribbons, Magnetic Force, Divide Chocolate |

---
# Tier 3: Counting Predicate over a Virtual Sorted Space

Same skeleton as Tier 2, new predicate flavor: **`feasible(x) = count(elements <= x) >= k`**. The "sorted array" you're searching is implicit — too big or too structured to materialize. You never sort; you only need to COUNT efficiently.

---

## Kth Smallest Element in a Sorted Matrix (LC 378)

**Description:** `n × n` matrix, each row and column sorted ascending. Find the kth smallest element.

**Example:** `matrix = [[1,5,9],[10,11,13],[12,13,15]], k = 8 → 13`

**Brute force:** Dump all n² values, sort: O(n² log n). Better brute: min-heap over row frontiers: O(k log n) — the "merge k sorted lists" view. Worth mentioning as the alternative.

**Intuition:** Binary search the VALUE range `[matrix[0][0], matrix[n-1][n-1]]`, not positions.

> `feasible(x) = countLE(x) >= k` — "at least k elements are ≤ x". Monotone: raising x can only include more elements. The boundary is the smallest x with count ≥ k — and that boundary is always an actual matrix element (proof sketch: if x weren't in the matrix, x−1 would have the same count, contradicting x being the *smallest* feasible).

`countLE(x)` in O(n) via the **staircase walk**: start bottom-left. If `matrix[r][c] <= x`, the whole column above is ≤ x too → add `r+1`, step right. Else step up. Each step discards a row or column.

Concretely, x = 13 on the example: col 0 → 12 ≤ 13, add 3, right; col 1 → 13 ≤ 13, add 3, right; col 2 → 15 > 13 up, 13 ≤ 13 add 2. countLE(13) = 8 ≥ 8 ✓. countLE(12) = 6 < 8 ✗. Boundary: 13. ✓

**Predicate:** `countLE(x) >= k`, space = value range.

**Time:** O(n · log(maxVal − minVal))
**Space:** O(1)

```java
public int kthSmallest(int[][] matrix, int k) {
    int n = matrix.length;
    int lo = matrix[0][0], hi = matrix[n - 1][n - 1];

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (countLE(matrix, mid) >= k) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}

private int countLE(int[][] matrix, int x) {
    int n = matrix.length;
    int r = n - 1, c = 0, count = 0;     // bottom-left corner
    while (r >= 0 && c < n) {
        if (matrix[r][c] <= x) { count += r + 1; c++; }  // whole column counts
        else r--;
    }
    return count;
}
```

**Tips:**
- The candidate `mid` need not exist in the matrix mid-search — that's fine; convergence lands on a real element. Don't add an "is it present?" check.
- The staircase walk is the same walker as Search a 2D Matrix II (LC 240) — one pointer trick, two problems.

**Trick:** Find K-th Smallest Pair Distance (LC 719) is the same shape: binary search the distance value, `countLE(d)` via two pointers over the sorted array. "Binary search the value, count with a linear structure walk" is the whole tier.

---

# Tier 4: Partition Binary Search

A different species: binary search a **cut position**, with a two-sided invariant instead of a one-sided predicate.

---

## Median of Two Sorted Arrays (LC 4)

**Description:** Two sorted arrays sizes m, n; find the median of the merged whole in O(log(m+n)).

**Example:** `A = [1,3], B = [2] → 2.0`;  `A = [1,2], B = [3,4] → 2.5`

**Brute force:** Merge (O(m+n) time/space), or two-pointer walk to the middle without storing (O(m+n) time, O(1) space). **Say this out loud before coding the log version** — it's the protocol, and it's insurance: if the cut search falls apart, a correct answer is still on the board.

---

### Intuition, built in five steps

**Step 1 — The median is defined by a SPLIT, not by a search.**

Forget "find the middle element." The median is whatever sits at the boundary when the merged array is cut into a left half and a right half such that:

1. left half has exactly `half = (m + n + 1) / 2` elements, and
2. everything on the left ≤ everything on the right.

If you can produce that split without merging, you can read the median off its edges.

**Step 2 — A split has ONE degree of freedom, not two.**

The left half's elements come from only two places: the front of A and the front of B. If you take `i` from A, the rest must come from B:

```
i  +  j  =  half        →        j = half − i
```

That's not an algorithm trick — it's just "the pieces add up to the total." Pick `i`, and `j` is **forced**. Two knobs would mean an O(m·n) search; the size constraint couples them into one knob → one binary search → O(log m).

> **Interview line:** *"The cut has one degree of freedom, not two, because the left half's size is fixed."*

**`i` and `j` move in opposite directions** — increasing `i` by 1 automatically decreases `j` by 1. A seesaw, not two independent cuts:

```
i = 1:   A: 1 | 3  8            B: 7  9  10 | 11
i = 2:   A: 1  3 | 8            B: 7  9 | 10  11
i = 3:   A: 1  3  8 |           B: 7 | 9  10  11
         ────────────→                ←────────────
         A's cut moves right      B's cut moves left
```

The left half always holds 4 elements. You're only choosing *how many of them come from A*.

**Step 3 — Four border values are enough to validate a cut.**

```
       aLeft ↓ ↓ aRight
   A:  1   3   |   8
   B:  7       |   9   10   11
       bLeft ↑ ↑ bRight
```

| name | meaning | index |
|---|---|---|
| `aLeft`  | last element on A's left side | `A[i-1]` |
| `aRight` | first element on A's right side | `A[i]` |
| `bLeft`  | last element on B's left side | `B[j-1]` |
| `bRight` | first element on B's right side | `B[j]` |

The indexing follows from what `i` means — **`i` = how many elements taken from A**. So A's left is `A[0 .. i-1]` (last = `A[i-1]`) and A's right starts at `A[i]`.

"Every left element ≤ every right element" is a lot of comparisons — but sortedness makes most free:

- `aLeft <= aRight` — **free**, A is sorted
- `bLeft <= bRight` — **free**, B is sorted
- `aLeft <= bRight` — **must check** (cross-array)
- `bLeft <= aRight` — **must check** (cross-array)

And borders suffice: `aLeft` is the **largest** thing on A's left; `bRight` is the **smallest** thing on B's right. If largest-left ≤ smallest-right, everything behind them follows. **Four numbers stand in for the whole array.**

**Step 4 — Failure tells you which way to slide.**

A cut is valid iff **both**:
```
(1)  aLeft <= bRight
(2)  bLeft <= aRight
```

Exactly one can fail, and *which* one names the direction:

- **(2) fails — `bLeft > aRight`:** an element on B's left is bigger than one on A's right. Wrong element is on the left → evict it → **shrink B's share** → decrease `j` → **increase `i`** → `lo = i + 1`.
- **(1) fails — `aLeft > bRight`:** mirror image → **decrease `i`** → `hi = i - 1`.

> **Mnemonic:** *whichever side's Left is misbehaving, take less from that side.* `aLeft` too big → take less from A (`i` down). `bLeft` too big → take less from B, which forces taking **more** from A (`i` up).

**Why this is a legal binary search:** each condition is monotone in `i`. As `i` grows, `aLeft`/`aRight` only grow (reaching further into sorted A) while `j` shrinks, so `bLeft`/`bRight` only shrink:

```
i:              0     1     2     3
condition (2)   F     F     F     T        bLeft <= aRight  — turns true as i grows
condition (1)   T     T     T     T        aLeft <= bRight  — turns false as i grows
                                  ↑
                            both true = the answer
```

Two monotone predicates moving in **opposite** directions, overlapping in exactly one place. Same seam machinery as everywhere else — just pinned from both sides. (This is also why both can never fail at once: they'd have to disagree about which way to move.)

**Step 5 — Read the median off the borders.**

Two facts, both from sortedness:
- **left half's max = `max(aLeft, bLeft)`** — the biggest thing on the left must be one of the two borders; everything else is behind them.
- **right half's min = `min(aRight, bRight)`** — mirror.

*Odd total →* `half = (m+n+1)/2` puts the **extra element on the left**, so the halves are uneven and the median lands *inside* the left half, at its right edge:

```
7 elements, half = 4:
merged:      1    3    7    8   |   9    10   11
position:    1    2    3    4       5    6    7
             └──── 4 left ────┘   └── 3 right ──┘
                            ↑
                    median = 4th = left half's max = max(aLeft, bLeft)
```
The right half starts at position 5 — **past** the median. `aRight`/`bRight` are irrelevant here; they only validated the cut.

*Even total →* halves are equal, so the median falls in the **gap** between them — need one value from each side:

```
4 elements, half = 2:
merged:      1    2   |   3    4
position:    1    2       3    4
                  ↑       ↑
             left max   right min   → (2 + 3) / 2.0 = 2.5
```

---

### Full dry run: `A = [1,3,8]`, `B = [7,9,10,11]`

A is smaller (3 ≤ 4) → no swap. `m=3, n=4`, total = 7 (odd), `half = (3+4+1)/2 = 4`. Search `i ∈ [0, 3]`.

**Iteration 1:** lo=0, hi=3 → **i = 1**, j = 4−1 = **3**
```
A:  1 | 3  8            B:  7  9  10 | 11
```
| | value | why |
|---|---|---|
| aLeft | `A[0]` = **1** | i≠0 |
| aRight | `A[1]` = **3** | i≠m |
| bLeft | `B[2]` = **10** | j≠0 |
| bRight | `B[3]` = **11** | j≠n |

`aLeft(1) ≤ bRight(11)` ✓ but `bLeft(10) ≤ aRight(3)` ✗ → **invalid**. `bLeft` misbehaving → took **too few** from A → `lo = 2`

**Iteration 2:** lo=2, hi=3 → **i = 2**, j = **2**
```
A:  1  3 | 8            B:  7  9 | 10  11
```
| aLeft | aRight | bLeft | bRight |
|---|---|---|---|
| `A[1]` = **3** | `A[2]` = **8** | `B[1]` = **9** | `B[2]` = **10** |

`3 ≤ 10` ✓ but `bLeft(9) ≤ aRight(8)` ✗ → **invalid**. Still too few from A → `lo = 3`

**Iteration 3:** lo=3, hi=3 → **i = 3**, j = **1**
```
A:  1  3  8 |           B:  7 | 9  10  11
```
| | value | why |
|---|---|---|
| aLeft | `A[2]` = **8** | i≠0 |
| aRight | **+∞** | **i == m → sentinel** |
| bLeft | `B[0]` = **7** | j≠0 |
| bRight | `B[1]` = **9** | j≠n |

`aLeft(8) ≤ bRight(9)` ✓ **and** `bLeft(7) ≤ aRight(+∞)` ✓ → **VALID**

Left half = {1,3,8} ∪ {7} = 4 = `half` ✓. Odd → `max(aLeft, bLeft)` = `max(8,7)` = **8**
Verify: merged = `[1,3,7,8,9,10,11]`, 4th element = 8 ✓

*Self-check when writing this cold: at iteration 3 your `aRight` must be `+∞`. If it isn't, the sentinel condition is backwards.*

---

### The three edge cases

**`A = [1,2], B = [3,4]`** — even total, two sentinels fire at once. `half = 2`

| iter | i | j | aLeft | aRight | bLeft | bRight | verdict |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 1 | 1 | 2 | 3 | 4 | `bLeft(3) ≤ aRight(2)` ✗ → lo = 2 |
| 2 | 2 | 0 | 2 | **+∞** | **−∞** | 3 | both ✓ → valid |

Even → `(max(2,−∞) + min(+∞,3)) / 2.0` = **2.5** ✓
Note iter 2: `i=m` **and** `j=0` — two sentinels, still zero branches.

**`A = [], B = [1]`** — empty array. `m=0, n=1, half=1`. One iteration: **i=0**, j=**1**

| | value | why |
|---|---|---|
| aLeft | **−∞** | i == 0 |
| aRight | **+∞** | i == m (here 0 == m, so *both* A sentinels) |
| bLeft | `B[0]` = **1** | |
| bRight | **+∞** | j == n |

`−∞ ≤ +∞` ✓, `1 ≤ +∞` ✓ → valid. Odd → `max(−∞, 1)` = **1** ✓
The empty array contributes nothing and needs **zero** special-casing — it's just `−∞ | +∞`.

**`A = [1,3], B = [2]`** — **the swap fires first.** `A.length(2) > B.length(1)` → swap → `A=[2], B=[1,3]`. `m=1, n=2, half=2`

| iter | i | j | aLeft | aRight | bLeft | bRight | verdict |
|---|---|---|---|---|---|---|---|
| 1 | 0 | 2 | **−∞** | 2 | 3 | **+∞** | `bLeft(3) ≤ aRight(2)` ✗ → lo = 1 |
| 2 | 1 | 1 | 2 | **+∞** | 1 | 3 | both ✓ → valid |

Odd → `max(2,1)` = **2** ✓

---

### `lo`/`hi` are NOT cuts — they're the search range FOR the cut

The #1 confusion on this problem. Two different levels:

- **`i`** = the actual cut in A. **This is the `mid`.** It's named `i` only because it indexes A.
- **`lo`, `hi`** = the range of `i` values still worth trying.

It's the identical structure to every other binary search you've written:

| Koko | Median |
|---|---|
| searching for a **speed** `k` | searching for a **cut** `i` |
| `lo=1, hi=max(piles)` = range of speeds | `lo=0, hi=m` = range of cuts |
| `mid` = the speed being tested | `i` = the cut being tested |
| `feasible(mid)` → move lo or hi | cut valid? → move lo or hi |

**Why `hi = m`, not `m − 1`:** `i` counts elements *taken*, and taking all `m` is a legal cut. Cuts live in the **gaps**, and `m` elements have `m+1` gaps:

```
i = 0  →  A: | 1  3  8        take nothing
i = 1  →  A: 1 | 3  8
i = 2  →  A: 1  3 | 8
i = 3  →  A: 1  3  8 |        take all of A
```

Same reasoning as **lower bound's `hi = n`**: "past the end" must be representable. Set `hi = m−1` and iteration 3 of the main trace becomes unreachable.

**What each setup line buys:**
- **the swap** → makes `j` structurally safe (never outside `[0,n]`) — that's the *real* reason, not speed
- **`half`** → makes `j` determined
- **sentinels** → make the extremes of `j` harmless

Three lines of setup, and then the loop only ever thinks about `i`.

---

**Predicate (implicit):** `bLeft <= aRight` steers the cut; the loop exits by *returning* when both conditions hold. This is **Style 3a** (`lo <= hi` + return-early) — NOT the converging skeleton. Don't graft.

**Time:** O(log(min(m, n)))
**Space:** O(1)

```java
public double findMedianSortedArrays(int[] A, int[] B) {
    if (A.length > B.length) return findMedianSortedArrays(B, A); // search the SMALLER

    int m = A.length, n = B.length;
    int half = (m + n + 1) / 2;
    int lo = 0, hi = m;                      // i = how many taken from A: [0, m]

    while (lo <= hi) {
        int i = lo + (hi - lo) / 2;          // i IS the mid
        int j = half - i;                    // forced by the size constraint

        int aLeft  = (i == 0) ? Integer.MIN_VALUE : A[i - 1];
        int aRight = (i == m) ? Integer.MAX_VALUE : A[i];
        int bLeft  = (j == 0) ? Integer.MIN_VALUE : B[j - 1];
        int bRight = (j == n) ? Integer.MAX_VALUE : B[j];

        if (aLeft <= bRight && bLeft <= aRight) {          // correct cut
            if ((m + n) % 2 == 1)
                return Math.max(aLeft, bLeft);             // odd: right side irrelevant
            return (Math.max(aLeft, bLeft) + Math.min(aRight, bRight)) / 2.0;
        } else if (aLeft > bRight) {
            hi = i - 1;                                     // aLeft misbehaving: less from A
        } else {
            lo = i + 1;                                     // bLeft misbehaving: more from A
        }
    }
    throw new IllegalStateException();   // unreachable for valid input
}
```

**Tips:**
- **Search the smaller array.** Since `i ∈ [0,m]`, `j = half − i` ranges over `[half−m, half]`. When `m ≤ n`, `half = (m+n+1)/2 ≥ m` so `half−m ≥ 0` ✓ and `half ≤ n` ✓. Search the *larger* and `j` goes negative → index out of bounds. Speed is the side effect, not the reason.
- **Name all four borders on their own lines** before comparing. Do NOT inline `A[i-1]` into the `if`. This problem is index-arithmetic-under-pressure in its purest form, and inlining is exactly where transposition errors live.
- The ±∞ **sentinels replace four edge cases with zero branches.** `i == m` → `aRight = +∞` = "nothing left in A to compare against" → condition passes trivially, which is correct.
- **`2.0`, not `2`.** Integer division silently truncates 2.5 → 2. No crash, just a wrong answer.
- **The odd branch never touches `aRight`/`bRight`.** If you reach for them, the parity is backwards — that's the tell.
- **Parity is checked on `(m + n)`, the total** — not on `half`, not on either array.
- Dry-run set (all three, no exceptions): `A=[], B=[1]` · `A=[1,2], B=[3,4]` · `A=[1,3], B=[2]`.

**Trick (follow-up):** "Kth smallest of two sorted arrays" — same cut search with `half = k`. The correct cut's answer is `max(aLeft, bLeft)`. Median is just k = middle.

**Trick (the `+1` is a convention, not a law):** `half = (m+n)/2` also works — the split becomes 3/4 instead of 4/3, and the odd median becomes `min(aRight, bRight)` (first element of the *right* half). The `+1` version is standard because it lets the even and odd branches share the `max(aLeft, bLeft)` term.

---

# Tier 5: Binary Search as a Subroutine

Binary search isn't the algorithm here — it's the O(log n) lookup step inside a larger design.

---

## Random Pick with Weight (LC 528)

**Description:** `w[i]` = weight of index i. `pickIndex()` returns i with probability `w[i] / sum(w)`.

**Example:** `w = [1,3]` → pickIndex returns 0 with prob 1/4, 1 with prob 3/4.

**Brute force:** Materialize an array with `w[i]` copies of each index, pick uniformly: O(sum(w)) space — dies on large weights.

**Intuition:** Lay the weights out as segments on a number line via **prefix sums**:

`w = [1,3] → prefix = [1,4]` → index 0 owns targets {1}, index 1 owns {2,3,4}.

Draw a uniform target in `[1, totalSum]`, then find **the first prefix ≥ target** — lower bound, straight from Tier 0. Segment lengths are proportional to weights, so probabilities come out right.

Concretely: target 1 → lowerBound over `[1,4]` → index 0 (prob 1/4). Targets 2,3,4 → index 1 (prob 3/4). ✓

**Predicate:** `prefix[i] >= target`.

**Time:** O(n) build, O(log n) per pick
**Space:** O(n)

```java
class Solution {
    private final int[] prefix;
    private final Random rand = new Random();

    public Solution(int[] w) {
        prefix = new int[w.length];
        int sum = 0;
        for (int i = 0; i < w.length; i++) {
            sum += w[i];
            prefix[i] = sum;
        }
    }

    public int pickIndex() {
        int target = rand.nextInt(prefix[prefix.length - 1]) + 1;  // uniform in [1, total]
        int lo = 0, hi = prefix.length - 1;      // answer guaranteed to exist
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (prefix[mid] >= target) hi = mid;
            else lo = mid + 1;
        }
        return lo;
    }
}
```

**Tips:**
- `nextInt(total) + 1` gives `[1, total]`. Using `[0, total-1]` with `>=` off by one silently skews probabilities — the bug won't crash, it will just be *wrong*, which is worse.
- Very common actual Google phone-screen problem. Practice the 60-second version.

**Trick:** Follow-up is usually "weights update frequently" → prefix sums become O(n) per update → segue to Binary Indexed Tree, or (your toolkit) a `TreeMap<Long, Integer>` of cumulative-weight → index.

---

## Time Based Key-Value Store (LC 981)

**Description:** `set(key, value, timestamp)` and `get(key, timestamp)` → the value with the largest timestamp ≤ the query timestamp ("", if none). Timestamps in `set` are strictly increasing per key.

**Example:** set("foo","bar",1) → get("foo",1)="bar", get("foo",3)="bar"; set("foo","bar2",4) → get("foo",4)="bar2", get("foo",5)="bar2".

**Brute force:** Per key, scan all versions: O(versions) per get.

**Intuition:** Per key, versions arrive sorted by timestamp → each key holds a sorted list → "largest timestamp ≤ t" is a **floor** query = predecessor search.

Two idiomatic Java implementations — know BOTH:

**(a) TreeMap version** — `floorEntry(t)` IS the floor query. Three lines of logic. This is the TreeMap-as-segment-tree-alternative pattern from your toolkit.

**(b) ArrayList + manual binary search** — what the interviewer usually wants to SEE. Floor = "last index with ts ≤ t" = a maximize-direction search (ceil-mid pairing from Part 0!), or equivalently `lowerBound(t+1) − 1`.

**Predicate:** floor query: largest i with `ts[i] <= t`.

**Time:** set O(1) amortized / O(log n) TreeMap; get O(log n)
**Space:** O(total sets)

```java
// (a) TreeMap version
class TimeMap {
    private final Map<String, TreeMap<Integer, String>> map = new HashMap<>();

    public void set(String key, String value, int timestamp) {
        map.computeIfAbsent(key, k -> new TreeMap<>()).put(timestamp, value);
    }

    public String get(String key, int timestamp) {
        TreeMap<Integer, String> tm = map.get(key);
        if (tm == null) return "";
        Map.Entry<Integer, String> e = tm.floorEntry(timestamp);
        return e == null ? "" : e.getValue();
    }
}
```

```java
// (b) ArrayList + manual floor search
class TimeMap {
    private final Map<String, List<int[]>> ts = new HashMap<>();       // {timestamp}
    private final Map<String, List<String>> vals = new HashMap<>();

    public void set(String key, String value, int timestamp) {
        ts.computeIfAbsent(key, k -> new ArrayList<>()).add(new int[]{timestamp});
        vals.computeIfAbsent(key, k -> new ArrayList<>()).add(value);
    }

    public String get(String key, int timestamp) {
        List<int[]> t = ts.get(key);
        if (t == null || t.get(0)[0] > timestamp) return "";

        int lo = 0, hi = t.size() - 1;
        while (lo < hi) {
            int mid = lo + (hi - lo + 1) / 2;          // CEIL: lo = mid below
            if (t.get(mid)[0] <= timestamp) lo = mid;  // valid floor candidate
            else hi = mid - 1;
        }
        return vals.get(key).get(lo);
    }
}
```

**Tips:**
- Say the trade-off out loud: TreeMap = simpler + supports out-of-order timestamps; ArrayList = exploits the sorted-input guarantee for O(1) set. Choosing based on stated guarantees is a signal interviewers score.
- Snapshot Array (LC 1146) is the same floor-query idea per array slot: list of `(snapId, value)` pairs, floor search on `snap_id`.

**Trick:** Floor/ceiling queries over dynamic sorted data = `TreeMap.floorEntry / ceilingEntry` — the same tool backing your planned interval problems. This entry doubles as that revision.

---

# Cheat Sheet

| Shape | Recognize by | Predicate | Space |
|---|---|---|---|
| Lower bound | "first ≥", "insertion point", count occurrences | `a[i] >= t` | indices `[0, n]` |
| Rotated | one seam in sorted data | "which half is sorted?" invariant | indices |
| Structural | peak, single element | slope / parity misalignment | indices (maybe derived) |
| BS on answer (min) | "min speed/capacity such that..." | greedy check `<= budget` | answer values, floor mid |
| BS on answer (max) | "max length/gap such that..." | greedy check `>= k` | answer values, **ceil mid** |
| Counting | "kth smallest" over huge implicit set | `countLE(x) >= k` | value range |
| Partition | median / kth of two sorted arrays | two-sided border invariant | cut positions |
| Subroutine | floor/predecessor lookups in a design | `prefix[i] >= target`, `ts[i] <= t` | indices / TreeMap |

**Interview one-liners to volunteer:**
- "The predicate is monotonic because extra capacity/time/slack never hurts — so binary search applies."
- "Brute force checks every candidate answer linearly; the F...FT...T structure lets us binary search it."
- "I'll dry-run on a 2-element range to verify the mid-bias pairing."
