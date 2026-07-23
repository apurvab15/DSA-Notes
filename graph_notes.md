# Graph Notes

> Companion to `dp_notes.md`. Same philosophy: state the idea in one English sentence, derive the code from the idea, and know exactly why the complexity is what it is.
>
> **Read Section 0 first** — it's the decision procedure that routes to every other section, plus what to do when you're stuck. It's protocol, not new material, so it's also the right thing to reread on a taper day.

---

## 0. How to Decide What to Use (read this first)

The most common failure mode is pattern-matching on the **surface of the input** instead of the question. "It's a grid" triggers a BFS reflex — but the same grid is BFS, Dijkstra, DP, or DSU depending entirely on what's being *asked*. The input shape tells you almost nothing. Three questions, in order, do.

### Step 1 — Classify the verb, not the noun

The question's verb picks the algorithm family, and it's usually decidable from one sentence of the problem statement.

| The question asks... | Family | Because |
|---|---|---|
| "Is there a path / are they connected / how many groups" | **DSU (Section 8) or DFS (Section 6)** | Membership only — no distance needed |
| "Shortest / minimum steps / fewest / earliest" | **Shortest-path family (Section 3–5)** | Needs the distance guarantee |
| "Order / before / prerequisite / valid sequence" | **Topological sort (Section 7)** | Precedence constraints |
| "All / count / enumerate / every path" | **DFS + backtracking (Section 6)** | Exhaustive; call stack carries the path |
| "Best value achievable over paths" (longest, max sum, # of ways) | **DP on a DAG (Section 6, LC 329)** | Optimization with reusable subproblems |
| "Detect a cycle" | **DSU** (undirected) / **Kahn's or 3-color DFS** (directed) | Direction changes the tool entirely |

If you can't name the verb, you don't understand the problem yet — that's a signal to re-read, not to start coding.

### Step 2 — Read the cost structure

Once you're in the shortest-path family, the **edge weights** pick the variant. Strict ladder; each rung generalizes the one above:

```
no weights (every move costs 1)   → BFS            O(V+E)      queue
weights ∈ {0, 1}                  → 0-1 BFS        O(V+E)      deque
arbitrary weights ≥ 0             → Dijkstra       O(E log V)  min-heap
any negative weight               → Bellman-Ford   O(V·E)
```

Always take the **cheapest rung that fits**. Dijkstra on an unweighted graph isn't wrong, but it's a `log` factor of noise and signals you didn't notice the structure.

Then the subtler question: **what operator combines an edge with the running path cost?** Almost everyone assumes `+`:

| Operator | Problem type | Example |
|---|---|---|
| `+` | ordinary shortest path | Network Delay Time |
| `max` | minimax — minimize the largest edge | Swim in Rising Water |
| `min` | maximin — maximize the smallest value | Path With Maximum Minimum Value |
| `×` | maximize probability (max-heap) | Path with Maximum Probability |

Dijkstra's correctness needs only that the operator is **monotonic** — extending a path can never improve its cost. That's why the template survives every one of these swaps. Asking "sum, or something else?" catches a whole class of "hard" problems that are the template with one line changed.

### Step 3 — Test whether your state is sufficient

This is what separates mediums from hards, and the test is mechanical:

> **Can two different ways of arriving at the same node lead to different futures?**

If yes, the node alone is **not** the state — augment it.

| Problem | Two arrivals differ by... | State |
|---|---|---|
| Cheapest Flights Within K Stops | stops used (2 vs 4) | `(city, stopsUsed)` |
| Shortest Path with Obstacles Elimination | eliminations left | `(cell, eliminationsUsed)` |
| Shortest Path to Get All Keys | keys held | `(cell, keyBitmask)` |

The tell in the statement is a **side constraint**: "at most K," "you may remove up to," "while carrying." When you see one, immediately ask whether it belongs in the state. The algorithm doesn't change — BFS is still BFS — you just widen the tuple in the queue and in `visited`/`dist`.

The mirror error matters too: **augmenting when you don't need to** multiplies the search space for nothing. If arrival history genuinely doesn't affect the future, the plain node is the state.

### Step 4 — If Steps 1–3 don't crack it, remodel the graph

Nearly every hard graph problem is a familiar template plus one of these seven moves:

1. **Reverse the edges.** Many-to-many becomes few-to-many. Pacific Atlantic: instead of "can each cell reach both oceans?" (O((M·N)²)), DFS *inward from the borders* — two sweeps, intersect.
2. **Add an imaginary super-source.** Multi-source BFS is exactly this: one node joined to every source by a 0-cost edge. Any "distance to nearest X" is a super-source problem.
3. **Redefine what a node is.** Nodes needn't be given. Open Lock: a node is a 4-digit *string*. Bitmask problems: `(position, visitedSet)`. When the given nodes don't support the question, invent better ones.
4. **Swap the aggregation operator.** Step 2 — sum → max / min / product.
5. **Reverse time.** Deletions become additions, which DSU can handle. Bricks Falling When Hit is the archetype.
6. **Sort to manufacture an order.** LC 329's topological order *is* the cell value. Offline queries sort by threshold so edges are only ever added.
7. **Binary search on the answer + feasibility check.** "Minimize the maximum X" → ask instead "for a fixed X, is it feasible?", which is usually a plain BFS/DFS reachability test. Same monotone-predicate move as `binarysearch_notes.md`.

**Flag for move 7:** *"minimize the maximum" / "maximize the minimum" is a two-solution problem every single time* — binary search + reachability, or Dijkstra with a swapped operator. Naming both is strong signal.

### The 30-second protocol (slots into step 2 of the six-step interview protocol)

1. **Name the nodes and edges out loud**, especially when implicit. *"Each cell is a node; edges go to the 4 neighbors; the cost of an edge is the neighbor's elevation."*
2. **Name the verb** → family.
3. **Name the cost structure** → rung on the ladder + which operator.
4. **Ask the sufficiency question** → augment the state, or explicitly say you don't need to.
5. **State the algorithm and complexity** → then code.

Steps 1 and 4 are the ones that get skipped under pressure, and they're where hard problems are won. "Let me define what a node is here" buys thinking time *and* reads as senior.

---

## 0b. Contingency Plan — What To Do When You're Stuck

Silence is the only truly bad outcome. A wrong idea narrated clearly is recoverable; three quiet minutes are not. The interviewer is grading your *process*, and they cannot grade what they cannot hear.

### The escalation ladder

Work down this list. Each rung takes ~60–90 seconds; most blocks break by rung 3.

**Rung 1 — Say where you are.** Before anything else, externalize:
> *"Let me say where I am: I'm confident this is a shortest-path problem, but I'm not sure yet whether the K-constraint needs to go into the state. Let me think about that out loud."*

This costs nothing, buys thinking time, and often prompts a free nudge. It also converts a scary silence into visible structured thinking.

**Rung 2 — Solve a smaller or easier version.** Drop a constraint and solve *that*:
- Ignore the side constraint entirely → does plain BFS/Dijkstra work? Then re-add it and ask what breaks.
- Solve for `n = 2` or a 1×3 grid by hand.
- Solve the *unweighted* version, then generalize.

Say it explicitly: *"Let me first solve it without the 'at most K stops' constraint, then see what changes."* Relaxing a constraint and reintroducing it is a legitimate, senior-looking technique — not an admission of defeat.

**Rung 3 — Brute force out loud, then optimize.** State the O(n²) or exponential approach and its complexity, *then* attack it:
> *"The brute force is a BFS per query, O(q·(V+E)) — too slow at 10⁵. So I need to preprocess into something answering queries in O(1). That points at either precomputed component labels or a DSU."*

A stated brute force is worth real credit and gives you something concrete to improve. **Never** let the first thing you say be "I don't know."

**Rung 4 — Walk a concrete example by hand.** Take the given example, or invent a 3×3, and trace what the answer *should* be. Ask yourself aloud: *"why is the answer 4 and not 3 — what specifically forced that?"* The forcing reason is usually the algorithmic insight. This is your strongest personal technique; you catch things by tracing that you don't catch by staring.

**Rung 5 — Run the remodeling checklist.** Out loud, quickly: *"Can I reverse the edges? Add a super-source? Redefine what a node is? Is the cost a sum or a max? Should I binary search on the answer?"* Naming the checklist is itself signal, and one of the seven usually fires.

**Rung 6 — Ask a targeted question.** See below. Do this by ~5 minutes of being stuck, not 15.

### Recovering from a wrong turn

If you're 10 minutes into an approach that isn't working, **say so and pivot deliberately**:

> *"I've been building this as a plain BFS, but I now see that revisiting a cell with fewer eliminations left can be better, so node-only state isn't sufficient. I want to switch to an augmented `(cell, used)` state. That's about 5 lines different — can I take a minute to redo it?"*

Catching your own error and naming *why* it was wrong is a **positive** signal. Silently limping forward with a broken approach is the failure mode. Interviewers are far more forgiving of "I was wrong, here's the fix" than of a solution that doesn't work and isn't acknowledged.

### If you're stuck on implementation rather than approach

Different problem, different fix. Apply your three-pass debugging protocol out loud — the noun check (name every array and index, tag arrays by what they hold), the arrow check (read every comparator and `Math.max`/`Math.min` for direction), the seed check (initialization and answer-read placement against n=1). Narrating that you have a *systematic* debugging method is worth as much as the fix itself.

If a specific detail is what's blocking you (exact `PriorityQueue` comparator syntax, `TreeMap` method name), just ask: *"I'd normally look up the exact signature here — may I write it as a comment and move on?"* Almost every interviewer says yes. Burning four minutes on syntax is a far worse outcome than asking.

### Time-awareness rules of thumb (45-min screen)

| By this point | You should have... |
|---|---|
| ~5 min | Clarified the problem, stated brute force + complexity |
| ~10 min | Committed to an approach and stated its complexity |
| ~30 min | Working code |
| ~35 min | Dry run done, unprompted |
| remainder | Follow-ups, optimizations, edge cases |

If you hit 15 minutes without an approach, **escalate to a question immediately.** A hint costs you far less than an unfinished problem. Interviewers expect to give hints; how you *use* one is part of the evaluation.

---

## 0c. Questions to Ask the Interviewer

Questions divide into two kinds and it's worth knowing which you're asking. **Clarifying questions** (up front, cheap, expected) and **unsticking questions** (mid-problem, strategic). Your notes flag *over-asking* as a phone-screen failure mode — so front-load 2–3 sharp clarifiers, not eight, and make each one carry information.

### Up-front clarifiers — ask 2–3, then start

The good ones prove you're already thinking about the algorithm. Pick the ones whose answer would actually change your approach:

- **"Is the graph directed or undirected?"** — changes cycle detection and whether DSU is even viable.
- **"Can edge weights be negative?"** — the Dijkstra/Bellman-Ford fork.
- **"Roughly how large are `n` and the number of queries?"** — *the single most valuable question.* 10³ vs 10⁵ decides whether O(n²) preprocessing is acceptable, and constraints are the strongest hint available about the intended complexity.
- **"Can the graph be disconnected? Are self-loops or duplicate edges possible?"** — drives the driver loop and edge-case list.
- **"Are the nodes labeled `0..n-1`, or arbitrary?"** — array vs `HashMap` representation.
- **"Should I return any valid answer, or is there a tie-break rule?"** — e.g. lexicographically smallest in Alien Dictionary; changes `Queue` to `PriorityQueue`.
- **"Is the input already sorted / are there guarantees I can rely on?"** — exactly the question that unlocks LC 3532.

**Skip** questions whose answer can't change your code ("can I assume valid input?"). One well-chosen constraint question beats five generic ones.

### Mid-problem unsticking questions — how to ask without sounding lost

The framing rule: **always show your work first, then ask a narrow question.** A question that reveals thinking is nearly free; a bare "I'm stuck, can you help?" is expensive.

Template: *"Here's what I've established… here's my specific uncertainty… does that direction seem reasonable?"*

Concrete versions:

- **"I have an O(q·(V+E)) BFS-per-query solution. Given the constraints, I assume you want something closer to O(1) per query after preprocessing — should I optimize, or would you rather I code the brute force first?"**
  Shows you know it's suboptimal, and lets them tell you what they actually want to see. Sometimes the answer is "brute force is fine, let's move on."

- **"I see two directions — DSU for connectivity, or exploiting the sortedness for a single labeling pass. I lean toward the second because it's O(n + q). Would you like me to go that way?"**
  Presenting *alternatives with a reasoned preference* is the strongest possible way to ask for a hint. It's indistinguishable from thinking out loud, and they'll usually steer you.

- **"Is the K-constraint meant to be part of the state, or is there a way to avoid the extra dimension?"**
  Narrow and technical — signals you've already identified the crux.

- **"Am I overcomplicating this? I want to check my read of the problem before I invest in this approach."**
  Fine to use once. Interviewers generally answer honestly, and it can save ten minutes.

- **"Would you like me to optimize this further, or move on to testing?"**
  Useful near the end — respects their agenda and surfaces what they still need to see.

### On receiving a hint

Take it, use it visibly, and say what it unlocked: *"That's helpful — since `nums` is sorted, the non-adjacent edges are all redundant, which means components are contiguous."* Absorbing a hint and running with it scores well. Politely resisting a hint scores badly.

### Closing questions (if there's time at the end)

Only if the coding is genuinely finished. Two or three, real ones:
- "What does the team you're on work on day to day?"
- "What's the biggest technical challenge the team is facing right now?"
- "What does the rest of the interview process look like from here?"

Skip these if the coding ran long — nobody penalizes a candidate for using the whole slot on the problem.

---

## 1. Graph Representation in Java

There are three shapes you'll see in interviews. Pick based on how the input arrives — don't convert unless it helps.

### 1a. Adjacency list from an edge list (most common)

```java
// n nodes labeled 0..n-1, edges[i] = {u, v}
List<List<Integer>> buildGraph(int n, int[][] edges) {
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
    for (int[] e : edges) {
        adj.get(e[0]).add(e[1]);
        adj.get(e[1]).add(e[0]);   // drop this line for a directed graph
    }
    return adj;
}
```

**When node labels aren't 0..n-1** (strings, sparse ids), use a map:

```java
Map<String, List<String>> adj = new HashMap<>();
adj.computeIfAbsent(u, k -> new ArrayList<>()).add(v);
```

### 1b. Grid as an implicit graph

An `m × n` grid **is** a graph — you just never materialize the adjacency list. Each cell `(r, c)` is a node; its neighbors are generated on the fly with a directions array:

```java
int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};   // add diagonals if the problem says so

for (int[] d : DIRS) {
    int nr = r + d[0], nc = c + d[1];
    if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;  // bounds check FIRST
    // ... then visited / wall checks
}
```

### 1c. Weighted graph (for Dijkstra later)

```java
List<List<int[]>> adj;         // adj.get(u) = list of {v, weight}
adj.get(u).add(new int[]{v, w});
```

### Visited tracking

- Nodes `0..n-1` → `boolean[] visited`
- Grid → `boolean[][] visited`, or mutate the grid in place (e.g., flip `'1'` → `'0'`) if allowed
- Arbitrary labels / states → `Set<String>` or encode `(r, c)` as `r * n + c` into a `Set<Integer>`

---

## 2. Example Test Cases to Always Mention

Saying these out loud before coding is free interview signal.

### Empty graph
- `n = 0`, `edges = []` → loops over nothing; answer is usually `0` / empty list. Make sure you don't index into `adj.get(0)`.
- `n = 1`, `edges = []` → a single isolated node. Connected-components count = 1, not 0. Classic off-by-one.

### M × N grid
- `grid = [[]]` or `grid.length == 0` → guard at the top: `if (grid == null || grid.length == 0) return ...;`
- `1 × 1` grid → start == target; BFS should return `0` (or `1` if the problem counts cells, e.g., LC 1091). Read the problem's distance definition carefully.
- `1 × n` single row → catches bugs where you swapped `m`/`n` or `r`/`c`.
- Grid where the start or end cell is blocked → answer is `-1` immediately.

---

## 3. BFS

### Intuition

**One sentence:** BFS explores the graph in expanding rings — all nodes at distance `k` are fully processed before any node at distance `k+1` — so the *first* time you reach a node, you've reached it by a shortest path (in edge count).

That last clause is the whole reason BFS exists: **on unweighted graphs, BFS = shortest path.** The queue is a conveyor belt that never lets a far node cut in front of a near one.

**Concrete enumeration** — grid, start `S` at (0,0):

```
S . .        ring 0: {(0,0)}                      dist 0
. . .        ring 1: {(0,1), (1,0)}               dist 1
. . .        ring 2: {(0,2), (1,1), (2,0)}        dist 2
             ring 3: {(1,2), (2,1)}               dist 3
             ring 4: {(2,2)}                      dist 4
```

Each ring is exactly one `for (int i = 0; i < size; i++)` sweep of the queue in level-order BFS.

### Boilerplate — adjacency list

```java
int bfs(List<List<Integer>> adj, int start, int target) {
    Queue<Integer> queue = new ArrayDeque<>();
    boolean[] visited = new boolean[adj.size()];
    queue.offer(start);
    visited[start] = true;              // mark when ENQUEUING, not when dequeuing
    int dist = 0;

    while (!queue.isEmpty()) {
        int size = queue.size();        // freeze this level's size
        for (int i = 0; i < size; i++) {
            int node = queue.poll();
            if (node == target) return dist;
            for (int next : adj.get(node)) {
                if (!visited[next]) {
                    visited[next] = true;
                    queue.offer(next);
                }
            }
        }
        dist++;                          // one ring finished
    }
    return -1;
}
```

### Boilerplate — grid

```java
int bfs(int[][] grid, int[] start, int[] target) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    Queue<int[]> queue = new ArrayDeque<>();
    boolean[][] visited = new boolean[m][n];
    queue.offer(start);
    visited[start[0]][start[1]] = true;
    int dist = 0;

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] cell = queue.poll();
            if (cell[0] == target[0] && cell[1] == target[1]) return dist;
            for (int[] d : DIRS) {
                int nr = cell[0] + d[0], nc = cell[1] + d[1];
                if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
                if (visited[nr][nc] || grid[nr][nc] == 1) continue;  // 1 = wall (adjust per problem)
                visited[nr][nc] = true;
                queue.offer(new int[]{nr, nc});
            }
        }
        dist++;
    }
    return -1;
}
```

**The #1 BFS bug:** marking visited at *dequeue* time instead of *enqueue* time. It's still correct, but the same node gets enqueued many times → queue blows up, TLE on large inputs. Mark the moment you `offer`.

**Time:** O(V + E) for adjacency list — every node enqueued once, every edge scanned once. O(M·N) for a grid (V = M·N, E ≤ 4·M·N).
**Space:** O(V) — visited array plus the queue; the queue's worst case is one whole ring, which can be O(V) (grid diagonal ring ≈ O(min(M, N)) typical, O(M·N) worst).

### Multi-source BFS

**One sentence:** instead of one starting point, seed the queue with *all* sources at distance 0 before the loop starts — the rings then expand from every source simultaneously, and each node's BFS distance is its distance to the *nearest* source.

Mental model: it's ordinary BFS on a graph with an imaginary "super-source" connected to every real source by a 0-cost edge. Nothing else changes — same loop, same complexity.

**When to reach for it:** the question asks "for each cell, distance to the nearest X" or "how long until X spreads everywhere." If you catch yourself planning to run BFS once per source (O(k · V·E)), stop — one multi-source pass does it in O(V + E).

```java
// Only the seeding changes:
for (int r = 0; r < m; r++)
    for (int c = 0; c < n; c++)
        if (grid[r][c] == SOURCE) {
            queue.offer(new int[]{r, c});
            visited[r][c] = true;
        }
// ...then the identical level-order loop as above
```

### Example problem — Rotten Oranges (LC 994, Medium)

> Grid of empty cells (0), fresh oranges (1), rotten oranges (2). Every minute, fresh oranges adjacent to rotten ones rot. Return minutes until no fresh orange remains, or -1.

**Why it's multi-source:** every rotten orange rots its neighbors *in the same minute*. Rot spreads from all sources in parallel — that's exactly the expanding-rings picture with multiple centers. One ring = one minute.

**Concrete enumeration:**

```
2 1 1        minute 0: rotten = {(0,0)}          fresh = 6
1 1 0        minute 1: (0,1), (1,0) rot          fresh = 4
0 1 1        minute 2: (0,2), (1,1) rot          fresh = 2
             minute 3: (2,1) rots                fresh = 1
             minute 4: (2,2) rots                fresh = 0  → answer 4
```

**Structure of the solution:** seed queue with all `2`s, count fresh oranges, run level-order BFS decrementing the fresh count as you rot; answer is the number of completed rings, `-1` if `fresh > 0` at the end. The fresh counter is what saves you a final full-grid scan.

**Edge cases:** no fresh oranges at all → answer `0` (not -1, and be careful not to count an extra minute); a fresh orange sealed off by walls → `-1`.

**Time:** O(M·N) **Space:** O(M·N)

### Example problem — Shortest Distance from All Buildings (LC 317, Hard)

> Grid with buildings (1), obstacles (2), empty land (0). Find an empty cell minimizing the total travel distance to *all* buildings; you can only walk through empty land. Return -1 if impossible.

**Why it's the natural "hard" step up:** it's *k separate* BFS runs, one per building, accumulating two things per empty cell: `distSum[r][c]` (total distance from all buildings so far) and `reach[r][c]` (how many buildings can reach it). The answer is the min `distSum` over cells with `reach == numBuildings`.

**Key insight / pruning:** after each building's BFS, any empty cell *not* reached by that building can never be the answer. A classic trick: instead of a visited array per BFS, walk on cells whose value equals `0, -1, -2, ...` (decrementing a "walkable" marker each round) so unreachable cells are automatically excluded from later rounds.

**Why not multi-source here?** Multi-source BFS gives distance to the *nearest* source; this problem needs the *sum* over all sources — different aggregate, so the sources can't be merged into one queue. Knowing when multi-source *doesn't* apply is as important as knowing when it does.

**Time:** O(k · M·N) where k = number of buildings (each BFS is O(M·N)).
**Space:** O(M·N) for the two accumulator grids plus the queue.

### Bi-directional BFS

**One sentence:** run BFS from the start *and* from the target simultaneously, always expanding the smaller frontier, and stop the moment the two frontiers touch.

**Why it's fast:** if the answer path has length `d` and branching factor is `b`, one-directional BFS explores ~`b^d` states; two searches meeting in the middle explore ~`2·b^(d/2)`. For `b = 8, d = 10`: `8^10 ≈ 10^9` vs `2·8^5 ≈ 65k`. It's an exponent cut, not a constant-factor trick.

**Requirements:** you must know the target explicitly, and edges must be traversable in reverse (undirected, or you can invert the graph). Use `Set`s instead of a queue so you can check `endSet.contains(next)` in O(1).

```java
int bidirectionalBFS(Set<String> dead, String start, String target) {
    Set<String> begin = new HashSet<>(List.of(start));
    Set<String> end   = new HashSet<>(List.of(target));
    Set<String> visited = new HashSet<>(dead);   // treat deadends as pre-visited
    int steps = 0;

    while (!begin.isEmpty() && !end.isEmpty()) {
        if (begin.size() > end.size()) {          // always expand the smaller side
            Set<String> tmp = begin; begin = end; end = tmp;
        }
        Set<String> next = new HashSet<>();
        for (String cur : begin) {
            for (String nb : neighbors(cur)) {    // problem-specific move generator
                if (end.contains(nb)) return steps + 1;   // frontiers met
                if (visited.add(nb)) next.add(nb);
            }
        }
        begin = next;
        steps++;
    }
    return -1;
}
```

#### Open the Lock (LC 752, Medium)

> 4-wheel lock, each wheel 0–9, one move turns one wheel ±1 (with wraparound 9↔0). Given deadend combinations and a target, find the minimum moves from "0000", or -1.

**Graph framing:** each 4-digit string is a node; each node has exactly 8 neighbors (4 wheels × 2 directions). 10,000 nodes total. It's shortest-path-in-an-unweighted-graph → BFS; and since start and target are both explicit strings, it's the canonical bidirectional showcase.

**Neighbor generation** (the only problem-specific piece):

```java
List<String> neighbors(String s) {
    List<String> res = new ArrayList<>();
    for (int i = 0; i < 4; i++) {
        char[] a = s.toCharArray();
        char orig = a[i];
        a[i] = orig == '9' ? '0' : (char)(orig + 1);  res.add(new String(a));
        a[i] = orig == '0' ? '9' : (char)(orig - 1);  res.add(new String(a));
    }
    return res;
}
```

**Edge cases:** "0000" itself is a deadend → -1 immediately; target == "0000" → 0.

**Time:** O(V + E) worst case = O(10^4 · 8), but bidirectional typically touches a tiny fraction.
**Space:** O(V) for the sets.

#### Word Ladder (LC 127, Hard)

> Transform `beginWord` into `endWord`, changing one letter at a time, every intermediate word must be in the dictionary. Return the length of the shortest transformation sequence.

**Graph framing:** words are nodes, edges connect words differing in one letter. The expensive part is neighbor generation: comparing all pairs is O(N² · L). Two standard fixes:

1. **Try all 26 letters per position** — for each word, mutate each of L positions to each of 26 letters and check set membership: O(L · 26) per node. Wins when the dictionary is large.
2. **Wildcard buckets** — preprocess every word into patterns like `h*t`, `ho*`, `*ot`; words sharing a pattern are neighbors. Wins for repeated queries.

**Why bidirectional shines here:** branching factor can be huge (25 per position), so cutting the depth in half is dramatic — the classic benchmark shows ~10–40× speedups. Also note: if `endWord` isn't in the dictionary, return 0 *before* doing any work.

**Careful with the distance definition:** LC 127 counts *words in the sequence* (so `hit → hot → dot → dog → cog` = 5), not edges (4). Off-by-one trap.

**Time:** O(N · L · 26) with trick 1 (N = dict size, L = word length).
**Space:** O(N · L) for the visited set and frontiers.

---

## 4. 0-1 BFS

### Intuition

**One sentence:** 0-1 BFS computes shortest paths on a graph where every edge weighs exactly **0 or 1**, in O(V + E) — it's the bridge between plain BFS and Dijkstra, using a **deque** instead of a priority queue.

**Why plain BFS breaks.** BFS's correctness rests on one invariant: everything in the queue is at distance `d` or `d+1`, in order — because one edge = one unit of cost, so one ring = one distance value. A 0-weight edge violates this: following it, the neighbor is at the *same* distance as you, not one more. If you `offer()` it to the back, it lands *behind* nodes that are actually farther, and "first arrival = shortest" collapses.

**The fix — front for 0, back for 1.** When you relax an edge:
- weight **0** → push the neighbor to the **front** of the deque (same ring, process it now),
- weight **1** → push to the **back** (next ring, as usual).

This preserves the sorted-queue invariant: the deque always holds at most **two distinct distance values**, non-decreasing from front to back. That's exactly Dijkstra's "always expand the closest node" property — but because there are only two possible edge increments, the deque gives you the ordering for free, so no `log V` heap. That's the whole trick.

### Boilerplate — adjacency list

```java
int zeroOneBFS(List<List<int[]>> adj, int start, int target) {  // adj.get(u) = list of {v, weight∈{0,1}}
    int n = adj.size();
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    Deque<Integer> deque = new ArrayDeque<>();
    dist[start] = 0;
    deque.offerFirst(start);

    while (!deque.isEmpty()) {
        int u = deque.pollFirst();
        for (int[] e : adj.get(u)) {
            int v = e[0], w = e[1];
            if (dist[u] + w < dist[v]) {          // relaxation IS the guard
                dist[v] = dist[u] + w;
                if (w == 0) deque.offerFirst(v);  // free move → current ring
                else        deque.offerLast(v);   // costed move → next ring
            }
        }
    }
    return dist[target] == Integer.MAX_VALUE ? -1 : dist[target];
}
```

### Boilerplate — grid

```java
int zeroOneBFS(int[][] grid) {   // reach bottom-right; weight = cost of ENTERING a cell
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    int[][] dist = new int[m][n];
    for (int[] row : dist) Arrays.fill(row, Integer.MAX_VALUE);
    Deque<int[]> deque = new ArrayDeque<>();
    dist[0][0] = 0;
    deque.offerFirst(new int[]{0, 0});

    while (!deque.isEmpty()) {
        int[] cell = deque.pollFirst();
        int r = cell[0], c = cell[1];
        for (int[] d : DIRS) {
            int nr = r + d[0], nc = c + d[1];
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
            int w = grid[nr][nc];                 // 0 or 1 — problem-specific
            if (dist[r][c] + w < dist[nr][nc]) {
                dist[nr][nc] = dist[r][c] + w;
                if (w == 0) deque.offerFirst(new int[]{nr, nc});
                else        deque.offerLast(new int[]{nr, nc});
            }
        }
    }
    return dist[m - 1][n - 1];
}
```

**Two structural differences from your BFS boilerplate** — call these out if asked why it's not "just BFS":

1. **No visited-at-enqueue marking.** A node can be reached, then reached again more cheaply, so it's re-added (like Dijkstra's lazy deletion). The guard is the relaxation check `dist[u] + w < dist[v]`, not a `visited` flag. A node may be popped more than once; the stale pops are harmless because their `dist` is already optimal and no neighbor improves.
2. **No level-order `size` loop.** Distances live in the `dist` array, not in ring-counting, because rings no longer line up with distances one-to-one.

**Time:** O(V + E) — each edge relaxed O(1) times; the deque, unlike a heap, is O(1) per push/pop.
**Space:** O(V) — `dist` array plus the deque (which can hold up to O(V) nodes).

### Recognition signal

The problem *looks* like grid BFS, but **some moves are free and some cost 1**. The moment you see a shortest-path/min-cost question where costs are binary, reach for the deque. (Unweighted → BFS; weights ∈ {0,1} → 0-1 BFS; arbitrary non-negative weights → Dijkstra.)

### Example problem — Minimum Obstacle Removal to Reach Corner (LC 2290, Hard)

> Grid of `0` (empty) and `1` (obstacle). Start at top-left, reach bottom-right (both guaranteed empty). You may remove obstacles; return the **minimum number removed**.

**Why 0-1 BFS:** stepping into an empty cell costs `0` (no removal), stepping into an obstacle costs `1` (remove it). The grid values **are** the edge weights — `dist[nr][nc]` is literally "fewest obstacles removed to reach this cell." Binary costs → deque.

**Concrete enumeration** on a small grid (`S` = start, `T` = target):

```
S 1 0        Pop S(0,0), dist 0:
0 1 0          →(0,1) obstacle, w=1 → BACK,  dist=1
0 0 T          ↓(1,0) empty,    w=0 → FRONT, dist=0
             Pop (1,0) first (dist 0 — it jumped the queue):
               ↓(2,0) empty, w=0 → FRONT, dist=0 ...
             The entire free region floods (all dist 0) BEFORE any dist-1
             node is touched, so we snake 0→0→0 down col 0, along row 2, to T.
             answer: 0   (a fully-free path exists — never remove anything)
```

Plain BFS would have processed the obstacle at `(0,1)` in the same "ring" as the free cell `(1,0)`, potentially locking in a path that removes an obstacle unnecessarily. The deque guarantees the 0-cost frontier is exhausted first — that's the correctness the front-push buys you.

**Time:** O(M·N) **Space:** O(M·N)

### Example problem — Minimum Cost to Make at Least One Valid Path in a Grid (LC 1368, Hard)

> Each cell holds an arrow (1=right, 2=left, 3=down, 4=up). Following a cell's own arrow is free; entering any of the other three neighbors costs `1` (you "rewrite" that cell's sign). Find the min cost to reach the bottom-right from the top-left.

**Why 0-1 BFS:** from cell `(r,c)`, the move in the direction its arrow already points is a **0-edge**; the other three directions are **1-edges**. Identical structure to LC 2290 — the only change is *how you compute `w`* for each neighbor.

**Neighbor generation** (the problem-specific piece):

```java
int[][] DIRS = {{0,1},{0,-1},{1,0},{-1,0}};  // index+1 matches sign values 1..4
// for cell (r,c) with sign s (1..4):
for (int dir = 0; dir < 4; dir++) {
    int nr = r + DIRS[dir][0], nc = c + DIRS[dir][1];
    // ... bounds check ...
    int w = (dir + 1 == grid[r][c]) ? 0 : 1;   // free iff we follow the existing arrow
    // relax with front-push if w==0, back-push if w==1
}
```

**Concrete enumeration** — cell `(r,c)` with sign `1` (points right):

```
following the arrow  → (r, c+1)  costs 0   → deque FRONT
turning to go left   → (r, c-1)  costs 1   → deque BACK
turning to go down   → (r+1, c)  costs 1   → deque BACK
turning to go up     → (r-1, c)  costs 1   → deque BACK
```

So the "natural" arrow-following paths flood outward at zero cost, and you only pay a `1` each time the intended path forces a sign rewrite — exactly the minimum number of rewrites, which is the answer.

**Why not Dijkstra?** You *could* — it gives the same answer in O(V log V). But with only two distinct edge weights the deque removes the heap's `log` factor, and (equally worth saying) it's less code and no `PriorityQueue` comparator to get wrong.

**Time:** O(M·N) **Space:** O(M·N)

### Generalization (mention, don't implement)

Weights ∈ `{0, 1, ..., k}` → **Dial's algorithm**: `k+1` buckets indexed by `dist % (k+1)`, a natural extension of the two-ended deque. Beyond interview scope, but naming it signals you understand *why* the deque works — it's the k=1 case of bucket-based shortest paths.

---

## 5. Dijkstra's Algorithm

### Intuition

**One sentence:** Dijkstra is BFS generalized to **non-negative weighted** graphs — instead of expanding in rings of equal edge-count, you always expand the **unfinished node with the smallest known distance**, and a min-heap gives you that node in O(log V).

Think of it as the direct sequel to what you already have:

```
unweighted        → BFS        (queue: every edge costs 1, so FIFO = sorted by distance)
weights ∈ {0,1}   → 0-1 BFS    (deque: front for 0, back for 1 keeps it sorted, no heap)
arbitrary w ≥ 0   → Dijkstra   (min-heap: general weights need a real priority queue)
```

**Why the heap is forced.** BFS's FIFO queue stays sorted by distance only because every edge adds exactly 1 — the next-nearest node is always already at the front. With arbitrary weights that breaks: a node reached via a cheap 2-hop path can be closer than one reached via an expensive 1-hop path. You need whatever's *actually* closest next, so you need a priority queue.

**The core invariant** (this is the whole proof, in one line): the first time Dijkstra pops a node from the heap, its distance is **final and optimal**. Reason: every edge is ≥ 0, so any *other* path to that node goes through some node still in the heap whose distance is already ≥ the popped node's — it can only get worse from there. **This is exactly why negative edges break Dijkstra:** a later negative edge could undercut a distance you already "finalized," violating the invariant. (Negative edges → Bellman-Ford.)

**Concrete enumeration** — start at A, edges `A→B(4), A→C(1), C→B(2), C→D(5), B→D(1)`:

```
heap: (0,A)                              dist: A=0
pop A(0) → push (4,B),(1,C)              dist: A=0, B=4, C=1
pop C(1) → C→B: 1+2=3 < 4, update B=3    dist: B=3(was 4), D=6   push (3,B),(6,D)
          C→D: 1+5=6, D=6
pop B(3) → B→D: 3+1=4 < 6, update D=4     dist: D=4               push (4,D)
pop D(4) → done                          FINAL: A=0,B=3,C=1,D=4
          (the stale (6,D) still in heap is skipped — see "lazy deletion")
```

Notice B got **relaxed twice** (4 → 3): Dijkstra improves a tentative distance whenever a cheaper path appears, and only "locks it in" at pop time.

### Boilerplate — adjacency list

```java
int[] dijkstra(int n, List<List<int[]>> adj, int src) {   // adj.get(u) = list of {neighbor, weight}
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    // heap holds {distance, node}, ordered by distance
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, src});

    while (!pq.isEmpty()) {
        int[] top = pq.poll();
        int d = top[0], u = top[1];
        if (d > dist[u]) continue;         // LAZY DELETION: stale entry, skip
        for (int[] e : adj.get(u)) {
            int v = e[0], w = e[1];
            if (dist[u] + w < dist[v]) {   // relaxation
                dist[v] = dist[u] + w;
                pq.offer(new int[]{dist[v], v});
            }
        }
    }
    return dist;
}
```

### Boilerplate — grid

```java
int dijkstraGrid(int[][] cost) {   // cost[r][c] = weight to ENTER that cell; reach bottom-right
    int m = cost.length, n = cost[0].length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    int[][] dist = new int[m][n];
    for (int[] row : dist) Arrays.fill(row, Integer.MAX_VALUE);
    dist[0][0] = cost[0][0];
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{dist[0][0], 0, 0});   // {distance, row, col}

    while (!pq.isEmpty()) {
        int[] top = pq.poll();
        int d = top[0], r = top[1], c = top[2];
        if (d > dist[r][c]) continue;
        if (r == m - 1 && c == n - 1) return d;   // popped target ⇒ optimal, early exit
        for (int[] dir : DIRS) {
            int nr = r + dir[0], nc = c + dir[1];
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
            int nd = d + cost[nr][nc];
            if (nd < dist[nr][nc]) {
                dist[nr][nc] = nd;
                pq.offer(new int[]{nd, nr, nc});
            }
        }
    }
    return dist[m - 1][n - 1];
}
```

**Lazy deletion — the one habit that separates correct Dijkstra from buggy Dijkstra.** Java's `PriorityQueue` has no efficient decrease-key, so instead of updating an existing heap entry you just push a *new* `{smaller_dist, node}` and leave the old one behind. The stale entries are harmless as long as you skip them with the `if (d > dist[u]) continue;` guard at pop time. Forgetting that guard doesn't give a wrong answer, but it re-processes nodes and can blow the complexity up — always write it.

**Early exit:** the moment you *pop* the target, its distance is final (the invariant), so you can return immediately. Popping ≠ pushing — don't exit when you push the target, only when you pop it.

### Complexity

**Time:** O(E log V) — each edge triggers at most one heap push, and each of the ≤ E heap operations is O(log V). (With lazy deletion the heap can hold up to E entries, so some write it O(E log E); since E ≤ V², `log E ≤ 2 log V`, same class.)
**Space:** O(V + E) — adjacency list O(V + E), `dist` array O(V), heap up to O(E).

### When to use it

- **Weighted shortest path, non-negative weights.** The defining case. "Minimum cost / time / distance" where edges carry unequal, non-negative costs.
- **Single source to one or all targets.** One run gives shortest distances from the source to *every* node.
- **NOT for negative weights** → Bellman-Ford (O(VE)) or SPFA. **NOT for all-pairs on dense graphs** → Floyd-Warshall (O(V³)) is simpler to write. **NOT when all weights are equal** → plain BFS is O(V+E), skip the heap.

### How to adapt it to problem variants

Same skeleton — heap, `dist`, relaxation, lazy-deletion guard — with one piece swapped:

| You need... | Adaptation |
|---|---|
| **A constraint like "≤ K stops"** | Augment the state: `dist` is keyed by `(node, stopsUsed)`, not just `node`. Heap entries carry the extra dimension. |
| **Minimax / maximin path** (minimize the *largest* edge, or maximize the *smallest*) | Change the relaxation aggregate from **sum** to **max** (or **min**). The heap still pops the best "path cost" first; only how you combine an edge with the running cost changes. |
| **Maximize a product** (e.g. probability) | Use a **max-heap**, relax by multiplication, and negate/invert the comparator. Same greedy skeleton. |
| **Count shortest paths** | Keep a `ways[]` array; when you *find a strictly shorter* path reset `ways[v] = ways[u]`, when you find an *equal-length* one do `ways[v] += ways[u]`. |
| **Weights are only 0/1** | Don't use Dijkstra — drop to **0-1 BFS** (Section 4), O(V+E), no heap. |

The meta-skill: Dijkstra is a *template*, and most "hard" graph problems are this template with **either an augmented state or a swapped aggregation operator**. Spotting which of those two knobs the problem turns is the entire game.

### Example problem — Network Delay Time (LC 743, Medium) · *the clean one*

> `times[i] = [u, v, w]` is a directed edge with travel time `w`. A signal starts at node `k`. Return the time for **all** `n` nodes to receive it, or `-1` if some node never does.

**Why it's the perfect intro:** it's textbook Dijkstra with zero twist. "Time for all nodes to receive the signal" = the **maximum** of the shortest distances from `k` to every node. Run one Dijkstra from `k`; if any node is still `∞`, return `-1`; otherwise return the largest `dist`.

```java
public int networkDelayTime(int[][] times, int n, int k) {
    List<List<int[]>> adj = new ArrayList<>();
    for (int i = 0; i <= n; i++) adj.add(new ArrayList<>());  // nodes are 1-indexed
    for (int[] t : times) adj.get(t[0]).add(new int[]{t[1], t[2]});

    int[] dist = new int[n + 1];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[k] = 0;
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, k});

    while (!pq.isEmpty()) {
        int[] top = pq.poll();
        int d = top[0], u = top[1];
        if (d > dist[u]) continue;
        for (int[] e : adj.get(u)) {
            int v = e[0], w = e[1];
            if (d + w < dist[v]) {
                dist[v] = d + w;
                pq.offer(new int[]{dist[v], v});
            }
        }
    }

    int ans = 0;
    for (int i = 1; i <= n; i++) {
        if (dist[i] == Integer.MAX_VALUE) return -1;  // unreachable node
        ans = Math.max(ans, dist[i]);
    }
    return ans;
}
```

**Edge cases:** a node with no incoming edges (unreachable → `-1`); `k` itself always has `dist 0`; 1-indexing (`n + 1` sized arrays) is the usual slip.
**Time:** O(E log V) **Space:** O(V + E)

### Example problem — Cheapest Flights Within K Stops (LC 787, Medium) · *tricky: augmented state*

> `n` cities, `flights[i] = [from, to, price]`, find the cheapest price from `src` to `dst` using **at most `k` stops** (so at most `k+1` edges), or `-1`.

**The twist that trips people up:** plain Dijkstra minimizes price *ignoring* stop count — but here a cheaper path might use too many stops, and a pricier path might be the only legal one. So price alone isn't a sufficient state. **Augment the state to `(city, stopsUsed)`** and let the heap order by price. Crucially, you must **not** use a single `dist[]` keyed by city and skip-when-seen — the same city is legitimately worth revisiting if you reach it with *fewer stops left*.

```java
public int findCheapestPrice(int n, int[][] flights, int src, int dst, int k) {
    List<List<int[]>> adj = new ArrayList<>();
    for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
    for (int[] f : flights) adj.get(f[0]).add(new int[]{f[1], f[2]});   // to, price

    // heap entry: {costSoFar, city, stopsUsed}, ordered by cost
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, src, 0});
    // best[city] = fewest stops we've reached it with; lets us prune dominated states
    int[] minStops = new int[n];
    Arrays.fill(minStops, Integer.MAX_VALUE);

    while (!pq.isEmpty()) {
        int[] top = pq.poll();
        int cost = top[0], city = top[1], stops = top[2];
        if (city == dst) return cost;                 // popped dst with min cost ⇒ answer
        if (stops > k) continue;                       // no budget to expand further
        if (stops >= minStops[city]) continue;         // a cheaper pop already used ≤ stops here
        minStops[city] = stops;
        for (int[] e : adj.get(city)) {
            int next = e[0], price = e[1];
            pq.offer(new int[]{cost + price, next, stops + 1});
        }
    }
    return -1;
}
```

**Why popping `dst` is safe as the answer:** the heap is ordered by cost, so the first time `dst` comes off the heap it's via the cheapest cost *among all paths within the stop budget* — the ones exceeding `k` stops were pruned before they could expand. The `minStops` prune keeps the heap from exploding on graphs with many cheap-but-long detours.

> **Interview aside worth volunteering:** this problem is *also* cleanly solved by **Bellman-Ford run exactly `k+1` rounds** (O(k·E), often stated as the "intended" solution because the stop-limit maps perfectly onto Bellman-Ford's round = edge-count structure). Naming both — "Dijkstra with an augmented `(city, stops)` state, or Bellman-Ford capped at k+1 relaxation rounds" — is exactly the kind of two-solution fluency L4 interviewers reward.

**Time:** O(E·K·log(...)) in the augmented state space. **Space:** O(V + E) plus heap.

### Example problem — Swim in Rising Water (LC 778, Hard) · *tricky: minimax path*

> `grid[r][c]` is the elevation at `(r,c)`. At time `t` a cell is submerged-passable iff `elevation ≤ t`; swimming between adjacent cells is instant. Return the **least time** to get from top-left to bottom-right.

**Reframe (this is the whole insight):** the time to finish a given path is the **maximum elevation** along that path (you must wait for the highest cell on your route to be reachable). You want the path whose maximum elevation is **smallest** — a classic **minimax path**. That's Dijkstra with **one operator swapped**: instead of `dist = sum of weights`, `dist = max elevation on the path`, and you relax with `max` instead of `+`.

```java
public int swimInWater(int[][] grid) {
    int n = grid.length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    int[][] time = new int[n][n];                 // time[r][c] = min possible "max elevation" to reach
    for (int[] row : time) Arrays.fill(row, Integer.MAX_VALUE);
    time[0][0] = grid[0][0];
    // heap ordered by the running max-elevation cost
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{grid[0][0], 0, 0});         // {maxElevSoFar, row, col}

    while (!pq.isEmpty()) {
        int[] top = pq.poll();
        int t = top[0], r = top[1], c = top[2];
        if (t > time[r][c]) continue;              // lazy deletion
        if (r == n - 1 && c == n - 1) return t;    // popped target ⇒ optimal
        for (int[] d : DIRS) {
            int nr = r + d[0], nc = c + d[1];
            if (nr < 0 || nr >= n || nc < 0 || nc >= n) continue;
            int nt = Math.max(t, grid[nr][nc]);    // ← MAX, not sum: the only real change
            if (nt < time[nr][nc]) {
                time[nr][nc] = nt;
                pq.offer(new int[]{nt, nr, nc});
            }
        }
    }
    return time[n - 1][n - 1];
}
```

**Why the Dijkstra invariant still holds with `max`:** elevations are non-negative and `max` is monotonic — extending a path can only *raise or keep* its running max, never lower it, so the first pop of a cell is still its optimal (smallest achievable) max-elevation. Same proof shape as the sum version; that's why the template survives the operator swap.

> **Two other classic approaches to name** (shows range): **binary search on `t`** + BFS/DFS reachability check — "is there a path using only cells ≤ t?" — O(n²·log(max)); or **Union-Find**, adding cells in elevation order until start and end connect. The Dijkstra/heap version is usually the cleanest to code live, but volunteering the binary-search framing signals you saw the monotonic structure.

**Time:** O(n²·log n) — n² cells, each heap op O(log n²)=O(log n). **Space:** O(n²).

### Tips & tricks

- **Always write the lazy-deletion guard** `if (d > dist[u]) continue;`. It's the single most common Dijkstra bug when there's no decrease-key.
- **Early-exit on *pop*, never on push.** The invariant only fires when a node leaves the heap.
- **Augmented state vs swapped operator** — nearly every hard Dijkstra variant is one of these two. Ask "is there a side constraint?" (→ augment the state tuple) and "is the path cost a sum, or a max/min/product?" (→ swap the relaxation op).
- **Negative weight anywhere ⇒ not Dijkstra.** Say "Bellman-Ford" out loud so the interviewer knows you know the boundary. All-equal weights ⇒ drop to BFS; weights ∈ {0,1} ⇒ 0-1 BFS.
- **1-indexed nodes** (Network Delay Time) → size arrays `n + 1` and start loops at 1. Silent off-by-one otherwise.
- **`int` overflow** on summed costs with large weights → use `long` for the running distance when constraints are big.

---

## 6. DFS

### Intuition

**One sentence:** DFS commits to one path all the way to a dead end, then backtracks to the most recent choice point and tries the next option — it explores by *depth*, trading BFS's distance guarantees for a natural fit with recursion, exhaustive exploration, and structure discovery.

DFS answers "**can** I reach it / **what's connected** / **enumerate everything**" — never "what's the *shortest* way" on unweighted graphs. If you catch yourself writing DFS for a shortest-path question, stop and switch to BFS.

The recursion *is* the stack: the call stack at any moment holds exactly the current root-to-here path. That's why DFS is the natural engine for backtracking, cycle detection (a node on the current stack = back edge = cycle), and path-dependent state.

### Boilerplate — adjacency list (recursive)

```java
void dfs(List<List<Integer>> adj, int node, boolean[] visited) {
    visited[node] = true;
    // --- process node here (pre-order position) ---
    for (int next : adj.get(node)) {
        if (!visited[next]) dfs(adj, next, visited);
    }
    // --- post-order position: all descendants done (used by topo sort) ---
}

// Driver — also how you count connected components:
int components = 0;
for (int i = 0; i < n; i++) {
    if (!visited[i]) { components++; dfs(adj, i, visited); }
}
```

### Boilerplate — adjacency list (iterative, explicit stack)

```java
void dfsIterative(List<List<Integer>> adj, int start, boolean[] visited) {
    Deque<Integer> stack = new ArrayDeque<>();
    stack.push(start);
    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (visited[node]) continue;   // may be pushed multiple times — check at pop
        visited[node] = true;
        for (int next : adj.get(node)) {
            if (!visited[next]) stack.push(next);
        }
    }
}
```

Know this cold: interviewers sometimes ask "what if the graph is deep enough to overflow the call stack?" — Java's default stack (~512KB–1MB) dies around 10k–100k frames, and a path graph of 10^5 nodes will overflow the recursive version.

### Boilerplate — grid (recursive, Number-of-Islands shape)

```java
void dfs(char[][] grid, int r, int c) {
    if (r < 0 || r >= grid.length || c < 0 || c >= grid[0].length) return;
    if (grid[r][c] != '1') return;     // water or already visited
    grid[r][c] = '0';                  // sink it = mark visited in place
    dfs(grid, r + 1, c);
    dfs(grid, r - 1, c);
    dfs(grid, r, c + 1);
    dfs(grid, r, c - 1);
}
```

Guards at the *top* of the call (rather than before each recursive call) keeps the code short and symmetric — a style interviewers read easily.

**Time:** O(V + E) adjacency list; O(M·N) grid. Same as BFS — both visit every node and edge once.
**Space:** O(V) worst case. For recursion it's the call-stack depth: O(H) where H = longest path — which is O(M·N) on a snake-shaped grid, not O(M+N). Iterative: O(V) explicit stack.

### Multi-source / bidirectional — do they exist for DFS?

- **"Multi-source DFS"** exists but means something different: it's just *running DFS from every unvisited node in a driver loop* (the connected-components driver above). There's no simultaneous expansion because DFS has no notion of rings — sources are processed one after another, not in parallel. The parallel-spread problems (Rotten Oranges) are inherently BFS.
- A genuinely useful two-sided DFS pattern: **Pacific Atlantic Water Flow (LC 417)** — instead of asking "from each cell, can water reach both oceans?" (O((M·N)²)), flip the direction: DFS *inward from each ocean's border*, marking reachable cells; the answer is the intersection of the two marked sets. The trick isn't bidirectional search meeting in the middle — it's **reversing the edge direction to turn many queries into two sweeps**. That reversal instinct transfers to lots of problems.
- **Bidirectional search is a BFS-only trick.** Two DFS probes racing down arbitrary deep paths have essentially zero chance of colliding — the meet-in-the-middle math relies on both sides expanding *complete distance rings*.

### When to use DFS

- **Connectivity & components** — islands, provinces, flood fill. (BFS works too; DFS is less code.)
- **Exhaustive enumeration / backtracking** — all paths, permutations, combinations; the call stack carries the current path for free.
- **Cycle detection** — directed: 3-color / "on current stack" check. Undirected: visited neighbor that isn't the parent.
- **Topological sort** — post-order DFS reversed (or Kahn's BFS; know both).
- **Structure discovery** — bridges, articulation points, SCCs (Tarjan). Low-level DFS tree properties; BFS can't do these.
- **DFS + memoization on a DAG** — this is just top-down DP wearing a graph costume. Your DP instincts transfer directly (see LC 329 below).

### Example problem — Number of Islands (LC 200, Medium)

> Grid of '1' (land) and '0' (water). Count islands (4-directionally connected land groups).

**One sentence:** each island is one connected component; scan every cell, and each time you find unvisited land, increment the count and sink the entire component with DFS so it's never counted again.

**Concrete enumeration:**

```
1 1 0        scan (0,0): land → count=1, DFS sinks (0,0),(0,1),(1,0),(1,1)
1 1 0        scan (0,1)..(1,1): already '0', skip
0 0 1        scan (2,2): land → count=2, DFS sinks (2,2)
             answer: 2
```

**Time:** O(M·N) — each cell visited a constant number of times.
**Space:** O(M·N) recursion depth worst case (all-land grid traversed as a snake). Mention this unprompted; it's a favorite follow-up. The BFS variant makes it O(min(M,N)) queue space.

**Follow-ups Google likes:** count *distinct* island shapes (LC 694 — serialize the DFS path), Number of Islands II with additions over time (LC 305 — Union-Find, not DFS).

### Example problem — Longest Increasing Path in a Matrix (LC 329, Hard)

> Find the length of the longest strictly increasing path in a matrix (4-directional moves).

**Why this one:** it's the perfect bridge from your DP work — **DFS + memoization on an implicit DAG**. Draw an edge from cell `a` to neighbor `b` whenever `matrix[b] > matrix[a]`; strict increase means no cycles, so the grid becomes a DAG and "longest path" (NP-hard in general graphs!) becomes tractable.

**State definition (one English sentence):** `memo[r][c]` = the length of the longest increasing path *starting at* cell `(r, c)`.

**Recurrence:** `memo[r][c] = 1 + max(memo[nr][nc])` over increasing neighbors, or `1` if none.

**Concrete enumeration** for the center of

```
9 9 4
6 6 8        solve(1,1) [value 6]: up=9 ✓ → 1+solve(0,1);  right=8 ✓ → 1+solve(1,2)
2 1 1                       down=1 ✗;  left=6 ✗ (not strictly greater)
             solve(0,1)=1 (9 has no greater neighbor); solve(1,2): 8→9 → 2
             memo[1][1] = 1 + max(1, 2) = 3        (path 6→8→9)
             global answer: 4                       (path 1→2→6→9)
```

**No visited array needed** — strictness prevents revisiting (you can never come back to an equal-or-smaller cell), and the memo prevents recomputation. If an interviewer asks why, that's the answer: *the DAG property replaces the visited set*.

**Type / knobs:** DP on an implicit DAG. State = a single cell `(r,c)`. Transition = `max` over ≤ 4 strictly-larger neighbors. **Topological order = cell value** (a cell depends only on larger-valued cells), which is what makes the recursion acyclic and the tabulation possible.

Following your DP-notes format — recurrence → memo → tabulation → space-opt — with the rungs paired.

#### Rung 1 & 2 — Recurrence next to Memo

The memo *is* the recurrence plus three cache lines (check / compute / store). Since a cell depends only on **larger** neighbors, no visited set and no explicit ordering are needed — recursion discovers the topological order for free.

```java
// RECURRENCE (math)                          |  // MEMO (top-down DFS — recurrence + cache)
//                                            |
// f(r,c) = length of longest strictly        |  Integer[][] memo;                 // null = uncomputed
//          increasing path STARTING at (r,c)  |  int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
//                                            |
// f(r,c) = 1 + max{ f(nr,nc) :               |  int longestIncreasingPath(int[][] matrix) {
//            (nr,nc) a 4-neighbor,           |      int m = matrix.length, n = matrix[0].length;
//            matrix[nr][nc] > matrix[r][c] }  |      memo = new Integer[m][n];
//        = 1  if no larger neighbor           |      int best = 0;
//                                            |      for (int r = 0; r < m; r++)
// answer = max over all cells of f(r,c)       |          for (int c = 0; c < n; c++)
//                                            |              best = Math.max(best, dfs(matrix, r, c));
//                                            |      return best;
//                                            |  }
//                                            |
//                                            |  int dfs(int[][] mat, int r, int c) {
//                                            |      if (memo[r][c] != null) return memo[r][c];  // cache hit
//                                            |      int m = mat.length, n = mat[0].length, best = 1;
//                                            |      for (int[] d : DIRS) {
//                                            |          int nr = r + d[0], nc = c + d[1];
//                                            |          if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
//                                            |          if (mat[nr][nc] <= mat[r][c]) continue;   // must strictly increase
//                                            |          best = Math.max(best, 1 + dfs(mat, nr, nc));
//                                            |      }
//                                            |      return memo[r][c] = best;                     // store
//                                            |  }
```

#### Rung 3 — Tabulation (explicit topological order = sort cells by value)

To remove recursion you must evaluate cells in an order where every larger neighbor is already done. That order is simply **cells sorted by value, descending** — the DAG's topological order made explicit. Then apply the identical recurrence iteratively.

```java
int longestIncreasingPath(int[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};

    // All cells as (value, r, c), sorted by value DESCENDING
    int[][] cells = new int[m * n][3];
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            cells[r * n + c] = new int[]{matrix[r][c], r, c};
    Arrays.sort(cells, (a, b) -> b[0] - a[0]);   // largest first

    int[][] dp = new int[m][n];                  // dp[r][c] = f(r,c); every cell is ≥ 1
    int best = 0;
    for (int[] cell : cells) {                    // process high → low
        int r = cell[1], c = cell[2];
        dp[r][c] = 1;                             // path of just this cell
        for (int[] d : DIRS) {
            int nr = r + d[0], nc = c + d[1];
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
            if (matrix[nr][nc] <= matrix[r][c]) continue;   // larger neighbor
            dp[r][c] = Math.max(dp[r][c], 1 + dp[nr][nc]);  // already computed (higher value)
        }
        best = Math.max(best, dp[r][c]);
    }
    return best;
}
```

The mapping from memo to tabulation is exact: the recursion visited cells in *some* topological order lazily; here we impose that order eagerly via the sort. Same recurrence, same `dp` values, no call stack.

#### Rung 4 — Space optimization: **not available here** (and knowing *why* is the point)

In grid/sequence DP you collapse `dp[i][j]` to one row because row `i` depends only on row `i-1` — a clean, bounded frontier. **That collapse fails here.** A cell can depend on *any* of its four neighbors, including the one directly **below** it (next row) or **beside** it (same row, either direction). An increasing path can snake up, down, left, and right arbitrarily, so the set of cells a given cell depends on isn't confined to an adjacent row or column. There's no rolling frontier to keep — you genuinely need the full `O(M·N)` table.

This is a useful negative example for your "when does space-opt apply?" instinct: **row-collapse works only when dependencies point to a bounded, fixed set of previous rows/columns.** Grid-path DP (depends on up + left) collapses; LIS-in-matrix (depends on arbitrary-direction neighbors) does not.

#### Trick / clever variant — Kahn's peeling (drops the sort, ties to Section 7)

The tabulation's `O(M·N log(M·N))` sort can be removed. This is a **longest-path-on-a-DAG** problem, and Section 7 solves those by topological *layering*: give each cell an out-degree = number of strictly-larger neighbors, seed a queue with the **local maxima** (out-degree 0), and peel layer by layer. The number of layers peeled *is* the longest increasing path length — because each layer strips off the current "tops" of all paths.

```java
int longestIncreasingPath(int[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    int[][] outdeg = new int[m][n];              // # of strictly-larger neighbors
    Queue<int[]> queue = new ArrayDeque<>();

    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++) {
            for (int[] d : DIRS) {
                int nr = r + d[0], nc = c + d[1];
                if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
                if (matrix[nr][nc] > matrix[r][c]) outdeg[r][c]++;
            }
            if (outdeg[r][c] == 0) queue.offer(new int[]{r, c});   // local maxima = path ends
        }

    int layers = 0;
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] cell = queue.poll();
            int r = cell[0], c = cell[1];
            for (int[] d : DIRS) {               // walk to SMALLER neighbors (reverse edges)
                int nr = r + d[0], nc = c + d[1];
                if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
                if (matrix[nr][nc] >= matrix[r][c]) continue;      // only strictly smaller
                if (--outdeg[nr][nc] == 0) queue.offer(new int[]{nr, nc});
            }
        }
        layers++;                                 // one full ring peeled = one more path-length
    }
    return layers;
}
```

This is O(M·N) (no sort) and reuses the exact Kahn's-peeling machinery from the topological-sort section — a nice demonstration that "longest path on a DAG" and "topological sort" are the same tool wearing different hats.

**Java note:** `Integer[][] memo` with `null` sentinel (your standard convention) suits the memo rung; `int[][]` with a `0` sentinel also works since every valid answer is ≥ 1.

**Time:**
- Recurrence / Memo: **O(M·N)** — each cell computed once, 4 edges each.
- Tabulation (sort): **O(M·N · log(M·N))** — dominated by the sort.
- Peeling variant: **O(M·N)** — no sort.

**Space:**
- Memo: **O(M·N)** table + **O(M·N)** recursion stack worst case (snake-shaped increasing path).
- Tabulation: **O(M·N)** for `dp` + the cell array; no recursion.
- Peeling: **O(M·N)** for `outdeg` + queue.
- Space-optimized rung: **not applicable** — dependencies aren't row-bounded, so the full table is required.

---

## 7. Topological Sort

### Intuition

**One sentence:** a topological sort of a **directed acyclic graph** is a linear ordering of its nodes such that for every edge `u → v`, `u` comes before `v` — i.e., "everything a node depends on is already placed before it."

Two mental models, both worth holding:

- **Kahn's (BFS / peel-off):** repeatedly remove a node with **no remaining prerequisites** (indegree 0), append it to the order, and delete its outgoing edges — which may free up new indegree-0 nodes. You're peeling the graph from its "roots" inward. If you ever get stuck with nodes left but none at indegree 0, those nodes form a **cycle** — and a cycle means *no* valid ordering exists.
- **DFS (post-order / finish-time):** a node isn't truly "done" until all its descendants are done. So if you push a node onto a stack *at the moment its DFS finishes*, the stack — read top to bottom — is a valid topological order (reverse post-order).

**Concrete enumeration** (Kahn's on courses, edge = prereq → course):

```
Edges: 0→1, 0→2, 1→3, 2→3       indegree: [0, 1, 1, 2]

peel 0 (indeg 0) → order=[0];   drop 0→1, 0→2  → indegree [_, 0, 0, 2]
peel 1           → order=[0,1]; drop 1→3        → indegree [_, _, 0, 1]
peel 2           → order=[0,1,2]; drop 2→3      → indegree [_, _, _, 0]
peel 3           → order=[0,1,2,3]
```

The key property this reveals: at the step where both `1` and `2` sit at indegree 0, **either could go next** — that ambiguity is exactly what "multiple valid orderings" means, and detecting it (queue size > 1) is how you check for a *unique* order.

### Boilerplate — Kahn's algorithm (BFS, indegree peeling)

```java
// Returns a topo order, or an EMPTY list if the graph has a cycle.
List<Integer> topoSort(int n, List<List<Integer>> adj) {   // adj.get(u) = nodes u points to
    int[] indegree = new int[n];
    for (int u = 0; u < n; u++)
        for (int v : adj.get(u)) indegree[v]++;

    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < n; i++)
        if (indegree[i] == 0) queue.offer(i);       // seed with all roots

    List<Integer> order = new ArrayList<>();
    while (!queue.isEmpty()) {
        int u = queue.poll();
        order.add(u);
        for (int v : adj.get(u))
            if (--indegree[v] == 0) queue.offer(v);  // v just became a root
    }
    return order.size() == n ? order : new ArrayList<>();   // short list ⇒ cycle
}
```

### Boilerplate — DFS post-order (3-color cycle detection)

```java
// state: 0 = unvisited, 1 = on current recursion stack, 2 = fully finished
int[] state;
Deque<Integer> stack;   // collects nodes in reverse post-order

List<Integer> topoSortDFS(int n, List<List<Integer>> adj) {
    state = new int[n];
    stack = new ArrayDeque<>();
    for (int i = 0; i < n; i++)
        if (state[i] == 0 && !dfs(adj, i)) return new ArrayList<>();  // cycle ⇒ empty
    return new ArrayList<>(stack);   // stack top→bottom is the topo order
}

boolean dfs(List<List<Integer>> adj, int u) {
    state[u] = 1;                        // enter: mark "on stack"
    for (int v : adj.get(u)) {
        if (state[v] == 1) return false; // back edge to an ancestor ⇒ CYCLE
        if (state[v] == 0 && !dfs(adj, v)) return false;
    }
    state[u] = 2;                        // leave: fully finished
    stack.push(u);                       // push AFTER all descendants ⇒ reverse post-order
    return true;
}
```

Why 3 colors and not a plain `boolean visited`? A directed graph can revisit an *already-finished* node legitimately (two paths into it) — that is **not** a cycle. Only an edge back to a node still **on the current stack** (state 1) is a cycle. A 2-state visited array can't tell those apart. (Undirected graphs are different — there, "visited neighbor that isn't the parent" is the cycle test.)

### Complexity

**Time:** O(V + E) — every node enqueued/visited once, every edge relaxed once. Identical for Kahn's and DFS.
**Space:** O(V + E) — adjacency list O(V + E), plus indegree array + queue (Kahn's) or state array + recursion/explicit stack (DFS), each O(V).

### When to use it

Reach for topological sort whenever the problem is **"order things subject to precedence/dependency constraints"** or **"is this dependency graph acyclic?"** Concretely:

- Task/course/build scheduling — "A must finish before B."
- Compilation order, spreadsheet cell recomputation, package/dependency resolution.
- Detecting a cycle in a *directed* graph (Kahn's: `order.size() < n`; DFS: a back edge).
- Deriving a hidden global order from local constraints (Alien Dictionary, Sequence Reconstruction).
- As a preprocessing step for **DP on a DAG** — process nodes in topo order so every dependency's answer is ready (longest path, counting paths).

If the graph might have a cycle, topo sort *is* your cycle detector — you get it for free.

### How to adapt it to problem variants

This is where most interview mileage is. The core loop barely changes; you bolt on one check:

| You need... | Adaptation |
|---|---|
| **Cycle detection** | Kahn's: if `order.size() != n`, the leftover nodes are a cycle. DFS: a back edge (state == 1) during traversal. |
| **A *unique* ordering** | Run Kahn's; if the queue ever holds **> 1** node, more than one valid order exists → not unique. (This is the whole trick behind Sequence Reconstruction.) |
| **Lexicographically smallest order** | Swap the `Queue` for a `PriorityQueue<Integer>` (min-heap). Cost rises to O(V log V + E). Common Alien-Dictionary follow-up. |
| **Longest path / counting paths on a DAG** | Process nodes in topo order; run a DP relaxation as you pop each node. Topo order guarantees dependencies are finalized first. |
| **The graph is implicit** | The real work is *building* it: Alien Dictionary derives edges from adjacent word pairs; Sequence Reconstruction from consecutive elements. The sort itself is boilerplate. |
| **Isolated nodes with no constraints** | Make sure they're in the indegree map with indegree 0 so they still get emitted — a classic Alien Dictionary bug (letters that appear but have no ordering edge). |

### Example problems

#### Course Schedule (LC 207, Medium) — *can* it be done?

> `numCourses` courses labeled `0..numCourses-1`; `prerequisites[i] = [a, b]` means you must take `b` before `a`. Return whether you can finish all courses.

**Framing:** edge `b → a` ("prereq points to the course it unlocks"). "Can finish all" ⇔ the dependency graph is a **DAG** ⇔ Kahn's peels every node. Watch the edge direction: `[a, b]` means `b → a`, not `a → b`.

```java
public boolean canFinish(int numCourses, int[][] prerequisites) {
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) adj.add(new ArrayList<>());
    int[] indegree = new int[numCourses];
    for (int[] p : prerequisites) {   // p = [course, prereq]
        adj.get(p[1]).add(p[0]);      // prereq → course
        indegree[p[0]]++;
    }

    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < numCourses; i++)
        if (indegree[i] == 0) queue.offer(i);

    int taken = 0;
    while (!queue.isEmpty()) {
        int c = queue.poll();
        taken++;
        for (int next : adj.get(c))
            if (--indegree[next] == 0) queue.offer(next);
    }
    return taken == numCourses;   // all taken ⇔ no cycle
}
```

**Course Schedule II (LC 210)** just asks for *an* order — return the `order` list instead of a count, and return an empty array if `order.size() != numCourses`:

```java
public int[] findOrder(int numCourses, int[][] prerequisites) {
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) adj.add(new ArrayList<>());
    int[] indegree = new int[numCourses];
    for (int[] p : prerequisites) {
        adj.get(p[1]).add(p[0]);
        indegree[p[0]]++;
    }

    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < numCourses; i++)
        if (indegree[i] == 0) queue.offer(i);

    int[] order = new int[numCourses];
    int idx = 0;
    while (!queue.isEmpty()) {
        int c = queue.poll();
        order[idx++] = c;
        for (int next : adj.get(c))
            if (--indegree[next] == 0) queue.offer(next);
    }
    return idx == numCourses ? order : new int[0];   // cycle ⇒ empty
}
```

**Time:** O(V + E) where V = numCourses, E = prerequisites.length. **Space:** O(V + E).

#### Sequence Reconstruction (LC 444, Medium) — is the order *unique*?

> `nums` is a permutation of `1..n`. `sequences` is a list of subsequences of `nums`. Return `true` iff `nums` is the **only** shortest supersequence of all the `sequences` — i.e., the constraints force exactly one topological order, and it equals `nums`.

**Framing:** each `sequences[i]` imposes edges between **consecutive** elements (`seq[j] → seq[j+1]`). The order is *unique* iff, at every step of Kahn's, the queue holds **exactly one** node (no branching choice) — and that emitted order must match `nums`. This is the "unique ordering" adaptation from the table, made concrete.

```java
public boolean sequenceReconstruction(int[] nums, List<List<Integer>> sequences) {
    int n = nums.length;
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i <= n; i++) adj.add(new ArrayList<>());
    int[] indegree = new int[n + 1];
    Arrays.fill(indegree, -1);        // -1 = this value never appeared in any sequence

    for (List<Integer> seq : sequences) {
        for (int i = 0; i < seq.size(); i++) {
            int cur = seq.get(i);
            if (cur < 1 || cur > n) return false;      // value out of range ⇒ invalid
            if (indegree[cur] == -1) indegree[cur] = 0; // mark as "seen"
            if (i > 0) {
                adj.get(seq.get(i - 1)).add(cur);       // prev → cur
                indegree[cur]++;
            }
        }
    }

    Queue<Integer> queue = new ArrayDeque<>();
    for (int v = 1; v <= n; v++)
        if (indegree[v] == 0) queue.offer(v);

    int idx = 0;
    while (!queue.isEmpty()) {
        if (queue.size() > 1) return false;            // >1 choice ⇒ not unique
        int node = queue.poll();
        if (node != nums[idx++]) return false;         // order must equal nums
        for (int next : adj.get(node))
            if (--indegree[next] == 0) queue.offer(next);
    }
    return idx == n;   // every value placed ⇒ no cycle, full coverage
}
```

**The two failure modes this guards against:** `queue.size() > 1` means the constraints don't pin down the next element (multiple supersequences), and `idx != n` at the end means a cycle or a value that never appeared (so `nums` isn't even reconstructible). Both make the answer `false`.

**Time:** O(V + E), V = n, E = total length of all sequences. **Space:** O(V + E).

#### Alien Dictionary (LC 269, Hard) — recover the order from evidence

> Given `words` sorted lexicographically in an unknown alien alphabet, return any valid ordering of the distinct letters (as a string), or `""` if no valid order exists.

**Framing — the hard part is building the graph:** compare each **adjacent pair** of words; the **first** position where they differ gives one ordering edge (`firstDiffChar(a) → firstDiffChar(b)`). Then topo-sort the letters. Two traps: (1) the invalid **prefix case** — if `a` is longer than `b` but `b` is a prefix of `a` (e.g. `["abc","ab"]`), that's impossible in a real dictionary → return `""`; (2) **every letter that appears** must be in the graph, even letters with no ordering constraint, or you'll drop them from the output.

```java
public String alienOrder(String[] words) {
    Map<Character, List<Character>> adj = new HashMap<>();
    Map<Character, Integer> indegree = new HashMap<>();
    for (String w : words)                      // register EVERY letter first
        for (char c : w.toCharArray()) {
            adj.putIfAbsent(c, new ArrayList<>());
            indegree.putIfAbsent(c, 0);
        }

    for (int i = 0; i + 1 < words.length; i++) {
        String a = words[i], b = words[i + 1];
        if (a.length() > b.length() && a.startsWith(b)) return "";  // prefix trap
        int len = Math.min(a.length(), b.length());
        for (int j = 0; j < len; j++) {
            char ca = a.charAt(j), cb = b.charAt(j);
            if (ca != cb) {
                adj.get(ca).add(cb);
                indegree.put(cb, indegree.get(cb) + 1);
                break;                          // ONLY the first difference is an edge
            }
        }
    }

    Queue<Character> queue = new ArrayDeque<>();
    for (char c : indegree.keySet())
        if (indegree.get(c) == 0) queue.offer(c);

    StringBuilder sb = new StringBuilder();
    while (!queue.isEmpty()) {
        char c = queue.poll();
        sb.append(c);
        for (char next : adj.get(c))
            if (indegree.merge(next, -1, Integer::sum) == 0) queue.offer(next);
    }
    return sb.length() == indegree.size() ? sb.toString() : "";   // leftover ⇒ cycle ⇒ ""
}
```

**Why `break` after the first difference:** dictionary order only tells you about the *first* distinguishing character. From `"abc"` before `"abd"` you learn `c → d` and **nothing** about later positions — adding edges past the first difference would invent constraints that aren't implied.

**Lexicographically-smallest follow-up:** swap `ArrayDeque` for `PriorityQueue<Character>` so ties break alphabetically → O(V log V + E).

**Time:** O(C) where C = total length of all words (building edges dominates). **Space:** O(1) unique letters (≤ 26) → effectively O(V + E).

### Tips & tricks

- **Default to Kahn's in an interview.** It's iterative (no stack-overflow risk on deep graphs), and cycle detection falls out for free by comparing `order.size()` to `n`. Reach for DFS post-order only when you specifically want finish-time structure.
- **Nail the edge direction before coding.** "A before B" / "A depends on B" flip the arrow — say the direction out loud and write one example edge. Course Schedule's `[a, b]` = `b → a` is the classic slip.
- **Queue size > 1 ⇔ ambiguous order.** Memorize this equivalence; it's the reusable core of every "is the ordering unique / is the reconstruction unique" question.
- **PriorityQueue ⇒ lexicographically smallest** topo order. One-line change, frequent follow-up.
- **Register all nodes before adding edges.** Isolated nodes (indegree 0, no edges) must still be seeded into the queue, or they silently vanish from the output — the sneakiest Alien Dictionary bug.
- **Topo sort turns a DAG into a DP tape.** Once you have the order, "longest path," "number of paths," and similar become one-pass DP — the same top-down instincts from your DP work, just sequenced by dependency instead of by index.

---

## 8. Union-Find (Disjoint Set Union)

### Intuition

**One sentence:** Union-Find maintains a collection of disjoint sets under two operations — `union(a, b)` merges the sets containing `a` and `b`, and `find(x)` returns a canonical representative of `x`'s set — so "are `a` and `b` connected?" becomes `find(a) == find(b)` in near-constant time.

The mental model: each set is a **tree**, and the root is the set's ID. You don't care about the tree's shape — only "which root do I end up at." That indifference is what licenses the two optimizations:

- **Path compression** — on the way up during `find`, re-point nodes directly at the root. Since only the root's identity matters, flattening is free and permanent.
- **Union by size/rank** — always hang the smaller tree under the larger. Keeps depth logarithmic even before compression.

Together they give **O(α(n))** amortized per operation, where α is the inverse Ackermann function — ≤ 4 for any `n` you'll ever see. Treat it as constant, but *say* "amortized near-constant, α(n)" out loud.

**The key trade-off vs BFS/DFS:** traversal answers connectivity on a graph you already have, in one shot. Union-Find handles **incremental** connectivity — edges arriving one at a time, with queries interleaved — which traversal can't do without re-running from scratch. The flip side: DSU only ever *merges*. It cannot delete an edge or split a set. Edges-being-removed problems get solved by **processing the operations in reverse** so deletions become additions.

### Boilerplate

```java
class UnionFind {
    private int[] parent, size;
    private int count;                        // number of disjoint sets

    UnionFind(int n) {
        parent = new int[n];
        size = new int[n];
        count = n;
        for (int i = 0; i < n; i++) { parent[i] = i; size[i] = 1; }
    }

    // iterative find with path halving — no recursion, no stack-overflow risk
    int find(int x) {
        while (parent[x] != x) {
            parent[x] = parent[parent[x]];    // point x at its grandparent
            x = parent[x];
        }
        return x;
    }

    // returns false if a and b were ALREADY connected (i.e. this edge is redundant)
    boolean union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return false;
        if (size[ra] < size[rb]) { int t = ra; ra = rb; rb = t; }   // attach smaller under larger
        parent[rb] = ra;
        size[ra] += size[rb];
        count--;
        return true;
    }

    boolean connected(int a, int b) { return find(a) == find(b); }
    int componentCount() { return count; }
    int componentSize(int x) { return size[find(x)]; }
}
```

**The `union` return value is the workhorse.** `false` means "already connected," which directly answers *cycle detection* (an edge joining two nodes in the same set closes a cycle) and *redundant edge* problems. Don't throw it away.

**Non-integer nodes** (strings, coordinates): map to ints first — `r * n + c` for grid cells, or a `HashMap<String, Integer>` for labels. Alternatively use `Map<String, String> parent`, but the array version is faster and less error-prone.

### Complexity

**Time:** O(α(n)) amortized per `find`/`union` with both optimizations — effectively O(1). Building over E edges: **O(E · α(n))**, i.e. near-linear. Without union-by-size, worst case degrades to O(log n) per op; with neither optimization, O(n).
**Space:** O(n) for the `parent` and `size` arrays.

### When to use it

- **Incremental connectivity** — edges added over time with queries interleaved. This is DSU's unique niche; BFS/DFS would need a full re-run per query.
- **Many connectivity queries after a fixed edge set** — preprocess once, answer each query in O(α(n)).
- **Counting connected components** — start `count = n`, decrement on each successful union.
- **Cycle detection in an *undirected* graph** — a union that returns `false` means the edge closes a cycle. (For *directed* graphs use DFS 3-color or Kahn's — Section 7. DSU does not handle directed cycles.)
- **Kruskal's MST** — sort edges by weight, union greedily, skip edges whose union returns `false`.
- **Grid component problems** as an alternative to flood-fill DFS — especially "islands as land is added over time" (LC 305), which flood-fill can't do efficiently.

**When *not* to reach for it:** you need shortest path or distances (DSU knows *whether* connected, never *how far*), the graph is static and one traversal suffices (DFS is less code), or edges get deleted (unless you can reverse time).

### Example problem — Path Existence Queries in a Graph I (LC 3532, Medium)

> `n` nodes; `nums` is sorted non-decreasing; an undirected edge exists between **any** `i, j` with `|nums[i] - nums[j]| <= maxDiff`. For each query `[u, v]`, is there a path?

**Why it's the ideal opener:** it makes the case *for* and *against* DSU in the same problem.

**Why the obvious approach fails:** `n` and query count are both up to 10⁵. The edge set is O(n²) (a constant `nums` array is a complete graph), so you can't build the graph; and BFS-per-query is O(q·(V+E)). You need O(1) per query after preprocessing — which forces you to find structure rather than a faster traversal.

**The key insight — the word `sorted` is load-bearing.** Telescope any `i < j`:

```
nums[j] - nums[i] = (nums[i+1]-nums[i]) + (nums[i+2]-nums[i+1]) + ... + (nums[j]-nums[j-1])
```

Every term is ≥ 0. Two consequences:

1. **Non-adjacent edges are redundant.** If `nums[j] - nums[i] <= maxDiff`, then every adjacent gap in between is also `<= maxDiff` (each is a non-negative piece of a sum that already fits), so `i` and `j` are *already* connected by walking `i → i+1 → ... → j`. Every long edge duplicates a path through consecutive nodes.
2. **An oversized gap is an impassable wall.** If `nums[k+1] - nums[k] > maxDiff`, then no node at index ≤ `k` reaches any node at index ≥ `k+1` — for any such pair the telescoped sum contains that oversized term plus non-negative others.

So the O(n²) edge set has a **transitive closure generated by just the `n-1` adjacent edges**, and the components are **contiguous index intervals** carved out by the walls:

```
nums    = [2,   5,   6,   8]     maxDiff = 2
gaps    =    3    1    2
              ↑ WALL (3 > 2)
comp    = [0,   1,   1,   1]

query [1,3] → comp[1]==comp[3] → true      query [0,2] → 0 != 1 → false
```

**The DSU solution** — union each `i` with `i+1` when the gap fits, then answer with `connected`:

```java
public boolean[] pathExistenceQueries(int n, int[] nums, int maxDiff, int[][] queries) {
    UnionFind uf = new UnionFind(n);
    for (int i = 1; i < n; i++)
        if (nums[i] - nums[i - 1] <= maxDiff) uf.union(i - 1, i);   // adjacent edges only

    boolean[] ans = new boolean[queries.length];
    for (int i = 0; i < queries.length; i++)
        ans[i] = uf.connected(queries[i][0], queries[i][1]);
    return ans;
}
```

**...but contiguity lets you drop DSU entirely.** Because merges only ever happen between *neighbors*, the components can never be anything but intervals — so a running counter labels every node in one prefix scan:

```java
public boolean[] pathExistenceQueries(int n, int[] nums, int maxDiff, int[][] queries) {
    int[] comp = new int[n];                     // comp[i] = component id of node i
    for (int i = 1; i < n; i++)                  // wall → start a new component
        comp[i] = comp[i - 1] + (nums[i] - nums[i - 1] <= maxDiff ? 0 : 1);

    boolean[] ans = new boolean[queries.length];
    for (int i = 0; i < queries.length; i++)
        ans[i] = comp[queries[i][0]] == comp[queries[i][1]];
    return ans;
}
```

No `Math.abs` on the gap — sorted order guarantees it's non-negative. Worth saying you noticed, rather than defensively wrapping it.

**The transferable lesson:** DSU is built to merge *arbitrary* sets in *arbitrary* order. When your merges are **structurally ordered** (only neighbors ever merge), the disjoint-set machinery is subsumed by a counter. Reach for DSU as the general reflex, then ask whether the input's structure makes it unnecessary.

**Time:** O(n + q) for the scan; O((n + q)·α(n)) for DSU. **Space:** O(n) both.

**Follow-ups worth volunteering:** *what if `nums` weren't sorted?* → sort with original indices attached (O(n log n)), run the same scan, attach labels back to original indices. *What if you needed the minimum number of hops instead of yes/no?* → different technique entirely (greedy furthest-reach jumps or binary lifting); connectivity alone no longer suffices.

### Example problem — Number of Provinces (LC 547, Medium)

> `isConnected[i][j] == 1` means cities `i` and `j` are directly connected. Return the number of provinces (connected components).

The canonical component-count use: union every connected pair, and each *successful* union merges two groups into one, so the count drops by 1.

```java
public int findCircleNum(int[][] isConnected) {
    int n = isConnected.length;
    UnionFind uf = new UnionFind(n);
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)            // upper triangle only — matrix is symmetric
            if (isConnected[i][j] == 1) uf.union(i, j);
    return uf.componentCount();
}
```

**Time:** O(n²·α(n)) — dominated by scanning the matrix. **Space:** O(n).

### Example problem — Redundant Connection (LC 684, Medium)

> A tree with `n` nodes had one extra edge added. Return the edge that can be removed, choosing the one appearing last in `edges` if there are several.

Pure `union`-return-value play: process edges in order; the **first** edge whose `union` returns `false` joins two already-connected nodes — that edge closes the cycle, and processing in input order automatically yields the last valid answer.

```java
public int[] findRedundantConnection(int[][] edges) {
    UnionFind uf = new UnionFind(edges.length + 1);   // nodes are 1-indexed
    for (int[] e : edges)
        if (!uf.union(e[0], e[1])) return e;          // already connected ⇒ this edge makes the cycle
    return new int[0];
}
```

**Time:** O(E·α(n)) **Space:** O(n)

### Tips & tricks

- **`union` returning `false` is the answer to a whole class of problems** — cycle detection, redundant edges, counting merges, Kruskal's edge filter. Always return a boolean from `union`.
- **Write `find` iteratively.** Path halving (`parent[x] = parent[parent[x]]`) is two lines, has no recursion depth risk, and performs essentially as well as full recursive compression.
- **Both optimizations or neither is worth mentioning.** If you skip union-by-size, say so and state the degraded bound — interviewers notice.
- **1-indexed nodes → size the arrays `n + 1`.** Redundant Connection is the classic trap.
- **Undirected cycles only.** For directed graphs, DSU can't help — that's DFS 3-color or Kahn's (Section 7).
- **Deletions ⇒ process in reverse.** "Edges/cells removed over time" becomes "added over time" when you replay the operation list backwards.
- **Grid cells → flatten with `r * n + c`.** Keeps the array-based DSU instead of a slower map-based one.
- **Ask whether the structure makes DSU unnecessary** (LC 3532). Sorted input, interval merges, or neighbor-only unions often collapse to a single labeling pass.

---

## 9. When to Use BFS vs When to Use DFS

**The one-line rule:** the word "shortest" or "minimum steps" on an unweighted graph → BFS. "All / any / count / exists / detect structure" → DFS. Weighted shortest path → neither; that's Dijkstra.

| Signal in the problem | Choose | Why |
|---|---|---|
| Shortest path / min moves / min minutes (unweighted) | **BFS** | First arrival = shortest; that guarantee is BFS-only |
| "Distance to nearest X for every cell" | **Multi-source BFS** | One pass from all sources simultaneously |
| Known start *and* end, huge branching factor | **Bidirectional BFS** | `2·b^(d/2)` beats `b^d` |
| Level-by-level / simultaneous spread ("each minute...") | **BFS** | Rings model time steps exactly |
| Connected components / flood fill / reachability | **Either** (DFS = less code) | Both are O(V+E); no distance needed |
| Incremental connectivity (edges added over time, queries interleaved) | **Union-Find** | Traversal would need a full re-run per query |
| Many connectivity queries on a fixed edge set | **Union-Find** | Preprocess once, O(α(n)) per query |
| Cycle detection in an *undirected* graph | **Union-Find** (or DFS) | A `union` returning false means the edge closes a cycle |
| Enumerate all paths / backtracking / permutations | **DFS** | Call stack carries the current path for free |
| Topological sort / cycle detection on a directed graph | **Either** — Kahn's (BFS) or DFS post-order | Kahn's counts processed nodes; DFS uses the 3-color back-edge check |
| Bridges, articulation points, SCC | **DFS** | Needs DFS-tree structure (back edges, post-order) |
| Longest path on a DAG / counting paths with a recurrence | **DFS + memo** | It's top-down DP in disguise |
| Very deep graph (path-like), recursion risky | **BFS** or iterative DFS | Avoid stack overflow |
| Very wide graph (bushy tree), memory-tight | **DFS** | BFS queue holds a whole ring: O(width); DFS holds one path: O(depth) |
| Tree serialization / "does subtree X..." questions | **DFS** | Post-order aggregation is the natural shape |
| Edge weights all 0 or 1 (some moves free, some cost 1) | **0-1 BFS** (deque) | Front-push 0, back-push 1 keeps the queue sorted — O(V+E), no heap |
| Arbitrary non-negative weights | **Dijkstra** | Min-heap pops nearest unfinished node; ring/deque ordering no longer holds |
| Weighted + side constraint ("≤ K stops") | **Dijkstra, augmented state** | Key `dist` by `(node, stops)`; or Bellman-Ford capped at k+1 rounds |
| Minimax / maximin path (smallest max edge) | **Dijkstra, swap + for max** | Relax with `max` instead of sum; invariant survives (monotonic) |
| Any negative edge weight | **Bellman-Ford** (not Dijkstra) | Dijkstra's "first pop is final" breaks when a later edge can undercut it |

**Memory intuition worth internalizing:** BFS memory ∝ the *widest ring*; DFS memory ∝ the *deepest path*. A complete binary tree of depth 20: BFS's last level holds ~2^19 ≈ 500k nodes, DFS's stack holds 20 frames. A path graph of 10^5 nodes: DFS recursion holds 10^5 frames (overflow!), BFS's queue never exceeds 1.

**Interview tie-breaker:** when either works (components, reachability), say so out loud — "both are O(V+E); I'll use DFS since it's less code, but I'd switch to BFS/iterative if the grid could be deep enough to overflow the stack." That sentence demonstrates exactly the judgment L4 is graded on.

---

## Quick-Reference Complexity Table

| Algorithm | Time | Space |
|---|---|---|
| BFS (adj list) | O(V + E) | O(V) — visited + queue (worst ring) |
| BFS (grid) | O(M·N) | O(M·N) worst, O(min(M,N)) typical ring |
| Multi-source BFS | O(V + E) | O(V) — same as BFS, just more seeds |
| Bidirectional BFS | O(b^(d/2)) effective | O(b^(d/2)) frontier sets |
| 0-1 BFS (deque) | O(V + E) | O(V) — dist array + deque |
| DFS recursive | O(V + E) | O(H) stack, H = longest path (up to O(V)) |
| DFS iterative | O(V + E) | O(V) explicit stack |
| DFS + memo on DAG | O(V + E) | O(V) memo + stack |
| Dijkstra (min-heap) | O(E log V) | O(V + E) — dist + heap (up to O(E) entries) |
| Dijkstra, augmented state (K-constraint) | O(E·K log(V·K)) | O(V·K) states |
| Bellman-Ford (negative edges allowed) | O(V·E) | O(V) |
| Union-Find (path compression + union by size) | O(α(n)) ≈ O(1) amortized per op | O(V) — parent + size arrays |
| Topological sort (Kahn's / DFS) | O(V + E) | O(V + E) — adj + indegree/state + queue/stack |
| Topo sort, lexicographically smallest | O(V log V + E) | O(V + E) — PriorityQueue replaces the queue |
