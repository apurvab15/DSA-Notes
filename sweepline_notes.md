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
| two heaps (`free` + `busy`) | lowest free / earliest-freeing | Meeting Rooms III |

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
- My Calendar I / II / III (LC 729 / 731 / 732) — booking with overlap limits
- Number of Flowers in Full Bloom (LC 2251) — sweep / binary search on starts & ends
- Maximum Population Year (LC 1854) — tiny classic sweep
- Check if All the Integers in a Range Are Covered (LC 1893)

**Merging / gaps**
- Merge Intervals (LC 56)
- Insert Interval (LC 57)
- Interval List Intersections (LC 986)
- Employee Free Time (LC 759) ★

**2D geometry**
- The Skyline Problem (LC 218) ★
- Rectangle Area II (LC 850) ★
- Falling Squares (LC 699)
- Line-segment intersection (Bentley–Ottmann)

**Simulation / assignment (event-driven, not pure sweep)**
- Meeting Rooms III (LC 2402) ★
- Process Tasks Using Servers (LC 2410)

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

## One-line takeaway

Turn objects into events, sort, sweep while maintaining an active state — and the whole
family differs only in *what that state is* and *what you query from it*. For any new
interval problem, ask: **what is my active state, and does touching count as overlap?**
