# Heap — Problem Notes

Ordered for learning flow. Each section is one transferable idea; each problem escalates the previous one.

| # | Section | Transferable idea |
|---|---|---|
| 1 | Top-K | Min-heap of size k = "team of k"; root is the bar for entry |
| 2 | Two heaps & streams | Two heaps facing each other expose the middle; lazy deletion tolerates ghosts |
| 3 | K-way merge | Heap of frontier candidates, one per source; pop best, push its successor |
| 4 | Greedy scheduling | Heap = "best runnable thing right now" + simulate time |
| 5 | Heap-as-regret | Commit greedily, keep past choices in a heap so you can retroactively undo the worst |
| 6 | Sort one dim, heap the other | Two-variable optimization: sort fixes one variable, heap controls the other |
| 7 | Design + lazy deletion | Never update the heap — add new facts, validate on pop against a source of truth |
| 8 | Delayed decisions | Don't decide until forced; store options in an ordered structure until then |
| 9 | Boss | Multi-pattern crossover |

---

## Section 1 — Top-K

### 1. Top K Frequent Words
**Link:** https://leetcode.com/problems/top-k-frequent-words/

**Explanation:** Given words and k, return the k most frequent words, ordered by frequency desc; ties broken lexicographically asc.

**Intuition:** Team-of-k pattern: keep a min-heap of size k where the root is the *weakest current member*. Each word asks "am I better than the weakest?" — pushing then popping-if-over-k answers that automatically. The twist here is entirely in the comparator: "weakest" = lowest frequency, and among frequency ties, the lexicographically *largest* word (because the larger word should be evicted first).

**Brute force:** Count, then sort all d distinct words by (freq desc, word asc) — O(d log d).

**Optimization:** Heap capped at size k → O(d log k). Matters when k << d.

**How the heap helps:** It maintains "the k best so far" incrementally, with the eviction candidate always at the root — no full ordering ever computed.

```java
public List<String> topKFrequent(String[] words, int k) {
    Map<String, Integer> freq = new HashMap<>();
    for (String w : words) freq.merge(w, 1, Integer::sum);

    // Min-heap by "strength": root = weakest word currently in the top k.
    // Weakest = lowest count; tie -> lexicographically LARGEST (evicted first).
    PriorityQueue<String> heap = new PriorityQueue<>((a, b) ->
        freq.get(a).equals(freq.get(b))
            ? b.compareTo(a)
            : Integer.compare(freq.get(a), freq.get(b)));

    for (String w : freq.keySet()) {
        heap.offer(w);
        if (heap.size() > k) heap.poll();          // evict the weakest
    }

    // Heap yields weakest-first -> fill the answer back to front.
    String[] res = new String[k];
    for (int i = k - 1; i >= 0; i--) res[i] = heap.poll();
    return Arrays.asList(res);
}
```

**Time:** O(n) counting + O(d log k) heap, d = distinct words
**Space:** O(d) map + O(k) heap

**Tips & tricks:**
- The tie-break is *inverted* inside a min-heap: answer wants ties lex-ASC, so the heap must treat lex-LARGER as weaker. Getting this backwards is the #1 bug.
- Output must be extracted by repeated `poll()` (reverse order) — heap iteration order is meaningless.
- Follow-up to volunteer: bucket sort by frequency gives O(n + d log d_bucket) and is the true O(n)-ish alternative.

---

## Section 2 — Two Heaps & Streams

### 2. Find Median from Data Stream
**Link:** https://leetcode.com/problems/find-median-from-data-stream/

**Explanation:** Design a class supporting `addNum(num)` and `findMedian()` over a growing stream.

**Intuition:** A heap exposes one end; the median lives in the middle. So cut the data at the median and point two heaps at each other: `lo` = max-heap of the small half, `hi` = min-heap of the large half. The two roots are pressed against the "fence" — the median is always one of them (or their average). Invariants: (1) every lo element <= every hi element; (2) sizes differ by <= 1 (lo may hold the extra).

**Brute force:** Keep a sorted list; insert O(n), median O(1) → O(n) per add.

**Optimization:** Two heaps → O(log n) per add, O(1) per median.

**How the heap helps:** Each half only ever needs to surrender its boundary element (largest-of-small / smallest-of-large) — exactly what a heap root is.

```java
class MedianFinder {
    private final PriorityQueue<Integer> lo = new PriorityQueue<>(Comparator.reverseOrder()); // small half
    private final PriorityQueue<Integer> hi = new PriorityQueue<>();                          // large half

    public void addNum(int num) {
        lo.offer(num);
        hi.offer(lo.poll());              // push-through: largest of small side crosses the fence
        if (hi.size() > lo.size())
            lo.offer(hi.poll());          // restore balance; lo owns the odd extra
    }

    public double findMedian() {
        return lo.size() > hi.size()
            ? lo.peek()
            : ((double) lo.peek() + hi.peek()) / 2.0;
    }
}
```

**Time:** O(log n) per add, O(1) per median
**Space:** O(n)

**Tips & tricks:**
- The push-through insert (in via lo, spill root to hi, rebalance back) needs zero branches on the value — routing and rebalancing stay separate concerns.
- `/ 2.0` not `/ 2`; cast to double *before* adding roots if values can be near int limits.
- Famous follow-up: "99% of values in [0,100]" → counting buckets for the hot range.

---

### 3. Sliding Window Median
**Link:** https://leetcode.com/problems/sliding-window-median/

**Explanation:** Return the median of every window of size k as it slides across nums.

**Explanation of the new difficulty:** Same two-heap engine, but now elements *leave*, and `PriorityQueue.remove(Object)` is O(n).

**Intuition:** **Lazy deletion.** Never remove from the middle — record departures in a ghost map and skip ghosts when they surface at a root. Consequence: `heap.size()` lies, so track balance yourself. Each departure logically shrinks whichever side it belongs to (decided by comparing against `lo.peek()`), even though physically nothing moves yet.

**Brute force:** Sort each window: O(n*k log k).

**Optimization:** Two heaps + lazy deletion → O(n log n) (heaps hold up to n stale entries).

**How the heap helps:** Same fence idea as the median stream; lazy deletion is the standard trick to make heaps survive arbitrary removals.

```java
public double[] medianSlidingWindow(int[] nums, int k) {
    PriorityQueue<Integer> lo = new PriorityQueue<>(Comparator.reverseOrder()); // small half
    PriorityQueue<Integer> hi = new PriorityQueue<>();                          // large half
    Map<Integer, Integer> stale = new HashMap<>();  // value -> pending ghost count

    // Seed the first window: dump all into lo, spill half into hi.
    for (int i = 0; i < k; i++) lo.offer(nums[i]);
    for (int i = 0; i < k / 2; i++) hi.offer(lo.poll());

    double[] ans = new double[nums.length - k + 1];
    ans[0] = median(lo, hi, k);

    for (int i = k; i < nums.length; i++) {
        int in = nums[i], out = nums[i - k];
        int balance = 0;                    // net change to (live lo size - live hi size)

        // Lazy removal: record the ghost, adjust the balance only.
        balance += (out <= lo.peek()) ? -1 : +1;
        stale.merge(out, 1, Integer::sum);

        // Real insertion into the correct side.
        if (in <= lo.peek()) { lo.offer(in); balance++; }
        else                 { hi.offer(in); balance--; }

        // balance is now -2, 0, or +2 -> at most ONE live transfer fixes it.
        if (balance < 0) { prune(hi, stale); lo.offer(hi.poll()); }  // prune first: transfer a LIVE element
        if (balance > 0) { prune(lo, stale); hi.offer(lo.poll()); }

        // Roots must be live before reading the median.
        prune(lo, stale); prune(hi, stale);
        ans[i - k + 1] = median(lo, hi, k);
    }
    return ans;
}

private void prune(PriorityQueue<Integer> heap, Map<Integer, Integer> stale) {
    while (!heap.isEmpty() && stale.getOrDefault(heap.peek(), 0) > 0) {
        stale.merge(heap.peek(), -1, Integer::sum);
        heap.poll();
    }
}

private double median(PriorityQueue<Integer> lo, PriorityQueue<Integer> hi, int k) {
    return (k % 2 == 1) ? lo.peek()
                        : ((double) lo.peek() + hi.peek()) / 2.0;
}
```

**Time:** O(n log n)
**Space:** O(n) — ghosts accumulate until they reach a root

**Tips & tricks:**
- Ghosts only die at a root. Never trust `size()`; the `balance` variable is the live truth.
- Prune the *source* heap before any cross-heap transfer, or you transfer a ghost.
- `[2147483647, 2147483647]`-style tests punish int overflow in the even-median average — cast before adding.

---

### 4. Finding MK Average
**Link:** https://leetcode.com/problems/finding-mk-average/

**Explanation:** Stream + parameters m, k. Over the last m elements, drop the smallest k and largest k, return the floor-average of the rest. `addElement` and `calculateMKAverage` must both be fast.

**Intuition:** The median needed one fence; this needs **two** (bottom-k | middle | top-k), a running *sum* of the middle, and removals from arbitrary rank positions as the window slides. That combination kills heaps: you'd need lazy deletion on both sides of both fences, and the middle bucket must give away its min AND max. Level-0 fork: arbitrary removal + both ends → **TreeMap multisets**, not heaps. Keep a deque for arrival order — the deque answers WHO leaves (time), the TreeMaps answer WHERE everyone ranks (value). Two questions, two structures; don't fuse them.

**Brute force:** Re-sort last m on every calculate: O(m log m) per query.

**Optimization:** Three TreeMap multisets + running midSum → O(log m) per add, O(1) per query.

**How the heap helps:** It doesn't — and recognizing that is the lesson. This problem is the boundary marker for when heaps lose to ordered maps.

```java
class MKAverage {
    private final int m, k;
    private long midSum = 0;                                       // long: 1e5 elements x 1e5 values
    private final Deque<Integer> window = new ArrayDeque<>();      // arrival order: WHO leaves
    private final TreeMap<Integer, Integer> bot = new TreeMap<>(); // value -> count multisets
    private final TreeMap<Integer, Integer> mid = new TreeMap<>();
    private final TreeMap<Integer, Integer> top = new TreeMap<>();
    private int botN = 0, midN = 0, topN = 0;                      // element counts per bucket

    public MKAverage(int m, int k) { this.m = m; this.k = k; }

    public void addElement(int num) {
        window.offer(num);

        // 1. Route in by comparing against the two fences.
        if (botN > 0 && num <= bot.lastKey())        { add(bot, num); botN++; }
        else if (topN > 0 && num >= top.firstKey())  { add(top, num); topN++; }
        else                                         { add(mid, num); midN++; midSum += num; }

        // 2. Evict the oldest once the window overflows. Ties across the fence are
        //    fungible: removing "the wrong copy" is fixed by rebalancing below.
        if (window.size() > m) {
            int old = window.poll();
            if (botN > 0 && old <= bot.lastKey())        { remove(bot, old); botN--; }
            else if (topN > 0 && old >= top.firstKey())  { remove(top, old); topN--; }
            else                                         { remove(mid, old); midN--; midSum -= old; }
        }

        // 3. Rebalance: SPILL overfull edges into the middle first, THEN FILL starving
        //    edges from the middle. Every migration crosses a fence AT the fence,
        //    so the order invariant survives automatically.
        while (botN > k)             { int v = bot.lastKey();  remove(bot, v); botN--; add(mid, v); midN++; midSum += v; }
        while (topN > k)             { int v = top.firstKey(); remove(top, v); topN--; add(mid, v); midN++; midSum += v; }
        while (botN < k && midN > 0) { int v = mid.firstKey(); remove(mid, v); midN--; midSum -= v; add(bot, v); botN++; }
        while (topN < k && midN > 0) { int v = mid.lastKey();  remove(mid, v); midN--; midSum -= v; add(top, v); topN++; }
    }

    public int calculateMKAverage() {
        if (window.size() < m) return -1;
        return (int) (midSum / (m - 2 * k));       // integer division = floor, for free
    }

    // Multiset helpers — delete-at-zero or firstKey()/lastKey() read phantom keys.
    private void add(TreeMap<Integer, Integer> ms, int v)    { ms.merge(v, 1, Integer::sum); }
    private void remove(TreeMap<Integer, Integer> ms, int v) { if (ms.merge(v, -1, Integer::sum) == 0) ms.remove(v); }
}
```

**Time:** O(log m) per addElement (constant number of TreeMap ops), O(1) per calculate
**Space:** O(m)

**Tips & tricks:**
- Spill-before-fill in rebalancing: an overfull top must dump into mid *before* a starving bottom pulls from mid, or a query can land between fixes.
- midSum tracks the bucket you *actually* touched — with duplicates at a fence there is no "really came from".
- Write the two multiset helpers first; forgetting delete-at-zero corrupts every fence comparison afterwards.

---

## Section 3 — K-way Merge

### 5. Merge k Sorted Lists
**Link:** https://leetcode.com/problems/merge-k-sorted-lists/

**Explanation:** Merge k sorted linked lists into one sorted list.

**Intuition:** The next output element is the smallest among k "frontier" candidates — the current head of each list. A min-heap holds exactly those k candidates; pop the winner, and its list sends up the next node. The heap never holds more than one node per list.

**Brute force:** Concatenate all N nodes and sort: O(N log N). Or merge lists one-by-one into an accumulator: O(N*k).

**Optimization:** Heap of k frontiers → O(N log k). (Divide-and-conquer pairwise merging matches this — worth naming as the alternative.)

**How the heap helps:** Answers "which of the k lists currently owns the global minimum?" in O(log k) instead of a k-way linear scan.

```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> heap = new PriorityQueue<>((a, b) -> Integer.compare(a.val, b.val));
    for (ListNode head : lists)
        if (head != null) heap.offer(head);        // one frontier node per non-empty list

    ListNode dummy = new ListNode(0), tail = dummy;
    while (!heap.isEmpty()) {
        ListNode node = heap.poll();               // global minimum among the k frontiers
        tail.next = node;
        tail = node;
        if (node.next != null) heap.offer(node.next);  // that list's next candidate steps up
    }
    return dummy.next;
}
```

**Time:** O(N log k), N = total nodes
**Space:** O(k) heap

**Tips & tricks:**
- Null-check when seeding — empty lists in the input array are a guaranteed test case.
- Dummy head kills the "is this the first node?" branch.
- The pop-one-push-successor loop is structurally Dijkstra without weights: same skeleton, different priority.

---

### 6. Find K Pairs with Smallest Sums
**Link:** https://leetcode.com/problems/find-k-pairs-with-smallest-sums/

**Explanation:** Given two sorted arrays, return the k pairs (u from nums1, v from nums2) with the smallest sums.

**Intuition:** View the pairs as a matrix where cell (i, j) = nums1[i] + nums2[j]; rows and columns are sorted. This is k-way merge where each *row* is a list. The trap: from (i, j), both (i, j+1) and (i+1, j) look like successors — pushing both means (i+1, j+1) enters twice via two paths and the heap explodes with duplicates. Fix: seed every row's first cell (i, 0), and only ever advance *within* a row (i, j) → (i, j+1). Each row keeps exactly one representative in the heap, so no cell is reachable twice.

**Brute force:** Generate all n1*n2 sums, sort, take k: O(n1*n2 log(n1*n2)).

**Optimization:** Frontier heap of <= min(k, n1) entries → O(k log k).

**How the heap helps:** Explores the matrix in sum order while touching only O(k) cells — the frontier discipline prevents the quadratic blow-up.

```java
public List<List<Integer>> kSmallestPairs(int[] nums1, int[] nums2, int k) {
    List<List<Integer>> res = new ArrayList<>();
    // Heap entry {i, j} = pair (nums1[i], nums2[j]); ordered by sum. Long: values to 1e9.
    PriorityQueue<int[]> heap = new PriorityQueue<>(
        (a, b) -> Long.compare((long) nums1[a[0]] + nums2[a[1]],
                               (long) nums1[b[0]] + nums2[b[1]]));

    // Seed one representative per row — pairs (i, 0). No row beyond k can matter.
    for (int i = 0; i < Math.min(k, nums1.length); i++)
        heap.offer(new int[]{i, 0});

    while (k-- > 0 && !heap.isEmpty()) {
        int[] p = heap.poll();
        res.add(Arrays.asList(nums1[p[0]], nums2[p[1]]));
        if (p[1] + 1 < nums2.length)
            heap.offer(new int[]{p[0], p[1] + 1});   // successor within the SAME row only
    }
    return res;
}
```

**Time:** O(k log k)
**Space:** O(k)

**Tips & tricks:**
- "Advance in one dimension only, seed the other" is the general dedup recipe for 2D frontier searches (also: Kth Smallest in a Sorted Matrix).
- Values up to 1e9 → sum overflows int; compare with `Long.compare`.
- Seeding is capped at min(k, n1): row k or later can never contribute to the k smallest.

---

### 7. Smallest Range Covering Elements from K Lists
**Link:** https://leetcode.com/problems/smallest-range-covering-elements-from-k-lists/

**Explanation:** Given k sorted lists, find the smallest range [a, b] containing at least one element from every list.

**Intuition:** Hold a "window" of exactly one element per list — the range covering the window is [min of window, max of window]. To shrink a range you must raise its minimum, and the only legal move is advancing the pointer *in the list that owns the minimum* (advancing anyone else can't raise the min). So: min-heap over the window plus a running max; repeatedly pop the min, record the range, advance that list. When some list runs dry, no future window can cover all k — stop.

**Brute force:** For every element as range-start, binary search each list for its cheapest partner: O(N*k log N).

**Optimization:** The k-way sweep → O(N log k).

**How the heap helps:** The window's minimum — the only element whose advancement can help — is always at the root.

```java
public int[] smallestRange(List<List<Integer>> nums) {
    // Heap entry: {value, listIdx, elemIdx}. Window = one element per list.
    PriorityQueue<int[]> heap = new PriorityQueue<>((a, b) -> Integer.compare(a[0], b[0]));
    int curMax = Integer.MIN_VALUE;
    for (int i = 0; i < nums.size(); i++) {
        int v = nums.get(i).get(0);
        heap.offer(new int[]{v, i, 0});
        curMax = Math.max(curMax, v);
    }

    int bestLo = 0, bestHi = Integer.MAX_VALUE;
    while (true) {
        int[] min = heap.poll();                       // current range = [min, curMax]
        if (curMax - min[0] < bestHi - bestLo) { bestLo = min[0]; bestHi = curMax; }

        if (min[2] + 1 == nums.get(min[1]).size()) break;  // that list ran dry -> done
        int next = nums.get(min[1]).get(min[2] + 1);       // advance ONLY the min's owner
        heap.offer(new int[]{next, min[1], min[2] + 1});
        curMax = Math.max(curMax, next);
    }
    return new int[]{bestLo, bestHi};
}
```

**Time:** O(N log k), N = total elements
**Space:** O(k)

**Tips & tricks:**
- The max needs no heap: it only ever grows (new elements can only raise it), so a plain running variable suffices. Min shrinks and grows → heap. Asymmetry worth narrating.
- Strict `<` when updating best keeps the earliest (smallest-a) optimal range, matching the required tie-break.
- Same window idea powers "merge k sorted iterators" system-design follow-ups.

---

## Section 4 — Greedy Scheduling

### 8. Task Scheduler
**Link:** https://leetcode.com/problems/task-scheduler/

**Explanation:** CPU runs one task per tick; identical tasks must be >= n ticks apart. Minimize total ticks (idles allowed).

**Intuition:** Scarcity rules: the task with the most remaining copies is the bottleneck — schedule it the moment it's legal, or its copies pile up at the end forcing idles. So each tick, run the most-abundant *runnable* task. Tasks just run enter a cooldown; a plain FIFO queue holds them stamped with their release time (they re-enter in the order they left, since release times are monotone).

**Brute force:** Try all orderings — factorial, dead on arrival.

**Optimization:** Max-heap on remaining counts + cooldown queue, one pop per tick. Or pure math (below).

**How the heap helps:** "Most abundant among currently runnable" changes every tick; the heap keeps that query O(log 26).

```java
public int leastInterval(char[] tasks, int n) {
    int[] count = new int[26];
    for (char t : tasks) count[t - 'A']++;

    PriorityQueue<Integer> ready = new PriorityQueue<>(Comparator.reverseOrder());
    for (int c : count) if (c > 0) ready.offer(c);

    Deque<int[]> cooling = new ArrayDeque<>();   // {remaining count, tick it becomes runnable}
    int time = 0;
    while (!ready.isEmpty() || !cooling.isEmpty()) {
        time++;
        if (!cooling.isEmpty() && cooling.peek()[1] == time)
            ready.offer(cooling.poll()[0]);      // cooldown over -> back in the pool
        if (!ready.isEmpty()) {
            int c = ready.poll() - 1;            // run the most abundant task
            if (c > 0) cooling.offer(new int[]{c, time + n + 1});
        }                                        // else: forced idle tick
    }
    return time;
}
```

**Time:** O(T log 26) = O(T), T = total ticks
**Space:** O(26) = O(1)

**Tips & tricks:**
- Closed form to volunteer: `max(tasks.length, (maxCount - 1) * (n + 1) + numOfTasksWithMaxCount)` — picture maxCount rows of width n+1; the max() covers the "no idles needed" case.
- Only the *count* matters, not the identity — the heap holds bare ints. Noticing that halves the code.
- Cooldown release times are monotone → FIFO deque, not a second heap. Two structures, two jobs.

---

### 9. Reorganize String
**Link:** https://leetcode.com/problems/reorganize-string/

**Explanation:** Rearrange s so no two adjacent characters are equal; return "" if impossible.

**Intuition:** Task Scheduler with n = 1, output the actual string. Always place the most abundant *available* letter; the letter just placed sits out exactly one round (the `hold` variable is a cooldown queue of capacity 1). If the heap empties while something is still on hold, the leftovers are all one letter — impossible.

**Brute force:** Try permutations — no.

**Optimization:** Greedy max-heap placement, O(L log 26). (O(L) construction by filling even-then-odd indices with the sorted-by-count letters also exists — good to mention.)

**How the heap helps:** Re-answers "most abundant letter, excluding the one on cooldown" after every placement.

```java
public String reorganizeString(String s) {
    int[] count = new int[26];
    for (char c : s.toCharArray()) count[c - 'a']++;

    PriorityQueue<int[]> heap =                                   // {letter, remaining}
        new PriorityQueue<>((a, b) -> Integer.compare(b[1], a[1]));
    for (int i = 0; i < 26; i++) if (count[i] > 0) heap.offer(new int[]{i, count[i]});

    StringBuilder sb = new StringBuilder();
    int[] hold = null;                        // letter serving its 1-slot cooldown
    while (!heap.isEmpty()) {
        int[] cur = heap.poll();              // most abundant AVAILABLE letter
        sb.append((char) ('a' + cur[0]));
        cur[1]--;
        if (hold != null) heap.offer(hold);   // previous letter's cooldown ends
        hold = (cur[1] > 0) ? cur : null;     // this one sits out the next round
    }
    return (hold == null) ? sb.toString() : "";  // stuck holding leftovers -> impossible
}
```

**Time:** O(L log 26) = O(L)
**Space:** O(1) beyond the output

**Tips & tricks:**
- Early-exit check to volunteer: impossible iff maxCount > (L + 1) / 2 — the pigeonhole bound.
- Re-offer the held letter *after* popping the current one, or the same letter comes straight back.
- Generalization: Rearrange String k Distance Apart = same code with a k-1 capacity cooldown queue.

---

### 10. Single-Threaded CPU
**Link:** https://leetcode.com/problems/single-threaded-cpu/

**Explanation:** Tasks have {enqueueTime, processingTime}. An idle CPU takes the available task with the shortest processing time (tie: smallest index) and runs it to completion. Return execution order.

**Intuition:** The universal time-simulation template: sort by arrival to know *when tasks appear*; a heap of ready tasks knows *which runs next*; a clock advances by whole jobs. Only wrinkle: when the CPU is idle and nothing is ready, *jump* the clock to the next arrival instead of ticking.

**Brute force:** At each completion, linear-scan all tasks for the best available: O(n^2).

**Optimization:** Sort + ready-heap → O(n log n).

**How the heap helps:** The ready set changes as tasks arrive and finish; the heap keeps "shortest job, then smallest index" as its root through all of it.

```java
public int[] getOrder(int[][] tasks) {
    int n = tasks.length;
    int[][] jobs = new int[n][3];                       // {enqueue, processing, originalIdx}
    for (int i = 0; i < n; i++) jobs[i] = new int[]{tasks[i][0], tasks[i][1], i};
    Arrays.sort(jobs, (a, b) -> Integer.compare(a[0], b[0]));   // by arrival

    PriorityQueue<int[]> ready = new PriorityQueue<>((a, b) ->
        a[1] != b[1] ? Integer.compare(a[1], b[1])      // shortest processing time
                     : Integer.compare(a[2], b[2]));    // tie: smallest original index

    int[] order = new int[n];
    int idx = 0, done = 0;
    long clock = 0;                                     // sums of 1e9-scale times overflow int
    while (done < n) {
        while (idx < n && jobs[idx][0] <= clock)        // admit everything already arrived
            ready.offer(jobs[idx++]);
        if (ready.isEmpty()) {
            clock = jobs[idx][0];                       // idle -> JUMP to next arrival
            continue;
        }
        int[] job = ready.poll();
        clock += job[1];                                // run to completion
        order[done++] = job[2];
    }
    return order;
}
```

**Time:** O(n log n)
**Space:** O(n)

**Tips & tricks:**
- `long clock` — enqueue times up to 1e9 plus accumulated processing times overflow int. Classic silent-wrong-answer.
- Sorting destroys indices; carry the original index inside the tuple. (Index transposition territory — dry-run the tuple layout.)
- This skeleton (sort arrivals + ready-heap + clock-jump) solves Meeting Rooms III, Process Tasks Using Servers, and most "simulate a scheduler" interview originals.

---

## Section 5 — Heap-as-Regret

### 11. Furthest Building You Can Reach
**Link:** https://leetcode.com/problems/furthest-building-you-can-reach/

**Explanation:** Moving left to right over building heights with b bricks and l ladders; an upward climb of h costs h bricks or 1 ladder. How far can you go?

**Intuition:** You can't know upfront which climbs deserve ladders — the big climbs might be ahead. So don't decide: **optimistically spend a ladder on every climb**, and keep all laddered climbs in a min-heap. The moment you exceed l ladders, retroactively downgrade the *smallest* laddered climb to bricks (ladders must be saved for the biggest climbs seen so far). If bricks go negative, this building is unreachable — you stop just before it.

**Brute force:** Try every assignment of ladders to climbs: exponential.

**Optimization:** Greedy with retroactive undo → O(n log l).

**How the heap helps:** It stores past commitments so the *cheapest one to regret* is always at the root. The heap is a time machine, not a frontier.

```java
public int furthestBuilding(int[] heights, int bricks, int ladders) {
    PriorityQueue<Integer> laddered = new PriorityQueue<>();   // climbs currently on ladders
    for (int i = 0; i < heights.length - 1; i++) {
        int climb = heights[i + 1] - heights[i];
        if (climb <= 0) continue;                  // downhill/flat is free
        laddered.offer(climb);                     // optimistic: use a ladder
        if (laddered.size() > ladders) {           // over budget ->
            bricks -= laddered.poll();             //   smallest laddered climb pays bricks instead
            if (bricks < 0) return i;              // can't afford the downgrade -> stuck at i
        }
    }
    return heights.length - 1;                     // survived every climb
}
```

**Time:** O(n log l)
**Space:** O(l)

**Tips & tricks:**
- Heap size never exceeds ladders + 1 — the eviction fires immediately.
- The exchange argument, one line: if any ladder covers a smaller climb than any bricked climb, swapping them frees bricks. Have it ready.
- Return value on failure is i (last reachable), not i + 1 — off-by-one bait.

---

### 12. IPO
**Link:** https://leetcode.com/problems/ipo/

**Explanation:** Start with capital w. Each project needs capital[i] to unlock and pays profits[i] (pure profit, capital not spent). Pick at most k projects to maximize final capital.

**Intuition:** Capital only ever grows, so the set of affordable projects only ever grows — a one-way conveyor belt. Sort projects by capital requirement; keep a pointer that migrates newly-affordable projects into a max-heap by profit; k times, greedily take the max profit. Greedy is safe because taking the biggest profit now can never lock you out of anything (unlocks are monotone in w).

**Brute force:** Each round, scan all projects for the best affordable: O(k*n).

**Optimization:** Sort + conveyor + heap → O(n log n + k log n).

**How the heap helps:** Two orderings compete — capital (for unlocking) and profit (for choosing). The sort handles one, the heap the other; the pointer is the hand-off between them.

```java
public int findMaximizedCapital(int k, int w, int[] profits, int[] capital) {
    int n = profits.length;
    int[][] projects = new int[n][2];
    for (int i = 0; i < n; i++) projects[i] = new int[]{capital[i], profits[i]};
    Arrays.sort(projects, (a, b) -> Integer.compare(a[0], b[0]));   // by capital needed

    PriorityQueue<Integer> affordable = new PriorityQueue<>(Comparator.reverseOrder());
    int i = 0;
    while (k-- > 0) {
        while (i < n && projects[i][0] <= w)       // conveyor: unlock everything now affordable
            affordable.offer(projects[i++][1]);
        if (affordable.isEmpty()) break;           // nothing affordable; w only grows via picks -> stuck
        w += affordable.poll();                    // greedy: max profit among unlocked
    }
    return w;
}
```

**Time:** O(n log n + k log n)
**Space:** O(n)

**Tips & tricks:**
- The unlock pointer i never resets — each project enters the heap exactly once across all k rounds. That's why it's not O(k*n log n).
- Early break when the heap is empty is required, not cosmetic: w can't grow without a pick.
- Final w brushes Integer.MAX at extreme constraints (1e9 + 1e5*1e4 = 2e9 < 2.147e9) — worth saying "fits, barely; I'd use long defensively."

---

### 13. Minimum Number of Refueling Stops
**Link:** https://leetcode.com/problems/minimum-number-of-refueling-stops/

**Explanation:** Car starts with startFuel, targets position `target`; stations[i] = {position, fuel}. Minimize refueling stops, or -1.

**Intuition:** Heap-as-regret in its purest form. Drive past every station *without stopping*, but bank each station's fuel in a max-heap. Only when the tank would run dry do you retroactively declare "actually, I stopped at the best station I already passed" — pop the max, add its fuel, count a stop. The decision about station i is made miles after passing it. Each forced pop is the largest available, so stop count is minimal.

**Brute force:** DP over subsets of stations, or dp[i][j] = farthest reachable using j stops among first i stations — O(n^2), the standard non-heap answer.

**Optimization:** Greedy + max-heap → O(n log n).

**How the heap helps:** Keeps the not-yet-claimed past options ranked, so every forced retro-claim is optimal.

```java
public int minRefuelStops(int target, int startFuel, int[][] stations) {
    PriorityQueue<Integer> passed = new PriorityQueue<>(Comparator.reverseOrder());
    long reach = startFuel;                        // farthest position with current commitments
    int stops = 0, i = 0;
    while (reach < target) {
        // Bank every station we can already reach — WITHOUT stopping.
        while (i < stations.length && stations[i][0] <= reach)
            passed.offer(stations[i++][1]);
        if (passed.isEmpty()) return -1;           // stranded: nothing left to retro-claim
        reach += passed.poll();                    // retroactively stop at the best passed station
        stops++;
    }
    return stops;
}
```

**Time:** O(n log n)
**Space:** O(n)

**Tips & tricks:**
- `reach` as running total means you never track "current fuel" separately — position math collapses into one variable.
- Note the loop shape mirrors IPO exactly: unlock-pointer + heap + greedy pop. Same machine, regret flavor.
- Volunteer the O(n^2) DP as the alternative; the heap greedy is strictly better here and saying why (exchange argument on fuel amounts) scores.

---

## Section 6 — Sort One Dimension, Heap the Other

### 14. Minimum Cost to Hire K Workers
**Link:** https://leetcode.com/problems/minimum-cost-to-hire-k-workers/

**Explanation:** Worker i has quality[i] and minimum wage[i]. Hire exactly k, paying proportionally to quality, everyone >= their minimum. Minimize total pay.

**Intuition:** Proportional pay means the whole group shares one pay *rate* (dollars per quality unit), and that rate must satisfy everyone → rate = max(wage/quality) over the group. So total cost = (group quality sum) x (worst ratio in group). Two competing quantities. Fix one: sort by ratio ascending and sweep — when worker i is the *rate-setter*, everyone cheaper-ratio is eligible, and you want the k-1 smallest qualities among them → max-heap of qualities evicting the largest. Check cost at every i where the group has k members.

**Brute force:** For each candidate rate-setter, sort eligible workers by quality: O(n^2 log n).

**Optimization:** Sort by ratio + size-k quality heap → O(n log n).

**How the heap helps:** Maintains "k smallest qualities among all ratio-eligible workers" incrementally as the eligibility set grows — the size-k eviction pattern from Sec 1, embedded in a greedy sweep.

```java
public double mincostToHireWorkers(int[] quality, int[] wage, int k) {
    int n = quality.length;
    Integer[] idx = new Integer[n];
    for (int i = 0; i < n; i++) idx[i] = i;
    // Sort by ratio: the group's pay rate is its LARGEST ratio, so sweep
    // ascending — the current worker is always the rate-setter.
    Arrays.sort(idx, (a, b) -> Double.compare((double) wage[a] / quality[a],
                                              (double) wage[b] / quality[b]));

    PriorityQueue<Integer> qHeap = new PriorityQueue<>(Comparator.reverseOrder());
    long qSum = 0;
    double best = Double.MAX_VALUE;
    for (int j : idx) {
        qHeap.offer(quality[j]);
        qSum += quality[j];
        if (qHeap.size() > k) qSum -= qHeap.poll();   // evict largest quality = biggest cost
        if (qHeap.size() == k)
            best = Math.min(best, qSum * ((double) wage[j] / quality[j]));
    }
    return best;
}
```

**Time:** O(n log n)
**Space:** O(n)

**Tips & tricks:**
- The heap holds QUALITIES while the sort is on RATIOS — heaping the wrong quantity is the canonical mistake. Ratio and quality are two concerns handled by two mechanisms; don't fuse.
- The rate-setter i might itself be evicted from the quality heap — that's fine: the cost formula still uses ratio[i] as an upper bound, and the true optimum is checked when the actual max-ratio member is the sweep position.
- Keep qSum alongside the heap; recomputing the sum per step silently turns O(n log n) into O(nk).

---

## Section 7 — Design + Lazy Deletion

### 15. Stock Price Fluctuation
**Link:** https://leetcode.com/problems/stock-price-fluctuation/

**Explanation:** Stream of (timestamp, price) updates where a timestamp may be *corrected* later. Support current(), maximum(), minimum().

**Intuition:** Corrections mean heap entries go stale — but heaps can't update. So don't: keep a hashmap as the single source of truth (timestamp → latest price), push every update into both heaps as a new fact, and on maximum()/minimum() **validate the root against the truth map**, discarding entries whose price no longer matches their timestamp. Same lazy-deletion move as Sliding Window Median, with the truth map replacing the ghost count.

**Brute force:** Scan the truth map on every max/min query: O(n) per query.

**Optimization:** Two heaps + lazy validation → amortized O(log n) per operation.

**How the heap helps:** Max/min over a corrected stream, without ever deleting — each entry is popped at most once, so garbage costs amortized O(log n).

```java
class StockPrice {
    private final Map<Integer, Integer> truth = new HashMap<>();  // timestamp -> latest price
    private int latestTime = 0;
    // Heap entries {price, timestamp} — possibly stale; validate on peek.
    private final PriorityQueue<int[]> maxH = new PriorityQueue<>((a, b) -> Integer.compare(b[0], a[0]));
    private final PriorityQueue<int[]> minH = new PriorityQueue<>((a, b) -> Integer.compare(a[0], b[0]));

    public void update(int timestamp, int price) {
        truth.put(timestamp, price);
        latestTime = Math.max(latestTime, timestamp);
        maxH.offer(new int[]{price, timestamp});     // never delete — just add the new fact
        minH.offer(new int[]{price, timestamp});
    }

    public int current() { return truth.get(latestTime); }

    public int maximum() {
        while (truth.get(maxH.peek()[1]) != maxH.peek()[0]) maxH.poll();  // skip corrected entries
        return maxH.peek()[0];
    }

    public int minimum() {
        while (truth.get(minH.peek()[1]) != minH.peek()[0]) minH.poll();
        return minH.peek()[0];
    }
}
```

**Time:** O(log n) update; amortized O(log n) maximum/minimum; O(1) current
**Space:** O(n) including stale entries

**Tips & tricks:**
- Validation predicate is "does the truth map still say this (timestamp, price) pair?" — price alone isn't enough, two timestamps can share a price.
- Alternative to volunteer: TreeMap<price, count> with decrement-on-correction — O(log n) worst-case (not amortized), one structure instead of two. Great "heap vs TreeMap" discussion piece.
- peek-validate loops must run before EVERY read of the root, including inside the same method twice.

---

### 16. Exam Room
**Link:** https://leetcode.com/problems/exam-room/

**Explanation:** n seats in a row. seat(): sit maximizing distance to the closest person (tie → lowest index). leave(p): person at p leaves. Many mixed calls.

**Intuition:** Think in *gaps*, not seats. Represent each empty stretch by its bounding persons {l, r} (exclusive; virtual walls at -1 and n). The best seat is determined entirely by the best gap: interior gap → midpoint, distance (r-l)/2; wall gap → sit at the wall, distance r (left) or n-1-l (right). seat() = pop best gap, split into two. leave(p) = merge p's two neighboring gaps. The heap can't delete merged-away gaps → lazy deletion again: two maps (byLeft, byRight) hold the *live* gaps; a popped gap is valid only if the maps still point to that exact object.

**Brute force:** TreeSet of occupied seats, linear scan per seat(): O(P) per call — acceptable for small constraints, and worth naming.

**Optimization:** Gap heap + neighbor maps → O(log C) per call amortized.

**How the heap helps:** "Best gap right now" changes with every seat/leave; the heap keeps it at the root while the maps carry the truth.

```java
class ExamRoom {
    private final int n;
    // A gap is {l, r}: occupied (or virtual -1 / n) bounds; empty seats strictly between.
    private final PriorityQueue<int[]> gaps;
    private final Map<Integer, int[]> byLeft  = new HashMap<>();  // l -> live gap starting at l
    private final Map<Integer, int[]> byRight = new HashMap<>();  // r -> live gap ending at r

    public ExamRoom(int n) {
        this.n = n;
        gaps = new PriorityQueue<>((a, b) -> {
            int da = score(a), db = score(b);
            return da != db ? Integer.compare(db, da)             // larger distance first
                            : Integer.compare(seatOf(a), seatOf(b)); // tie: lowest seat
        });
        addGap(new int[]{-1, n});                                 // whole room
    }

    private int score(int[] g) {                 // distance to closest person if seated here
        if (g[0] == -1)   return g[1];           // left wall: sit at 0
        if (g[1] == n)    return n - 1 - g[0];   // right wall: sit at n-1
        return (g[1] - g[0]) / 2;                // interior: midpoint
    }
    private int seatOf(int[] g) {
        if (g[0] == -1) return 0;
        if (g[1] == n)  return n - 1;
        return (g[0] + g[1]) / 2;
    }

    private void addGap(int[] g) {
        byLeft.put(g[0], g);                     // maps always hold the gap (even width 0),
        byRight.put(g[1], g);                    //   so leave() can find neighbors to merge
        if (g[1] - g[0] >= 2) gaps.offer(g);     // heap only holds seatable gaps
    }
    private boolean live(int[] g) {              // stale check: reference equality vs the maps
        return byLeft.get(g[0]) == g && byRight.get(g[1]) == g;
    }

    public int seat() {
        int[] g;
        do { g = gaps.poll(); } while (!live(g));   // lazy deletion
        int s = seatOf(g);
        addGap(new int[]{g[0], s});                 // split; old gap left stale in heap
        addGap(new int[]{s, g[1]});
        return s;
    }

    public void leave(int p) {
        int[] left = byRight.get(p), right = byLeft.get(p);   // p's two flanking gaps
        byRight.remove(p); byLeft.remove(p);
        addGap(new int[]{left[0], right[1]});                 // merged gap replaces both
    }
}
```

**Time:** amortized O(log C) per operation, C = number of calls
**Space:** O(C)

**Tips & tricks:**
- Exclusive {personLeft, personRight} bounds make all three distance formulas clean; storing empty ranges inclusive breeds +/-1 bugs in every branch.
- Zero-width gaps live in the maps (leave() needs them for merging) but never in the heap (nothing to seat). Two homes, two admission rules.
- live() uses reference equality (==) deliberately — two distinct gap objects can have equal bounds across history.

---

## Section 8 — Delayed Decisions

### 17. Avoid Flood in the City
**Link:** https://leetcode.com/problems/avoid-flood-in-the-city/

**Explanation:** rains[i] > 0 fills that lake (flood if already full); rains[i] == 0 lets you dry ONE lake of your choice. Return a valid plan or [].

**Intuition:** On a dry day you don't know which lake to dry — the dangerous lake reveals itself later. So **don't decide**: bank the dry day in a sorted set. When rain hits an already-full lake, you're forced: you must have dried it on some dry day *after it last filled* — and greedily the **earliest** such day (later dry days are more flexible, keep them). That's a ceiling query on the banked days: `dryDays.ceiling(lastFillDay)`. Note this kills the heap: the day you need is not the min or max of the banked days, it's the smallest one after a *query-dependent* threshold. Level-0 fork again: arbitrary-threshold lookup → TreeSet, not heap.

**Brute force:** Try all assignments of dry days to lakes: exponential. Or per flood, linear-scan dry days: O(n^2).

**Optimization:** TreeSet + greedy earliest-after → O(n log n).

**How the heap helps:** It doesn't — the required query is ceiling(), which heaps can't answer. Paired with MK Average as the two "know when the heap loses" problems.

```java
public int[] avoidFlood(int[] rains) {
    int n = rains.length;
    int[] ans = new int[n];
    Map<Integer, Integer> fullSince = new HashMap<>();  // lake -> day it last filled
    TreeSet<Integer> dryDays = new TreeSet<>();         // banked, unspent dry days

    for (int day = 0; day < n; day++) {
        int lake = rains[day];
        if (lake == 0) {
            dryDays.add(day);                 // bank it; decide later
            ans[day] = 1;                     // placeholder if never needed (any lake is valid)
        } else {
            ans[day] = -1;
            Integer lastFill = fullSince.get(lake);
            if (lastFill != null) {
                // Forced: must have dried this lake between its last fill and today.
                Integer spend = dryDays.ceiling(lastFill);  // earliest dry day after the fill
                if (spend == null) return new int[0];       // flood unavoidable
                ans[spend] = lake;
                dryDays.remove(spend);
            }
            fullSince.put(lake, day);
        }
    }
    return ans;
}
```

**Time:** O(n log n)
**Space:** O(n)

**Tips & tricks:**
- Unused dry days must output a valid lake (1 is safe), never 0 — a spec detail graders check.
- Greedy proof in one line: spending the earliest eligible day can only leave a superset of options for future floods (exchange argument).
- If asked "why not a heap of dry days?": popping the global min might spend a day that is *before* the lake filled — useless — and you can't put it back selectively.

---

### 18. Zero Array Transformation III
**Link:** https://leetcode.com/problems/zero-array-transformation-iii/

**Explanation:** queries[j] = [l, r] lets you decrement (by <= 1) every index in [l, r]. Choose a subset of queries so nums can be reduced to all zeros; maximize the number of REMOVED queries (return removals, or -1).

**Intuition:** Maximize removals = minimize kept queries. Sweep indices left to right; at index i, you need nums[i] active decrements covering i. Two pools: **available** (queries with l <= i, not yet chosen — max-heap on r) and **active** (chosen queries still covering i — min-heap on r, popped as they expire). When short, commit available queries greedily by **furthest right endpoint** — among queries all covering i, the one reaching furthest right can only help more future indices (exchange argument). If the best available doesn't even reach i → -1.

**Brute force:** Try all subsets: 2^q.

**Optimization:** Sort by l + two heaps → O((n + q) log q).

**How the heap helps:** Two heaps, two questions: "best option to commit next" (max by r) and "which commitments have expired" (min by r). Delayed decisions with an ordered option pool — Avoid Flood's cousin, but here the extreme IS the right query, so heaps fit.

```java
public int maxRemoval(int[] nums, int[][] queries) {
    Arrays.sort(queries, (a, b) -> Integer.compare(a[0], b[0]));   // by left endpoint

    PriorityQueue<Integer> available = new PriorityQueue<>(Comparator.reverseOrder()); // r of usable queries
    PriorityQueue<Integer> active    = new PriorityQueue<>();      // r of CHOSEN queries, min to expire
    int qi = 0, used = 0;

    for (int i = 0; i < nums.length; i++) {
        while (qi < queries.length && queries[qi][0] <= i)   // unlock queries starting by i
            available.offer(queries[qi++][1]);
        while (!active.isEmpty() && active.peek() < i)       // expire commitments ending before i
            active.poll();

        while (active.size() < nums[i]) {                    // need more coverage at i
            if (available.isEmpty() || available.peek() < i)
                return -1;                                   // best option can't even reach i
            active.offer(available.poll());                  // commit the furthest-reaching query
            used++;
        }
    }
    return queries.length - used;
}
```

**Time:** O((n + q) log q)
**Space:** O(q)

**Tips & tricks:**
- available.peek() < i means the query starts <= i but ends before i — committing it is pure waste; that's the -1 detector, not just an emptiness check.
- Expired active entries never "come back" — expiry is monotone in i, so plain pops suffice, no lazy deletion needed.
- Same unlock-pointer + commit-heap skeleton as IPO/Refueling — regret machinery, interval flavor.

---

## Section 9 — Boss

### 19. Car Fleet II
**Link:** https://leetcode.com/problems/car-fleet-ii/

**Explanation:** Cars on one lane, sorted by position; cars[i] = {position, speed}. When a car catches the one ahead, it merges and adopts its speed. Return each car's collision time (-1 if never).

**Intuition:** Car i can only ever hit something *ahead*, so process right to left with a stack of candidate targets. Looking at the car j on top of the stack, discard it if (a) j is at least as fast — never catchable — or (b) j collides and vanishes at ans[j] *before* i would reach it (i's real target is whatever j merged into, deeper in the stack). What survives is i's true first victim; simple relative-speed division gives the time. Each car pushes and pops at most once → O(n).

**Brute force:** Simulate collisions event by event, or for each car scan ahead: O(n^2).

**Optimization:** Right-to-left **monotonic stack** → O(n). This is the optimal solution — and the honest note is that the heap is NOT it.

**How the heap helps (and why it loses):** The heap version simulates the *earliest pending collision* globally: min-heap of tentative collision times between adjacent pairs; pop the earliest, record it, delete the crashed car, re-link its neighbors, push their new tentative time — with lazy deletion for invalidated pairs. It works in O(n log n) and is a legitimate event-simulation pattern (same family as Meeting Rooms III), but the stack exploits the "only ahead matters" structure for O(n). Know both; code the stack.

```java
public double[] getCollisionTimes(int[][] cars) {
    int n = cars.length;
    double[] ans = new double[n];
    Deque<Integer> stack = new ArrayDeque<>();   // indices of surviving candidate targets ahead

    for (int i = n - 1; i >= 0; i--) {
        while (!stack.isEmpty()) {
            int j = stack.peek();
            // Discard j if uncatchable, or if j vanishes before i reaches it.
            if (cars[j][1] >= cars[i][1] ||
                (ans[j] > 0 && catchTime(cars, i, j) >= ans[j]))
                stack.pop();
            else break;
        }
        ans[i] = stack.isEmpty() ? -1 : catchTime(cars, i, stack.peek());
        stack.push(i);
    }
    return ans;
}

// Time for i to catch j, assuming j never changes: gap / speed difference.
private double catchTime(int[][] cars, int i, int j) {
    return (double) (cars[j][0] - cars[i][0]) / (cars[i][1] - cars[j][1]);
}
```

**Time:** O(n)
**Space:** O(n)

**Tips & tricks:**
- The discard test uses ans[j] > 0 to mean "j actually collides" (-1 = never, so j is a permanent target if slower).
- >= vs > in the vanish test: if i arrives exactly as j merges, i effectively hits the fleet j joined — popping is correct; the deeper target yields the same time.
- Cast before dividing; integer division here fails silently. And say the heap-event-simulation alternative out loud — it's the expected "how else?" answer.

---

## Cross-problem cheat sheet

| Trigger phrase in a problem | Reach for |
|---|---|
| "k largest / most frequent / closest" | size-k heap, inverted polarity (Sec 1) |
| "median / percentile of a stream" | two heaps at a fence (Sec 2) |
| "...and elements also leave" | + lazy deletion, self-tracked live counts (Sec 2, Sec 7) |
| "k sorted sources" | frontier heap, one entry per source (Sec 3) |
| "cooldown / can't repeat within n" | max-heap on counts + FIFO cooldown (Sec 4) |
| "simulate a scheduler / CPU / rooms" | sort arrivals + ready-heap + clock jump (Sec 4) |
| "spend limited resources on unknown future" | optimistic commit + regret heap (Sec 5) |
| "cost = f(one dim) x g(another dim)" | sort one, heap the other (Sec 6) |
| "values get corrected / updated" | truth map + validate-on-pop (Sec 7) |
| "decide later / earliest option after X" | TreeSet ceiling — heap loses (Sec 8) |
| needs arbitrary deletion AND both ends | TreeMap multiset — heap loses (Sec 2, MK Avg) |
