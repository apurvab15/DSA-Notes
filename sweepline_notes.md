# Sweep Line — Study Notes

## Intuition

Sweep line solves problems about **intervals or objects on a line** (a time axis, an
x-axis) where the question is really about *what is true at any given moment* — how many
things overlap, where the coverage changes, where a gap opens up.

The reframe that unlocks everything:

> The "state" (how many intervals are active, what the max height is, what's covered)
> **only changes at endpoints.** Between two consecutive endpoints nothing happens — the
> value is flat.

So you never inspect continuous space. You only look at the discrete *moments where the
state changes* — the **events**. Each interval contributes exactly two events: a start
(`+1`) and an end (`-1`). Sort all events by position and walk them left→right — that walk
*is* the sweep — maintaining an **active state**. Whatever you're asked for is read off that
state as you go.

### The reusable skeleton

1. **Turn each object into events** — usually its two endpoints, tagged with what they do.
2. **Sort events by position**, with a deliberate tie-breaking rule.
3. **Sweep in order**, maintaining an *active state*.
4. **At each event, update the state and record the answer.**

The only thing that changes between problems is what the active state *is*. That's the knob:

| Active state | Query each step | Example |
|---|---|---|
| integer counter | current value / running max | Meeting Rooms II, Car Pooling |
| `TreeMap<key,count>` (ordered multiset) | running **max** with deletes | Skyline |
| coverage over compressed strips | union **length** / area | Rectangle Area II |
| running merged `end` | where count hits **zero** | Employee Free Time |
| two heaps (`free` + `busy`) | lowest free / earliest-freeing | Meeting Rooms III, Process Tasks (2410) |
| min-heap on the answer, lazy-deleted | min over live items | Minimum Interval per Query (1851) |

### The recurring reflex — tie-breaking

Every sweep-line problem forces one decision: **does *touching* count as overlap?**
When a start and an end share a coordinate:

- **Half-open intervals** (`[a, b)`, end exclusive — a meeting ends *as* the next begins):
  process the **end before the start**. In an event list, sort key `(-1) < (+1)` does this.
  In a two-pointer form, strict `<` does this.
- **Closed intervals** (touching *does* count as overlap): process the **start first**,
  i.e. use `<=`.

Get this backwards and you are off by one *exactly at the boundaries* — a silent wrong
answer that is painful to spot in test output. Pin this down before writing any code.

### When to reach for it (recognition cues)

- Intervals with overlaps: "maximum concurrent X", "minimum rooms/resources".
- Merging or finding gaps in a set of intervals.
- Coverage / union length / union area on a line or in 2D.
- 2D geometry via a vertical line moving across the plane (skyline, rectangle union,
  segment intersection) — the active state represents the current cross-section.
- The tell: **the answer is fully determined by transitions, and transitions only happen
  at endpoints.**

Related muscle you already have: *merge intervals* is a degenerate sweep, and the
"sort to expose structure, then one disciplined pass" shape is the same as the two-pointer
merge in the greedy array-partitioning family.

---

## Problems it applies to

Marked ★ are worked in detail below.

**Interval overlap / resource counting**
- Meeting Rooms (LC 252) — do any two overlap?
- Meeting Rooms II (LC 253) — min rooms = max concurrent ★ (foundational entry)
- Car Pooling (LC 1094) — difference-array sweep over a capacity
- Corporate Flight Bookings (LC 1109) — difference array
- My Calendar I / II (LC 729 / 731) — booking with overlap limits
- My Calendar III (LC 732) — running max k-booking ★
- Number of Flowers in Full Bloom (LC 2251) — binary search on sorted starts & ends ★
- Minimum Interval to Include Each Query (LC 1851) — offline queries + lazy-deleted heap ★
- Maximum Population Year (LC 1854) — tiny classic sweep
- Check if All the Integers in a Range Are Covered (LC 1893)

**Merging / gaps**
- Merge Intervals (LC 56)
- Insert Interval (LC 57)
- Interval List Intersections (LC 986)
- Employee Free Time (LC 759) ★

**Coverage / painting / covering a range**
- Amount of New Area Painted Each Day (LC 2158) — union of painted intervals, pointer-jump ★
- Minimum Number of Taps to Open to Water a Garden (LC 1326) — min intervals to cover [0,n], greedy ★
- Range Module (LC 715) — add / remove / query interval coverage

**2D geometry**
- The Skyline Problem (LC 218) ★
- Rectangle Area II (LC 850) ★
- Falling Squares (LC 699)
- Line-segment intersection (Bentley–Ottmann)

**Rectangle from corners — geometric hashing** *(not a sweep; the diagonal-invariant family)*
- Minimum Area Rectangle (LC 939) — axis-aligned, diagonal → other two corners ★
- Minimum Area Rectangle II (LC 963) — any orientation, group by (center, diagonal) ★
- Number of Boomerangs / detect square — same "hash a geometric invariant" idea

**Simulation / assignment (event-driven, not pure sweep)**
- Meeting Rooms III (LC 2402) ★
- Process Tasks Using Servers (LC 2410) — two heaps, arrival time = index ★

---

## Meeting Rooms II — maximum concurrent intervals ★

**LC 253.** Given meeting intervals `[start, end]`, return the minimum number of rooms
required = the maximum number of meetings live at the same time.

**Example:** `[[1,4],[2,5],[3,6],[7,9]]` → `3` (meetings `[1,4] [2,5] [3,6]` all overlap
around time 3–4).

**Intuition.** The count of live meetings only changes at endpoints. Emit `+1` at each
start and `-1` at each end, sort, sweep, and track the peak of the running counter.
Enumerate the sorted events for the example:

```
pos:   1    2    3    4    5    6    7    9
event: +1   +1   +1   -1   -1   -1   +1   -1
count: 1    2    3    2    1    0    1    0
                 ^peak = 3
```

Tie-break: intervals are half-open here (a room frees *at* its end time), so an end at the
same coordinate as a start is processed first — `-1` sorts before `+1`.

**Type/knobs:** interval overlap · active state = integer counter · query = running max.

**Time:** `O(n log n)` — dominated by the sort.
**Space:** `O(n)` for the event list (or `O(1)` extra beyond the two sorted arrays form).

```java
// Event-list form — generalizes to every problem below.
public int minMeetingRooms(int[][] intervals) {
    List<int[]> events = new ArrayList<>();
    for (int[] iv : intervals) {
        events.add(new int[]{iv[0], +1});   // start
        events.add(new int[]{iv[1], -1});   // end
    }
    // sort by position; at a tie, end (-1) before start (+1)
    events.sort((a, b) -> a[0] != b[0] ? a[0] - b[0] : a[1] - b[1]);

    int cur = 0, max = 0;
    for (int[] e : events) { cur += e[1]; max = Math.max(max, cur); }
    return max;
}
```

**Tips.**
- The two-sorted-arrays form avoids building event objects: sort `starts` and `ends`
  separately, walk them like a merge, `if (starts[s] < ends[e]) rooms++ else e++`. The
  strict `<` is the half-open tie-break.
- If you need *which* meetings clash, keep the event form and record the active set.

**Trick / variant.** Car Pooling (LC 1094) is the same sweep with weighted events
(`+passengers` / `-passengers`) and a capacity check — if the running sum ever exceeds
capacity, return false.

---

## The Skyline Problem — active state is an ordered multiset ★

**LC 218.** Buildings as `[left, right, height]`. Return the skyline outline as key points
`[x, height]` where the visible height changes.

**Example:** `[[2,9,10],[3,7,15],[5,12,12]]` → `[[2,10],[3,15],[7,12],[12,0]]`.

**Intuition.** A plain counter fails: when the tallest building ends you need the *next*
tallest instantly. So the active state graduates to a `TreeMap<height, count>` — a multiset
that supports insert, delete, and `lastKey()` (current max) in `O(log n)`.

Clever encoding: emit start as `(x, -h)` and end as `(x, +h)`, sort by `(x, value)`. That
one key handles every tie:
- start vs end at same x → negative sorts first → a building starting where another ends
  keeps the line continuous (no false gap).
- two starts at same x → more-negative (taller) first → max jumps once, emit one point.
- two ends at same x → shorter (+h smaller) first → max only drops after the last leaves,
  emit one point.

Enumerate the example. Sorted events:
`(2,-10) (3,-15) (5,-12) (7,15) (9,10) (12,12)`.

```
event      heights (multiset)     lastKey   emit?
(2,-10)    {0,10}                 10        [2,10]
(3,-15)    {0,10,15}              15        [3,15]
(5,-12)    {0,10,12,15}           15        —
(7, 15)    {0,10,12}              12        [7,12]
(9, 10)    {0,12}                 12        —
(12,12)    {0}                    0         [12,0]
```

**Type/knobs:** 2D sweep · active state = `TreeMap<height,count>` · query = running max.

**Time:** `O(n log n)` — sort plus `O(log n)` per TreeMap op.
**Space:** `O(n)` for events and the multiset.

```java
public List<List<Integer>> getSkyline(int[][] buildings) {
    List<int[]> events = new ArrayList<>();
    for (int[] b : buildings) {
        events.add(new int[]{b[0], -b[2]});  // start: negative height
        events.add(new int[]{b[1],  b[2]});  // end:   positive height
    }
    events.sort((a, b) -> a[0] != b[0] ? a[0] - b[0] : a[1] - b[1]);

    List<List<Integer>> res = new ArrayList<>();
    TreeMap<Integer, Integer> heights = new TreeMap<>();
    heights.put(0, 1);                                  // ground, never removed
    int prevMax = 0;
    for (int[] e : events) {
        int h = e[1];
        if (h < 0) heights.merge(-h, 1, Integer::sum);  // start → add
        else       heights.merge(h, -1, Integer::sum);  // end   → remove
        heights.values().removeIf(c -> c == 0);         // drop empty heights
        int curMax = heights.lastKey();
        if (curMax != prevMax) { res.add(List.of(e[0], curMax)); prevMax = curMax; }
    }
    return res;
}
```

**Tips.**
- Seed the multiset with `0` (ground) so the outline can drop back to zero and `lastKey()`
  is always defined.
- Emit only when `lastKey()` *changes* — this is what collapses duplicate points.

**Trick / variant.** Falling Squares (LC 699) is a cousin: instead of "max over active
heights" you need "max height over an x-range as squares stack" — coordinate-compress x and
use a segment tree with range-max + range-assign.

---

## Rectangle Area II — active state is a set of intervals ★

**LC 850.** Given axis-aligned rectangles `[x1,y1,x2,y2]`, return the total area of their
union, modulo `1e9+7`.

**Example:** `[[0,0,2,2],[1,0,2,3],[1,0,3,1]]` → `6`.

**Intuition.** Sweep a vertical line left→right over the rectangles' x-edges. Between two
consecutive x-events the covered y-length is constant, so each vertical slab contributes
`covered_y_length × Δx`. Sum the slabs.

The active state is "which y-intervals are currently covered", and the query is the
*union length* of those intervals. Coordinate-compress the y-axis into strips, keep a
`cover[]` count per strip, and each slab's covered length is the sum of strip heights where
`cover[i] > 0`. The `> 0` test (not the raw count) is what dedupes overlap: a strip covered
by two rectangles still contributes its length exactly once.

Enumerate two overlapping rectangles as the line moves right: rect-1-only band (length =
height of rect 1) → overlap band (length = *union* height, counted once) → rect-2-only band.
No double counting.

**Type/knobs:** 2D sweep · active state = coverage over compressed y-strips · query = union
length.

**Time:** `O(n^2)` with the simple per-slab recount (`n` events × `n` strips); `O(n log n)`
with a segment tree replacing the inner recount.
**Space:** `O(n)` for events, strips, and the coverage array.

```java
public int rectangleArea(int[][] rects) {
    long MOD = 1_000_000_007L;
    TreeSet<Integer> ySet = new TreeSet<>();
    for (int[] r : rects) { ySet.add(r[1]); ySet.add(r[3]); }
    Integer[] ys = ySet.toArray(new Integer[0]);          // sorted y boundaries

    List<int[]> events = new ArrayList<>();               // (x, y1, y2, +1 open / -1 close)
    for (int[] r : rects) {
        events.add(new int[]{r[0], r[1], r[3],  1});      // left edge opens
        events.add(new int[]{r[2], r[1], r[3], -1});      // right edge closes
    }
    events.sort((a, b) -> Integer.compare(a[0], b[0]));

    int[] cover = new int[ys.length];
    long area = 0, prevX = events.get(0)[0];
    for (int[] e : events) {
        long width = e[0] - prevX;
        if (width > 0) {                                  // add the finished slab
            long covered = 0;
            for (int i = 0; i + 1 < ys.length; i++)
                if (cover[i] > 0) covered += ys[i + 1] - ys[i];
            area = (area + covered % MOD * (width % MOD)) % MOD;
        }
        for (int i = 0; i + 1 < ys.length; i++)           // apply event to spanned strips
            if (e[1] <= ys[i] && ys[i + 1] <= e[2]) cover[i] += e[3];
        prevX = e[0];
    }
    return (int) area;
}
```

**Tips.**
- Compress only the y-coordinates that actually appear; strip `i` spans `[ys[i], ys[i+1])`.
- Do the area accumulation for the *previous* slab *before* applying the current event, so
  the coverage reflects the slab you're closing.

**Trick / variant.** Same skeleton computes union *perimeter* (track where coverage toggles
0↔positive) and "amount of new area painted" style problems.

---

## Employee Free Time — sweep looking for count == 0 ★

**LC 759.** Each employee has a list of non-overlapping busy intervals. Return the finite
intervals of positive length where *every* employee is free.

**Example:** `[[[1,3],[6,7]],[[2,4]],[[2,5],[9,12]]]` → `[[5,6],[7,9]]`.

**Intuition.** Same overlap sweep as Meeting Rooms II, but instead of the max you watch for
the count dropping to **zero with a gap before the next event**. Equivalently: flatten all
intervals, sort by start, merge overlaps; the holes between merged busy blocks are the free
times. Track `end` = the far edge of the current merged busy block; when the next interval
starts strictly after `end`, everyone was free in between.

Enumerate: sorted intervals `[1,3] [2,4] [2,5] [6,7] [9,12]`. `end` walks `3 → 4 → 5`; then
`[6,7]` starts at `6 > 5` → free `[5,6]`, reset `end = 7`; then `[9,12]` starts at `9 > 7` →
free `[7,9]`. Result `[5,6] [7,9]`.

**Type/knobs:** interval merge / sweep · active state = running merged `end` · query = zero
coverage with positive gap.

**Time:** `O(N log N)`, `N` = total intervals (`O(N log K)` with a k-way heap merge over the
already-sorted per-employee lists).
**Space:** `O(N)`.

```java
public List<Interval> employeeFreeTime(List<List<Interval>> schedule) {
    List<Interval> all = new ArrayList<>();
    for (List<Interval> emp : schedule) all.addAll(emp);
    all.sort((a, b) -> a.start - b.start);

    List<Interval> res = new ArrayList<>();
    int end = all.get(0).end;                 // far edge of current busy block
    for (Interval iv : all) {
        if (iv.start > end) {                 // strict gap → free for everyone
            res.add(new Interval(end, iv.start));
            end = iv.end;
        } else {
            end = Math.max(end, iv.end);      // overlap → extend busy block
        }
    }
    return res;
}
```

**Tips.**
- The union of all busy intervals is the "count ≥ 1" region; free time is exactly the
  "count == 0" region with positive width — the boundaries of the result are start/end
  events where the balance touches zero.
- Use strict `>` for the gap so zero-length touches don't produce empty free intervals.

**Trick / variant.** For `O(N log K)`, seed a min-heap with the first interval of each
employee (lists are pre-sorted) and pop/advance — the k-way merge replaces the global sort.

---

## Meeting Rooms III — active state is two heaps ★

**LC 2402.** `n` rooms `0..n-1`. Meetings `[start, end]` processed in start order. Each
meeting takes the **lowest-numbered available** room. If none is free it **waits** for the
room that frees **earliest** (ties → lowest number), keeping its original duration. Return
the room that held the most meetings (ties → lowest number).

**Example:** `n=2`, `[[0,10],[1,5],[2,7],[3,4]]` → `0`.

**Intuition.** This graduates from *measuring* intervals to *simulating* an assignment
policy, so it is event-driven rather than a pure line sweep. Two heaps encode the two rules:
- `free` — a min-heap of free **room numbers** → gives "lowest-numbered available".
- `busy` — a min-heap keyed `(endTime, room)` → its top gives "frees earliest, lowest on
  tie".

For each meeting: reclaim every room in `busy` whose `endTime <= start`. If a room is free,
take the lowest and occupy it until `end`. Otherwise pop the earliest-freeing room, and the
meeting is delayed — it runs `soon.endTime + (end - start)`, i.e. **same duration, later
start** — then push it back.

**Type/knobs:** event-driven simulation · active state = `free` heap + `busy` heap ·
query = lowest free room / earliest-freeing room.

**Time:** `O(m log m + m log n)`, `m` = meetings.
**Space:** `O(n)` for the heaps and counts.

```java
public int mostBooked(int n, int[][] meetings) {
    Arrays.sort(meetings, (a, b) -> a[0] - b[0]);
    long[] count = new long[n];

    PriorityQueue<Integer> free = new PriorityQueue<>();          // room numbers
    for (int i = 0; i < n; i++) free.offer(i);
    PriorityQueue<long[]> busy = new PriorityQueue<>(             // (endTime, room)
        (a, b) -> a[0] != b[0] ? Long.compare(a[0], b[0]) : Long.compare(a[1], b[1]));

    for (int[] m : meetings) {
        long start = m[0], end = m[1];
        while (!busy.isEmpty() && busy.peek()[0] <= start)        // reclaim finished rooms
            free.offer((int) busy.poll()[1]);

        if (!free.isEmpty()) {
            int room = free.poll();                               // lowest free room
            busy.offer(new long[]{end, room});
            count[room]++;
        } else {
            long[] soon = busy.poll();                            // frees earliest, ties → lowest
            int room = (int) soon[1];
            long newEnd = soon[0] + (end - start);                // delayed start, same duration
            busy.offer(new long[]{newEnd, room});
            count[room]++;
        }
    }

    int best = 0;
    for (int i = 1; i < n; i++) if (count[i] > count[best]) best = i;
    return best;
}
```

**Tips (semantic-bug traps).**
- **Use `long` for end times.** A repeatedly-delayed meeting accumulates duration and can
  overflow `int` — a silent wrong answer, not a crash.
- **`busy.peek()[0] <= start`** uses `<=`: a room frees *at* its end time and can host a
  meeting starting that same instant. This is the identical touching-endpoint tie-break from
  every problem above.
- The `(endTime, room)` comparator resolves both "earliest free" and "lowest number on tie"
  in a single key.

**Trick / variant.** Process Tasks Using Servers (LC 2410) is the same two-heap pattern
with server weights instead of room numbers.

---

## Process Tasks Using Servers — the same two heaps, arrival time free ★

**LC 2410.** `servers[i]` = weight, `tasks[j]` = duration. **Task `j` arrives at second `j`.**
At its arrival, if a server is free, take the **lightest** (ties → **lowest index**). If none
is free, the task waits for the earliest-freeing server; waiting tasks are assigned **FIFO**.
Return `ans[j]` = the server index that ran task `j`.

**Example:** `servers = [3,3,2]`, `tasks = [1,2,3,2,1,2]` → `[2,2,0,2,1,2]`.

**Intuition.** Structurally identical to Meeting Rooms III — two heaps encoding two rules:
- `free` — min-heap keyed `(weight, index)` → "lightest, lowest index on tie".
- `busy` — min-heap keyed `(freeTime, weight, index)` → "frees earliest", and it carries
  `weight`/`index` so the server can be handed back to `free` intact.

The one genuinely new observation: **the task index *is* its arrival time.** Meeting Rooms III
has to `Arrays.sort(meetings)` by start; here `t = 0, 1, 2, …` *is* the sorted event order, so
the sort step is free. Same freebie as the difference array in LC 1589, where the coordinates
were already `0..n−1`.

**FIFO falls out for free** — it isn't coded anywhere. Because tasks are visited in index
order and each waiting task grabs the *next*-earliest-freeing server, assignment times come
out non-decreasing automatically. If no server is free at `t`, none is free at `t+1` either
(every `freeTime > t` and the one just taken is pushed back later still), so waiting tasks
queue up in exactly arrival order.

Enumerate the example (`free` shown as `(weight,index)`):

```
t=0  release none   free {(2,2),(3,0),(3,1)}  take (2,2)  → ans[0]=2  busy {(1,2,2)}
t=1  release (1,2,2) → free  take (2,2)       → ans[1]=2  busy {(3,2,2)}
t=2  release none   free {(3,0),(3,1)}  take (3,0) → ans[2]=0  busy {(3,2,2),(5,3,0)}
t=3  release (3,2,2) → free  take (2,2)       → ans[3]=2  busy {(5,2,2),(5,3,0)}
t=4  release none   free {(3,1)}  take (3,1)  → ans[4]=1  busy {(5,3,1),(5,2,2),(5,3,0)}
t=5  release all three → free  take (2,2)     → ans[5]=2
```

**Type/knobs:** event-driven simulation · active state = `free` heap + `busy` heap ·
query = lightest free server / earliest-freeing server.

**Time:** `O((n + m) log n)` — each server enters and leaves the heaps `O(m)` times total.
**Space:** `O(n)`.

```java
public int[] assignTasks(int[] servers, int[] tasks) {
    int n = servers.length, m = tasks.length;

    PriorityQueue<int[]> free = new PriorityQueue<>(                 // (weight, index)
        (a, b) -> a[0] != b[0] ? Integer.compare(a[0], b[0]) : Integer.compare(a[1], b[1]));
    for (int i = 0; i < n; i++) free.offer(new int[]{servers[i], i});

    PriorityQueue<long[]> busy = new PriorityQueue<>(                // (freeTime, weight, index)
        (a, b) -> a[0] != b[0] ? Long.compare(a[0], b[0])
                : a[1] != b[1] ? Long.compare(a[1], b[1]) : Long.compare(a[2], b[2]));

    int[] ans = new int[m];
    for (int t = 0; t < m; t++) {
        while (!busy.isEmpty() && busy.peek()[0] <= t) {              // <= : frees AT its end time
            long[] s = busy.poll();
            free.offer(new int[]{(int) s[1], (int) s[2]});
        }
        if (!free.isEmpty()) {
            int[] s = free.poll();                                    // lightest, ties → lowest idx
            ans[t] = s[1];
            busy.offer(new long[]{(long) t + tasks[t], s[0], s[1]});
        } else {
            long[] s = busy.poll();                                   // frees earliest
            ans[t] = (int) s[2];
            busy.offer(new long[]{s[0] + tasks[t], s[1], s[2]});      // starts when it frees
        }
    }
    return ans;
}
```

**Tips (the same two traps as Meeting Rooms III).**
- **`freeTime` must be `long`.** Worst case is one server running `2·10⁵` tasks of `2·10⁵`
  each → `4·10¹⁰`, well past `int`'s `2.1·10⁹`. It overflows *silently*, and a negative
  `freeTime` makes that server look like it frees first — wrong server, no exception.
- **`busy.peek()[0] <= t`** uses `<=`: a server frees *at* its end time and can take a task
  arriving that same second. The touching-endpoint tie-break, again.
- The `busy` entries carry `weight` and `index` so a released server re-enters `free` with its
  real key — don't try to look the weight up by index later, just carry it.

**Trick / variant.** Meeting Rooms III (LC 2402) is the same skeleton with room *numbers*
instead of `(weight, index)`, and it must sort by start because arrival times aren't indices.
Compare the two side by side — the diff is exactly "what's the key" and "is the order free".

---

## Number of Flowers in Full Bloom — binary search on two sorted arrays ★

**LC 2251.** `flowers[i] = [start, end]` means flower `i` blooms on `[start, end]`
**inclusive**. For each query time `people[j]`, return how many flowers are in bloom.

**Example:** `flowers = [[1,6],[3,7],[9,12],[4,13]]`, `people = [2,3,7,11]` → `[1,2,2,2]`.

**Intuition.** A flower is in bloom at time `t` iff `start <= t <= end`. Rewrite that as a
difference of two easy counts:

```
in bloom at t  =  (# flowers already started by t)  -  (# flowers already ended before t)
              =  count(start <= t)                  -  count(end < t)
```

The second term is a subset of the first (a flower can't end before it starts), so
subtracting is exact. Sort the `starts` and `ends` arrays independently — the times don't
need to stay paired — and answer each query with two binary searches. This is the *sorted-
two-arrays* form of the sweep, the same idea as Meeting Rooms II's two-pointer version, but
queried at arbitrary points instead of walked once.

Note the boundaries, driven by the **inclusive** end: `start <= t` (started counts a flower
starting exactly at `t`) but `end < t` (a flower ending exactly at `t` is *still* blooming,
so it must not be counted as ended).

Enumerate `t = 7` on the example: `starts = [1,3,4,9]`, `ends = [6,7,12,13]`.
`count(start <= 7) = 3` (1,3,4); `count(end < 7) = 1` (only 6). Answer `3 − 1 = 2` — flowers
`[3,7]` and `[4,13]` are open, `[1,6]` has closed, `[9,12]` hasn't opened.

**Type/knobs:** sweep via two sorted arrays · active state = cumulative started − ended ·
query = point lookup by binary search.

**Time:** `O((n + q) log n)` — sort the two arrays once, two binary searches per query.
**Space:** `O(n)` for the two arrays.

```java
public int[] fullBloomFlowers(int[][] flowers, int[] people) {
    int n = flowers.length;
    int[] starts = new int[n], ends = new int[n];
    for (int i = 0; i < n; i++) { starts[i] = flowers[i][0]; ends[i] = flowers[i][1]; }
    Arrays.sort(starts);
    Arrays.sort(ends);

    int[] res = new int[people.length];
    for (int j = 0; j < people.length; j++) {
        int t = people[j];
        res[j] = countLE(starts, t) - countLT(ends, t);  // started - ended
    }
    return res;
}

// number of elements <= key  (first index with a[i] > key)
private int countLE(int[] a, int key) {
    int lo = 0, hi = a.length;
    while (lo < hi) { int m = (lo + hi) >>> 1; if (a[m] <= key) lo = m + 1; else hi = m; }
    return lo;
}
// number of elements <  key  (first index with a[i] >= key)
private int countLT(int[] a, int key) {
    int lo = 0, hi = a.length;
    while (lo < hi) { int m = (lo + hi) >>> 1; if (a[m] <  key) lo = m + 1; else hi = m; }
    return lo;
}
```

### Cleaner: normalize the data, not the search (editorial form)

The two search variants exist only because the end is **inclusive**. Kill that at the source
— store `end + 1` instead of `end` — and `countLT(ends, t)` becomes `countLE(ends', t)`:

```
end + 1 <= t   ⟺   end < t          // identical predicate, shifted by one
```

So **one** binary search function serves both arrays. This is the same `end + 1` move as the
difference array in LC 1589 (`diff[r[1] + 1]--`): shift an inclusive end by one to make it
exclusive, then use uniform machinery everywhere.

```java
public int[] fullBloomFlowers(int[][] flowers, int[] people) {
    List<Integer> starts = new ArrayList<>(), ends = new ArrayList<>();
    for (int[] f : flowers) {
        starts.add(f[0]);
        ends.add(f[1] + 1);                 // <- inclusive end made exclusive
    }
    Collections.sort(starts);
    Collections.sort(ends);

    int[] ans = new int[people.length];
    for (int i = 0; i < people.length; i++) {
        int t = people[i];
        ans[i] = countLE(starts, t) - countLE(ends, t);   // ONE function, both arrays
    }
    return ans;
}
// number of elements <= target
private int countLE(List<Integer> a, int target) {
    int lo = 0, hi = a.size();
    while (lo < hi) {
        int m = (lo + hi) >>> 1;
        if (target < a.get(m)) hi = m;
        else lo = m + 1;
    }
    return lo;
}
```

**Tips.**
- Sorting `starts` and `ends` separately is the key move — you're not tracking *which*
  flower, only *how many*, so the pairing is irrelevant.
- The `<=` vs `<` split is the inclusive-endpoint tie-break in disguise. Either handle it in
  the **predicate** (two functions) or in the **data** (`end + 1`, one function). Prefer the
  data fix: one function is one place to get the boundary wrong.
- `end + 1` can't overflow here (`end ≤ 10⁹`, `int` holds `2.1·10⁹`) — but check that before
  reaching for the trick.
- Prefer `int[]` + `Arrays.sort` over `List<Integer>` + `Collections.sort`: no autoboxing per
  comparison, primitive dual-pivot quicksort instead of object TimSort, 4 bytes vs ~16 per
  element. Accepted either way at `n ≤ 5·10⁴`; the primitive version is the one to write.

**Trick / variant.** Equivalent event sweep: `+1` at each `start`, `-1` at each `end + 1`,
then sort queries with the events and sweep. The binary-search form avoids sorting the
queries and answers them in any order.

---

## Minimum Interval to Include Each Query — offline queries + lazy-deleted heap ★

**LC 1851.** `intervals[i] = [left, right]` (inclusive). For each `queries[j]`, return the
**size** (`right − left + 1`) of the *smallest* interval containing it, or `−1` if none does.

**Example:** `intervals = [[1,4],[2,4],[3,6],[4,4]]`, `queries = [2,3,4,5]` → `[3,3,1,4]`.

**Intuition.** Two ideas stack here.

**(1) Offline queries.** The queries arrive unsorted, but they're *independent* — nothing
evolves between them (the static-world property from LC 2251). So you may answer them in
**any order**. Sort them ascending, sweep, and write each answer back to its original index.
This is the move that makes a sweep possible at all; the original order is just bookkeeping.

**(2) A heap keyed on the answer, deleted on a different field.** Sweep queries left→right.
Push every interval whose `left <= q` into a **min-heap keyed on size**, carrying `right`
alongside. Now the top is the smallest interval among all that have *started* — but it might
have already *ended*. So before reading it, pop expired tops (`right < q`).

Why peeking is then correct: the heap holds every interval with `left <= q`. If the top has
`right >= q`, it contains `q`, and it's the min-size over a **superset** of the valid
intervals — so it's the min-size over the valid ones too.

Why the *lazy* deletion is safe (the crux): a heap can't remove from the middle, so you don't
try. Expired entries buried below the top are harmless — they're larger, so if one ever
surfaces you'll pop it then. And because queries are **monotone increasing**, an interval
expired at `q` can never revive for any later `q' > q`. Popping is permanent.

That monotonicity is the same licence as LC 2158's "painted → never unpainted": each interval
is pushed once and popped at most once, so the pops amortize to `O(n log n)` overall even
though a single query might pop many.

Enumerate the example. Intervals sorted by `left`, as `(size, right)`:
`[1,4]→(4,4)`, `[2,4]→(3,4)`, `[3,6]→(4,6)`, `[4,4]→(1,4)`.

```
q=2  push (4,4),(3,4)      heap top (3,4)  right 4 >= 2  → 3
q=3  push (4,6)            heap top (3,4)  right 4 >= 3  → 3
q=4  push (1,4)            heap top (1,4)  right 4 >= 4  → 1
q=5  push nothing          top (1,4) right 4 < 5 → pop
                           top (3,4) right 4 < 5 → pop
                           top (4,4) right 4 < 5 → pop
                           top (4,6) right 6 >= 5        → 4
```

Three pops at `q=5` — but each of those was pushed exactly once, which is the amortization
made visible.

**Type/knobs:** offline queries · active state = min-heap on size, lazy-deleted on `right` ·
query = min size among live intervals.

**Time:** `O(n log n + m log m)` — sorts dominate; each interval is pushed/popped once.
**Space:** `O(n + m)`.

```java
public int[] minInterval(int[][] intervals, int[] queries) {
    int m = queries.length;
    int[][] qs = new int[m][2];                       // (query value, original index)
    for (int i = 0; i < m; i++) { qs[i][0] = queries[i]; qs[i][1] = i; }
    Arrays.sort(qs, (a, b) -> Integer.compare(a[0], b[0]));
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));   // by left

    // min-heap on size; entry = (size, right)
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> Integer.compare(a[0], b[0]));

    int[] ans = new int[m];
    int i = 0;
    for (int[] q : qs) {
        int t = q[0];
        while (i < intervals.length && intervals[i][0] <= t) {       // started by t
            pq.offer(new int[]{intervals[i][1] - intervals[i][0] + 1, intervals[i][1]});
            i++;
        }
        while (!pq.isEmpty() && pq.peek()[1] < t) pq.poll();          // lazy-delete expired
        ans[q[1]] = pq.isEmpty() ? -1 : pq.peek()[0];                // write back to original slot
    }
    return ans;
}
```

**Tips.**
- Sort the `(value, index)` pairs, not the values — you must restore the original order. The
  `int[][]` pair form avoids the `Integer[]`-boxing dance that sorting an index array forces.
- The interval pointer `i` **never resets** across queries. That's what keeps the push phase
  linear overall; resetting it silently makes the whole thing `O(n·m)`.
- Two different orderings are live at once: the heap orders by **size**, expiry is decided by
  **right**. Naming that out loud is the fastest way to explain the solution.

**Trick / variant.** A second, very different `O((n + m) α)` route: sort *intervals* by size
ascending and sweep them, assigning each interval's size to every still-unanswered query in
`[left, right]` — using the pointer-jump / DSU skip from LC 2158. The first interval to reach
a query is the smallest, and each query is answered exactly once. Same amortization argument,
opposite axis.

---

## My Calendar III — difference-array sweep with a TreeMap ★

**LC 732.** Implement `book(start, end)` for half-open events `[start, end)`. After each
booking, return the maximum number of events that overlap at any single point (the max
"k-booking") across everything booked so far.

**Example:** `book(10,20)→1`, `book(50,60)→1`, `book(10,40)→2`, `book(5,15)→3`,
`book(5,10)→3`.

**Intuition.** This is the max-overlap sweep, kept live across insertions. Store a
**difference map** `TreeMap<coord, delta>`: each booking does `delta[start] += 1` and
`delta[end] -= 1`. Iterating the map in key order and running a prefix sum reconstructs the
overlap count at every boundary; the max of that prefix sum is the answer. Because the map is
sorted, a start and an end at the same coordinate land on the same key and their `+1/−1`
sum — which is exactly the half-open tie-break (`[a, x)` and `[x, b)` don't overlap) falling
out for free.

Enumerate after `book(5,15)`. The map is
`{5:+1, 10:+2, 15:−1, 20:−1, 40:−1, 50:+1, 60:−1}`; prefix sums `1, 3, 2, 1, 0, 1, 0` →
max `3`. Then `book(5,10)` makes key `5:+2` and key `10:+1` (the new `−1` end at 10 cancels
one of the starts there), giving prefix sums `2, 3, 2, 1, 0, 1, 0` → still `3`.

**Type/knobs:** overlap sweep, persisted · active state = `TreeMap` difference array ·
query = running max prefix sum.

**Time:** `O(n)` per `book` (sweep the whole map), `O(n^2)` over `n` bookings.
**Space:** `O(n)`.

```java
class MyCalendarThree {
    private final TreeMap<Integer, Integer> delta = new TreeMap<>();

    public int book(int start, int end) {
        delta.merge(start, 1, Integer::sum);   // one more event begins
        delta.merge(end, -1, Integer::sum);    // one event ends
        int cur = 0, max = 0;
        for (int d : delta.values()) { cur += d; max = Math.max(max, cur); }
        return max;
    }
}
```

**Tips.**
- The difference-array trick (`+1` at start, `−1` at end, prefix-sum) is the single most
  reusable sweep idiom — it's Car Pooling, Corporate Flight Bookings, and Maximum Population
  Year too.
- Don't prune zero entries here unless you also stop relying on their positions; harmless to
  leave them since a `0` delta doesn't change the prefix sum.

**Trick / variant.** The `O(log n)` optimum uses a segment tree with lazy propagation over
coordinate-compressed (or dynamic) ranges, doing range `+1` on `[start, end)` and querying
the global max. The `TreeMap` version is what you'd write first and is accepted; mention the
segment-tree upgrade when asked for optimality. My Calendar I (LC 729) / II (LC 731) are the
same map with a cap: reject a booking if it would push any point past `1` (I) or `2` (II).

---

## Amount of New Area Painted Each Day — union of intervals via pointer-jump ★

**LC 2158.** `paint[i] = [start, end]` paints the half-open segment `[start, end)` on day
`i`. Return `worklog[i]` = the amount of **new** area painted on day `i` (length not already
painted on an earlier day).

**Example:** `paint = [[1,4],[4,7],[5,8]]` → `[3,3,1]` (day 2 only adds unit `[7,8)` since
`[5,7)` was already covered).

**Intuition.** This is a *union of intervals* / covered-length problem rather than a strict
left-to-right sweep: each day you add an interval and must count only the part that wasn't
already in the union. With coordinates bounded (`end ≤ 5·10⁴`), the clean trick is a
**pointer-jump** structure (a union-find-flavored "next unpainted" map): `next[x]` is the
next still-unpainted coordinate `≥ x`. To paint `[start, end)`, hop from `find(start)`; each
landed coordinate is genuinely new — count it, mark it painted by pointing `next[x] → x+1`,
and jump again. Path compression makes each unit painted exactly once, so the total work is
near-linear over the whole input.

Enumerate day 2 (`[5,8)`) after days 0–1 painted `[1,7)`: `find(5)` compresses through the
already-painted run `5→6→7` and lands on `7`; `7 < 8` so count `1`, mark `next[7]→8`;
`find(8)=8` stops. New area `= 1`.

**Type/knobs:** interval union / covered length · active state = `next[]` unpainted-pointer ·
query = count of freshly covered units.

**Time:** `O((N + M) α)` ≈ linear, `N` = days, `M` = coordinate range.
**Space:** `O(M)` for the pointer array.

```java
class Solution {
    private int[] next;                       // next[x] = next unpainted coord >= x
    private int find(int x) {
        if (next[x] != x) next[x] = find(next[x]);   // path compression
        return next[x];
    }
    public int[] amountPainted(int[][] paint) {
        int MAX = 50001;
        next = new int[MAX + 1];
        for (int i = 0; i <= MAX; i++) next[i] = i;

        int[] res = new int[paint.length];
        for (int i = 0; i < paint.length; i++) {
            int end = paint[i][1], count = 0;
            for (int x = find(paint[i][0]); x < end; x = find(x + 1)) {
                count++;
                next[x] = x + 1;              // mark x painted → skip it next time
            }
            res[i] = count;
        }
        return res;
    }
}
```

**Tips.**
- Half-open `[start, end)` means you paint integer cells `start … end-1`; the loop condition
  `x < end` respects that.
- If coordinates were unbounded (up to `10⁹`), swap the array for a `TreeMap<Integer,
  Integer>` of painted `[l, r)` runs and merge on insert — same idea, `O(N log N)`.

**Trick / variant.** The TreeMap-of-runs version generalizes to Range Module (LC 715:
`addRange` / `removeRange` / `queryRange`) and to computing union length after each update.

---

## Minimum Number of Taps to Water a Garden — greedy interval cover ★

**LC 1326.** Garden spans `[0, n]`. Tap `i` (at position `i`) waters
`[i − ranges[i], i + ranges[i]]`. Return the minimum taps to cover all of `[0, n]`, or `−1`.

**Example:** `n = 5`, `ranges = [3,4,1,1,0,0]` → `1` (the tap at position 1 reaches
`[−3, 5]` → clamps to cover the whole garden).

**Intuition.** Honest classification: this is the **minimum-intervals-to-cover-a-segment**
greedy — the interval cousin of Jump Game II — not an event sweep. Convert each tap to an
interval `[l, r]`, and for each left endpoint remember the farthest right it reaches
(`maxRight[l]`). Then sweep a *reach frontier* left to right: from everything whose left
endpoint is `≤ curEnd`, jump `curEnd` to the farthest reachable right and spend one tap;
repeat until you cover `n`. If a round can't extend the frontier, there's an uncoverable gap
→ `−1`.

Enumerate `n = 7`, `ranges = [1,2,1,0,2,1,0,1]`. Reach-by-left-endpoint
`maxRight = [3,3,6,3,6,0,8,0]`. Frontier: from `≤0` reach `3` (tap 1) → `curEnd = 3`; from
`≤3` reach `6` (tap 4) → `curEnd = 6`; from `≤6` reach `8` (tap 7) → `curEnd = 8 ≥ 7`. Three
taps.

**Type/knobs:** interval cover / greedy frontier · active state = `curEnd` + `farthest` ·
query = farthest reach from left endpoints seen so far.

**Time:** `O(n)` — one pass to build `maxRight`, one frontier pass (each index visited once).
**Space:** `O(n)`.

```java
public int minTaps(int n, int[] ranges) {
    int[] maxRight = new int[n + 1];               // farthest right reachable from left endpoint l
    for (int i = 0; i <= n; i++) {
        int l = Math.max(0, i - ranges[i]);
        maxRight[l] = Math.max(maxRight[l], i + ranges[i]);
    }
    int taps = 0, curEnd = 0, farthest = 0, i = 0;
    while (curEnd < n) {
        while (i <= curEnd) {                       // absorb every tap starting within reach
            farthest = Math.max(farthest, maxRight[i]);
            i++;
        }
        if (farthest <= curEnd) return -1;          // no progress → gap in coverage
        curEnd = farthest;                          // commit one tap, extend the frontier
        taps++;
    }
    return taps;
}
```

**Tips.**
- Index `maxRight` by **left endpoint**, storing the max right — this collapses "which taps
  start here" into one number and is what makes the frontier pass `O(n)`.
- The `farthest <= curEnd` check is the `−1` case: the frontier is stuck, so some point in
  `[0, n]` can never be reached.

**Trick / variant.** Video Stitching (LC 1024) is the identical greedy on clip intervals.
The pure event-sweep alternative (sort intervals, sweep, extend the reach) works too but the
frontier greedy is tighter.

---

## Rectangle cluster — determine a rectangle from a diagonal *(geometric hashing, not a sweep)*

The unifying idea, and the reason 939 and 963 belong together: **a rectangle is pinned down
by one diagonal.** A diagonal is a pair of opposite corners. So instead of hunting for four
points at once (`O(n⁴)`), you reason about *pairs* of points (`O(n²)`) and let a hash table
supply the rest. What the hash stores is the only difference between the two problems.

### Minimum Area Rectangle (LC 939) — axis-aligned ★

**Points in the plane; find the min area of a rectangle with sides parallel to the axes, or
`0` if none.**

**Example:** `[[1,1],[1,3],[3,1],[3,3],[2,2]]` → `4` (corners `(1,1),(1,3),(3,1),(3,3)`).

**Intuition.** When the rectangle is axis-aligned, a diagonal gives the other two corners
*for free*. Pick two points forming a diagonal — different `x` **and** different `y` — say
`(x1,y1)` and `(x2,y2)`. The remaining corners can only be `(x1,y2)` and `(x2,y1)`. Drop all
points in a `HashSet`, and each diagonal pair is an `O(1)` two-lookup check.

Enumerate the diagonal `(1,1)–(3,3)`: the other corners must be `(1,3)` and `(3,1)`, both
present → rectangle, area `|1−3|·|1−3| = 4`. Every rectangle is found twice (once per
diagonal); harmless for a min.

**Type/knobs:** geometric hashing · pair = diagonal · lookup = the two derived corners.

**Time:** `O(n²)` — all pairs, `O(1)` per check.
**Space:** `O(n)` for the set.

```java
public int minAreaRect(int[][] points) {
    Set<Integer> seen = new HashSet<>();
    for (int[] p : points) seen.add(p[0] * 40001 + p[1]);   // encode (x,y), coords ≤ 4·10⁴

    int min = Integer.MAX_VALUE;
    for (int i = 0; i < points.length; i++)
        for (int j = i + 1; j < points.length; j++) {
            int x1 = points[i][0], y1 = points[i][1];
            int x2 = points[j][0], y2 = points[j][1];
            if (x1 != x2 && y1 != y2                          // must be a diagonal
                && seen.contains(x1 * 40001 + y2)            // corner (x1,y2)
                && seen.contains(x2 * 40001 + y1)) {         // corner (x2,y1)
                min = Math.min(min, Math.abs(x1 - x2) * Math.abs(y1 - y2));
            }
        }
    return min == Integer.MAX_VALUE ? 0 : min;
}
```

**Tips.**
- The `x * 40001 + y` encoding needs the multiplier `> max coordinate` (here `40000`) so no
  two `(x,y)` collide. A `long` key or `"x,y"` string works too; the int-pack is fastest.
- The `x1 != x2 && y1 != y2` guard is essential — a pair sharing a row or column is an *edge*,
  not a diagonal, and would derive a degenerate zero-area "rectangle".

**Trick / variant — the sweep-flavored one.** Sort points into columns by `x`, sweep columns
left→right. In each column, every pair of `y`-values is a candidate *vertical edge*; store
`(y1,y2) → last x seen` in a map. When the same `(y1,y2)` recurs in a later column, those two
columns close a rectangle of width `x − lastX`. This is a genuine sweep — the active state is
"vertical edges seen so far" — and it's the bridge from this cluster back to the rest of the
file. Same `O(n²)` worst case, often faster.

### Minimum Area Rectangle II (LC 963) — any orientation ★

**Same points, but the rectangle may be rotated to any angle. Return the min area (within
`1e-5`), or `0`.**

**Example:** `[[1,2],[2,1],[1,0],[0,1]]` → `2.0` (a square tilted 45°, center `(1,1)`).

**Intuition.** Now a diagonal *doesn't* hand you the other corners — there are infinitely many
orientations. So flip the move from **lookup** to **grouping**, using the two properties that
define a rectangle's diagonals:

1. they **bisect each other** — both diagonals share the same midpoint (center);
2. they are **equal length**.

Together, those two facts *are* "rectangle" (equal diagonals that bisect each other ⟺
rectangle). So: for every pair of points, compute `(center, diagonalLength²)` and bucket the
pair under that key. **Any two pairs in the same bucket are the two diagonals of a
rectangle** — their four endpoints are its corners.

Enumerate the example. Pair `(1,2)–(1,0)`: center `(1,1)`, length² `4`. Pair `(2,1)–(0,1)`:
center `(1,1)`, length² `4`. Same bucket → rectangle. Its corners are those four points;
adjacent sides from `(1,2)` are `(1,2)–(2,1)` and `(1,2)–(0,1)`, each `√2`, so
area `= √2 · √2 = 2`.

**Type/knobs:** geometric hashing · pair = diagonal · group by (center, length²), match = a
second diagonal.

**Time:** `O(n²)` to form buckets, plus `O(Σ kᵢ²)` to pair within buckets — fine at `n ≤ 50`.
**Space:** `O(n²)` for the buckets.

```java
public double minAreaFreeRect(int[][] points) {
    int n = points.length;
    // key: (2·centerX, 2·centerY, len²) — double the center to stay integer
    Map<String, List<int[]>> byDiag = new HashMap<>();
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++) {
            int cx = points[i][0] + points[j][0];            // = 2·centerX
            int cy = points[i][1] + points[j][1];            // = 2·centerY
            long d2 = sq(points[i][0] - points[j][0]) + sq(points[i][1] - points[j][1]);
            byDiag.computeIfAbsent(cx + "," + cy + "," + d2, k -> new ArrayList<>())
                  .add(new int[]{i, j});
        }

    double ans = Double.MAX_VALUE;
    for (List<int[]> diags : byDiag.values())
        for (int a = 0; a < diags.size(); a++)
            for (int b = a + 1; b < diags.size(); b++) {
                int[] c = points[diags.get(a)[0]];           // one corner
                int[] p = points[diags.get(b)[0]];           // adjacent corner (other diagonal)
                int[] q = points[diags.get(b)[1]];           // the other adjacent corner
                ans = Math.min(ans, dist(c, p) * dist(c, q));   // perpendicular sides
            }
    return ans == Double.MAX_VALUE ? 0.0 : ans;
}
private long sq(int v) { return (long) v * v; }
private double dist(int[] a, int[] b) {
    return Math.hypot(a[0] - b[0], a[1] - b[1]);
}
```

**Tips.**
- **Store `2·center`, not `center`** — the midpoint of two integer points can be a half, and
  doubling keeps the key exact integers, dodging float-equality-as-a-hash-key (a real bug
  source). Same reason `len²` is kept squared: integer, no `sqrt` rounding in the key.
- Area = product of two *adjacent* sides meeting at one corner (they're perpendicular in a
  rectangle), **not** the diagonals. Picking `c` from diagonal A and `p, q` from diagonal B
  gives exactly those two sides.
- `n ≤ 50` here, so `O(n²)` pairs is ~1225 — brute grouping is comfortably fast; don't
  over-engineer.

**Trick / variant.** The same "hash a geometric invariant" template detects squares (add a
"sides equal" check), counts rectangles instead of minimizing (sum `C(kᵢ,2)` per bucket), or
finds min-area triangles. The invariant changes; the pair-and-bucket skeleton doesn't.

---

## One-line takeaway

Turn objects into events, sort, sweep while maintaining an active state — and the whole
family differs only in *what that state is* and *what you query from it*. For any new
interval problem, ask: **what is my active state, and does touching count as overlap?**
