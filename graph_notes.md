# Graph Notes

> Short and simple. Each topic: the idea in one line, a code template, then LeetCode problems with commented code.
> Language: Java.

## Contents

1. Intro
2. Which algorithm to use when (+ sample questions)
3. Time & space complexity
4. BFS
5. DFS
6. Topological Sort
7. Shortest Path (0-1 BFS, Dijkstra, Bellman-Ford)
8. DSU (Union-Find)
9. MST (Kruskal & Prim)
10. Other graph concepts
11. Tips & tricks

---

## 1. Intro

A **graph** = **nodes** (things) + **edges** (connections between things).

Four questions to ask about any graph:

| Question | Why it matters |
|---|---|
| Directed or undirected? | Changes cycle detection, and whether DSU works |
| Weighted or unweighted? | Picks BFS vs Dijkstra |
| Can it have cycles? | Topological sort needs a DAG (no cycles) |
| Can it be disconnected? | You need a loop over all start nodes |

**Many problems are graphs in disguise.** Always say out loud: *"What is a node? What is an edge? What does an edge cost?"*

| Problem looks like | Node | Edge |
|---|---|---|
| Grid | a cell | to its 4 neighbors |
| Word ladder | a word | words that differ by 1 letter |
| Course schedule | a course | prerequisite → course |
| Combination lock | a 4-digit string | turn one wheel by 1 |

### Representations

**Adjacency list** (most common):

```java
// n nodes labeled 0..n-1, edges[i] = {u, v}
List<List<Integer>> buildGraph(int n, int[][] edges) {
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < n; i++) adj.add(new ArrayList<>());   // one empty list per node
    for (int[] e : edges) {
        adj.get(e[0]).add(e[1]);   // u -> v
        adj.get(e[1]).add(e[0]);   // v -> u  (delete this line for a DIRECTED graph)
    }
    return adj;
}
```

**Labels are not 0..n-1** (strings, sparse ids) → use a map:

```java
Map<String, List<String>> adj = new HashMap<>();
adj.computeIfAbsent(u, k -> new ArrayList<>()).add(v);   // creates the list if missing
```

**Weighted graph** (for Dijkstra / Bellman-Ford):

```java
List<List<int[]>> adj;                          // adj.get(u) = list of {v, weight}
adj.get(u).add(new int[]{v, w});
```

**Grid = implicit graph.** You never build the adjacency list; you generate neighbors on the fly:

```java
int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};     // down, up, right, left (add diagonals if asked)

for (int[] d : DIRS) {
    int nr = r + d[0], nc = c + d[1];
    if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;   // bounds check FIRST
    // ... then visited / wall checks
}
```

**Visited tracking:**
- Nodes `0..n-1` → `boolean[] visited`
- Grid → `boolean[][] visited`, or change the grid in place (e.g. `'1'` → `'0'`)
- Strings / states → `Set<String>`

### Edge cases to say out loud

- `n = 0`, no edges → loops run zero times, don't touch `adj.get(0)`.
- `n = 1`, no edges → one node, so components = 1 (not 0).
- Empty grid → guard at the top: `if (grid.length == 0) return ...;`
- `1 × 1` grid → start = target, answer is `0` (or `1` if the problem counts cells).
- `1 × n` single row → catches swapped `m`/`n` bugs.
- Start or end cell blocked → `-1` right away.
- Disconnected graph, self-loops, duplicate edges → ask if possible.

---

## 2. Which Algorithm to Use When

### Step 1 — Read the verb of the question

The *question* decides the algorithm, not the input shape. "It's a grid" tells you nothing: the same grid can need BFS, Dijkstra, DP, or DSU.

| The question asks... | Use | Sample LeetCode questions |
|---|---|---|
| Is there a path? How many groups? | **DFS** or **DSU** | 200 Number of Islands, 547 Provinces, 3532 Path Existence Queries |
| Fewest steps (every move costs 1) | **BFS** | 752 Open the Lock, 127 Word Ladder |
| Distance to the *nearest* X / spreading over time | **Multi-source BFS** | 994 Rotting Oranges |
| Fewest steps, but some moves are free (cost 0 or 1) | **0-1 BFS** | 2290 Min Obstacle Removal, 1368 Min Cost Valid Path |
| Cheapest path, weights ≥ 0 | **Dijkstra** | 743 Network Delay Time, 787 Cheapest Flights |
| Negative weights, or "at most K edges" | **Bellman-Ford** | 787 Cheapest Flights Within K Stops |
| Minimize the *largest* edge / maximize the *smallest* | **Dijkstra (swap +)**, **DSU**, or **binary search** | 778 Swim in Rising Water, 1102 Max-Min Path |
| Order with prerequisites | **Topological sort** | 207 / 210 Course Schedule, 269 Alien Dictionary, 444 Sequence Reconstruction |
| Cycle in a **directed** graph | **Kahn's** or **DFS 3-color** | 207, 802 Eventual Safe States |
| Cycle in an **undirected** graph | **DSU** | 684 Redundant Connection |
| Edges arrive over time, queries in between | **DSU** | 684, 305 Islands II |
| Connect everything at minimum cost | **MST** | 1584 Connect All Points, 1489 Critical Edges |
| All paths / enumerate everything | **DFS + backtracking** | 797 All Paths Source to Target |
| Longest path in a DAG | **DFS + memo** (or Kahn layers) | 329 Longest Increasing Path |
| Reach from many cells to a few targets | **Reverse the edges**, DFS from targets | 417 Pacific Atlantic |
| Every node has at most 1 outgoing edge | **Functional-graph walk** | 2360 Longest Cycle, 565 Array Nesting, 457 Circular Array Loop |

If you can't name the verb, re-read the problem before coding.

### Step 2 — Shortest path: pick by edge weights

Take the cheapest option that fits:

```
every move costs 1        → BFS            O(V+E)      queue
costs are only 0 or 1     → 0-1 BFS        O(V+E)      deque
any cost ≥ 0              → Dijkstra       O(E log V)  min-heap
negative costs            → Bellman-Ford   O(V·E)      loop over all edges
```

How is an edge combined with the path cost? Usually `+`, but not always:

| Operator | Problem type | Example |
|---|---|---|
| `+` | normal shortest path | 743 |
| `max` | minimize the biggest edge (minimax) | 778 |
| `min` | maximize the smallest value (maximin) | 1102 |
| `×` | maximize probability (max-heap) | 1514 |

Dijkstra still works for all of them as long as extending a path never makes it *better*.

### Step 3 — Is the node alone enough as state?

Ask: *"Can two ways of reaching the same node lead to different futures?"* If yes, add that extra info to the state.

| Problem | Extra info | State |
|---|---|---|
| 787 Cheapest Flights Within K Stops | stops used | `(city, stops)` |
| 1293 Shortest Path with Obstacles Elimination | removals left | `(cell, removalsUsed)` |
| 864 Shortest Path to Get All Keys | keys held | `(cell, keyBitmask)` |

Clue words: "at most K", "you may remove up to", "while carrying". The algorithm stays the same, you just make the state bigger. Don't add extra state when it doesn't affect the future.

### BFS or DFS?

| Signal | Pick | Why |
|---|---|---|
| Shortest / min steps (unweighted) | **BFS** | First arrival = shortest |
| Level by level, "each minute..." | **BFS** | Rings = time steps |
| Components, flood fill, reachability | **Either** (DFS = less code) | No distance needed |
| All paths, backtracking, permutations | **DFS** | Call stack holds the current path |
| Very deep graph (10^5 chain) | **BFS** or iterative DFS | Recursion would overflow |
| Bridges, SCC | **DFS** | Needs DFS-tree structure |

Memory: BFS uses space ∝ the widest level. DFS uses space ∝ the deepest path.

### 30-second routine before coding

1. Name the **nodes and edges** out loud.
2. Name the **verb** → algorithm family.
3. Name the **cost structure** → BFS / 0-1 BFS / Dijkstra / Bellman-Ford.
4. Ask the **state question** → is node alone enough?
5. State the algorithm and its **complexity**, then code.

---

## 3. Time & Space Complexity

V = nodes, E = edges.

| Algorithm | Time | Space |
|---|---|---|
| BFS (adj list) | O(V + E) | O(V) |
| BFS (grid M×N) | O(M·N) | O(M·N) worst |
| Multi-source BFS | O(V + E) | O(V) |
| Bidirectional BFS | O(b^(d/2)) | O(b^(d/2)) |
| 0-1 BFS | O(V + E) | O(V) |
| DFS (recursive) | O(V + E) | O(H) stack, H = longest path (up to V) |
| DFS (iterative) | O(V + E) | O(V) |
| DFS + memo on DAG | O(V + E) | O(V) |
| Topological sort (Kahn / DFS) | O(V + E) | O(V + E) |
| Topo sort, lexicographically smallest | O(V log V + E) | O(V + E) |
| Dijkstra (heap) | O(E log V) | O(V + E) |
| Dijkstra, state = (node, K) | O(E·K log(V·K)) | O(V·K) |
| Bellman-Ford | O(V·E) | O(V) |
| Union-Find (compression + size) | O(α(n)) ≈ O(1) per op | O(V) |
| Kruskal MST | O(E log E) | O(V + E) |
| Prim MST, heap | O(E log V) | O(V + E) |
| Prim MST, array (dense) | O(V²) | O(V) |
| Functional-graph walk | O(n) | O(n) |
| Floyd's tortoise & hare | O(n) | O(1) |

**Why BFS/DFS are O(V + E):** each node is visited once and each edge is looked at once.
**Why Dijkstra is O(E log V):** each edge causes at most one heap push, and a push costs log V.
**Why Bellman-Ford is O(V·E):** V-1 rounds, each round looks at all E edges.
**Why DSU is ~O(1):** α(n) is the inverse Ackermann function, which is ≤ 4 for any real input.

---
## 4. BFS

### Idea

BFS explores in **rings**: everything at distance 1, then distance 2, and so on. So the **first time you reach a node, it's by a shortest path** (counting edges). That's why **BFS = shortest path on an unweighted graph**.

```
S . .      ring 0: (0,0)                   dist 0
. . .      ring 1: (0,1) (1,0)             dist 1
. . .      ring 2: (0,2) (1,1) (2,0)       dist 2
           ring 3: (1,2) (2,1)             dist 3
           ring 4: (2,2)                   dist 4
```

### Template — adjacency list

```java
int bfs(List<List<Integer>> adj, int start, int target) {
    Queue<Integer> queue = new ArrayDeque<>();
    boolean[] visited = new boolean[adj.size()];
    queue.offer(start);
    visited[start] = true;               // mark when you ADD to the queue, not when you remove
    int dist = 0;

    while (!queue.isEmpty()) {
        int size = queue.size();         // freeze the size = exactly one ring
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
        dist++;                          // finished one ring
    }
    return -1;                           // target never reached
}
```

### Template — grid

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
                if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;   // out of grid
                if (visited[nr][nc] || grid[nr][nc] == 1) continue;     // seen or wall (1 = wall here)
                visited[nr][nc] = true;                                  // mark on enqueue
                queue.offer(new int[]{nr, nc});
            }
        }
        dist++;
    }
    return -1;
}
```

**#1 BFS bug:** marking visited when you *remove* from the queue. Still correct, but the same node gets added many times → queue explodes → TLE. Mark when you `offer`.

**Time:** O(V + E) (grid: O(M·N)). **Space:** O(V).

### Multi-source BFS

**Idea:** put **all sources** in the queue at the start (distance 0). The rings spread from every source at once, so each cell's distance = distance to the **nearest** source.

Use it when you see: "distance to the nearest X for every cell" or "how long until it spreads everywhere". Never run BFS once per source — one multi-source pass is enough.

```java
// Only the start changes: add ALL sources, then run the same ring loop.
for (int r = 0; r < m; r++)
    for (int c = 0; c < n; c++)
        if (grid[r][c] == SOURCE) {
            queue.offer(new int[]{r, c});
            visited[r][c] = true;
        }
```

#### LC 994 — Rotting Oranges (Medium)

> Grid: 0 empty, 1 fresh, 2 rotten. Each minute, fresh oranges next to a rotten one rot. Return the minutes until none are fresh, or -1.

**Why multi-source:** every rotten orange spreads at the same time. One ring = one minute.

```java
public int orangesRotting(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    Queue<int[]> queue = new ArrayDeque<>();
    int fresh = 0;

    // 1) add every rotten orange as a source, and count the fresh ones
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++) {
            if (grid[r][c] == 2) queue.offer(new int[]{r, c});
            else if (grid[r][c] == 1) fresh++;
        }

    int minutes = 0;
    // 2) one ring per minute; stop as soon as nothing is fresh (avoids an extra minute)
    while (!queue.isEmpty() && fresh > 0) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] cell = queue.poll();
            for (int[] d : DIRS) {
                int nr = cell[0] + d[0], nc = cell[1] + d[1];
                if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
                if (grid[nr][nc] != 1) continue;     // only fresh oranges can rot
                grid[nr][nc] = 2;                    // rot it (this also marks it visited)
                fresh--;
                queue.offer(new int[]{nr, nc});
            }
        }
        minutes++;                                   // one full minute passed
    }
    return fresh == 0 ? minutes : -1;                // fresh left = some orange is unreachable
}
```

Edge cases: no fresh oranges → `0`. A fresh orange blocked by empty cells → `-1`.
**Time:** O(M·N) **Space:** O(M·N)

#### LC 317 — Shortest Distance from All Buildings (Hard)

> Grid: 1 building, 2 obstacle, 0 empty. Pick an empty cell with the smallest **total** distance to **all** buildings. Return -1 if impossible.

**Why not multi-source?** Multi-source gives the distance to the *nearest* building. Here we need the *sum over all* buildings, so we run **one BFS per building** and add up distances.

For each empty cell we track: `dist` (sum of distances so far) and `reach` (how many buildings can reach it). The answer = smallest `dist` among cells with `reach == total buildings`.

```java
public int shortestDistance(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] dist = new int[m][n];     // sum of distances from every building to this cell
    int[][] reach = new int[m][n];    // how many buildings reached this cell
    int buildings = 0;

    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            if (grid[r][c] == 1) {            // one BFS per building
                buildings++;
                bfs(grid, r, c, dist, reach);
            }

    int ans = Integer.MAX_VALUE;
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            if (grid[r][c] == 0 && reach[r][c] == buildings)   // reachable from ALL buildings
                ans = Math.min(ans, dist[r][c]);
    return ans == Integer.MAX_VALUE ? -1 : ans;
}

private void bfs(int[][] grid, int sr, int sc, int[][] dist, int[][] reach) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    boolean[][] visited = new boolean[m][n];   // fresh visited array for each building
    Queue<int[]> queue = new ArrayDeque<>();
    queue.offer(new int[]{sr, sc});
    visited[sr][sc] = true;
    int steps = 0;

    while (!queue.isEmpty()) {
        steps++;                               // cells found in this ring are `steps` away
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] cell = queue.poll();
            for (int[] d : DIRS) {
                int nr = cell[0] + d[0], nc = cell[1] + d[1];
                if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
                if (visited[nr][nc] || grid[nr][nc] != 0) continue;   // walk on empty land only
                visited[nr][nc] = true;
                dist[nr][nc] += steps;         // add this building's distance
                reach[nr][nc]++;               // this building reached the cell
                queue.offer(new int[]{nr, nc});
            }
        }
    }
}
```

**Time:** O(k · M·N), k = number of buildings. **Space:** O(M·N)

### Bidirectional BFS

**Idea:** run BFS from the **start** and from the **target** at the same time, always expand the **smaller** side, stop when the two sides touch.

**Why it's fast:** if the answer has length `d` and each node has `b` neighbors, normal BFS visits ~`b^d` nodes. Two searches meeting in the middle visit ~`2·b^(d/2)`. For `b = 8, d = 10`: about 10^9 vs 65k.

**Needs:** a known target, and edges that can be walked backwards (undirected). Use `Set`s so `end.contains(x)` is O(1).

#### LC 752 — Open the Lock (Medium)

> 4 wheels, each 0-9. One move turns one wheel ±1 (9↔0 wraps). Given deadends and a target, return the fewest moves from "0000", or -1.

**Graph:** each 4-digit string is a node with 8 neighbors (4 wheels × 2 directions). Deadends are blocked nodes.

```java
public int openLock(String[] deadends, String target) {
    Set<String> dead = new HashSet<>(Arrays.asList(deadends));
    if (dead.contains("0000") || dead.contains(target)) return -1;   // can't start / can't finish
    if (target.equals("0000")) return 0;

    Set<String> begin = new HashSet<>(), end = new HashSet<>();
    begin.add("0000");
    end.add(target);
    Set<String> visited = new HashSet<>(dead);     // treat deadends as already visited
    visited.add("0000");
    visited.add(target);
    int steps = 0;

    while (!begin.isEmpty() && !end.isEmpty()) {
        if (begin.size() > end.size()) {           // always expand the smaller side
            Set<String> tmp = begin; begin = end; end = tmp;
        }
        Set<String> next = new HashSet<>();
        for (String cur : begin) {
            for (String nb : neighbors(cur)) {
                if (end.contains(nb)) return steps + 1;   // the two sides met
                if (visited.add(nb)) next.add(nb);        // add() is false if already visited
            }
        }
        begin = next;
        steps++;
    }
    return -1;
}

// all 8 neighbors: turn each wheel up or down by 1, with wraparound
private List<String> neighbors(String s) {
    List<String> res = new ArrayList<>();
    for (int i = 0; i < 4; i++) {
        char[] a = s.toCharArray();
        char orig = a[i];
        a[i] = (orig == '9') ? '0' : (char) (orig + 1);  res.add(new String(a));   // up
        a[i] = (orig == '0') ? '9' : (char) (orig - 1);  res.add(new String(a));   // down
    }
    return res;
}
```

**Time:** O(10^4 · 8) worst case, usually far less. **Space:** O(10^4)

#### LC 127 — Word Ladder (Hard)

> Change one letter at a time from `beginWord` to `endWord`; every word must be in the list. Return the length of the shortest sequence (counted in **words**), or 0.

**Graph:** words are nodes, an edge joins words that differ by one letter. To find neighbors, try all 26 letters at each position and check the dictionary (cheaper than comparing all pairs).

**Off-by-one trap:** the answer counts **words**, not edges. `hit → hot → dot → dog → cog` is 5.

```java
public int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> dict = new HashSet<>(wordList);
    if (!dict.contains(endWord)) return 0;          // can never finish

    Set<String> begin = new HashSet<>(), end = new HashSet<>();
    begin.add(beginWord);
    end.add(endWord);
    Set<String> visited = new HashSet<>();
    visited.add(beginWord);
    visited.add(endWord);
    int steps = 1;                                  // the begin word itself counts as 1

    while (!begin.isEmpty() && !end.isEmpty()) {
        if (begin.size() > end.size()) {            // expand the smaller side
            Set<String> tmp = begin; begin = end; end = tmp;
        }
        Set<String> next = new HashSet<>();
        for (String word : begin) {
            char[] arr = word.toCharArray();
            for (int i = 0; i < arr.length; i++) {          // each position
                char orig = arr[i];
                for (char c = 'a'; c <= 'z'; c++) {         // each letter
                    if (c == orig) continue;
                    arr[i] = c;
                    String nb = new String(arr);
                    if (end.contains(nb)) return steps + 1; // the two sides met
                    if (dict.contains(nb) && visited.add(nb)) next.add(nb);
                }
                arr[i] = orig;                              // put the letter back
            }
        }
        begin = next;
        steps++;
    }
    return 0;
}
```

**Time:** O(N · L · 26), N = words, L = word length. **Space:** O(N · L)

---

## 5. DFS

### Idea

DFS goes **as deep as possible** down one path, hits a dead end, **backtracks** to the last choice, and tries the next one. The recursion call stack *is* the current path.

- Good for: **can I reach it? what's connected? list everything.**
- Not for: shortest path on an unweighted graph (use BFS).

### Template — adjacency list (recursive)

```java
void dfs(List<List<Integer>> adj, int node, boolean[] visited) {
    visited[node] = true;
    // pre-order: process node here
    for (int next : adj.get(node))
        if (!visited[next]) dfs(adj, next, visited);
    // post-order: all descendants are done (used by topological sort)
}

// Driver: also how you count connected components
int components = 0;
for (int i = 0; i < n; i++)
    if (!visited[i]) { components++; dfs(adj, i, visited); }   // each new start = a new component
```

### Template — iterative (no stack overflow)

```java
void dfsIterative(List<List<Integer>> adj, int start, boolean[] visited) {
    Deque<Integer> stack = new ArrayDeque<>();
    stack.push(start);
    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (visited[node]) continue;        // a node can be pushed many times, so check on pop
        visited[node] = true;
        for (int next : adj.get(node))
            if (!visited[next]) stack.push(next);
    }
}
```

Java's call stack dies around 10k-100k frames. A 10^5-node chain overflows the recursive version.

### Template — grid

```java
void dfs(char[][] grid, int r, int c) {
    if (r < 0 || r >= grid.length || c < 0 || c >= grid[0].length) return;  // out of grid
    if (grid[r][c] != '1') return;       // water, or already visited
    grid[r][c] = '0';                    // "sink" the cell = mark visited in place
    dfs(grid, r + 1, c);
    dfs(grid, r - 1, c);
    dfs(grid, r, c + 1);
    dfs(grid, r, c - 1);
}
```

**Time:** O(V + E) (grid: O(M·N)). **Space:** O(H) stack, H = longest path, which can be O(M·N) on a snake-shaped grid. Say this out loud, it's a common follow-up.

### "Multi-source" and "bidirectional" for DFS?

- Multi-source DFS just means a driver loop that starts DFS from every unvisited node. DFS has no rings, so it can't spread in parallel (Rotting Oranges needs BFS).
- Bidirectional search is a BFS-only trick.
- Useful DFS trick: **reverse the direction** (LC 417 below).

### LC 200 — Number of Islands (Medium)

> Grid of '1' (land) and '0' (water). Count the islands (4-direction connected land).

**Idea:** each island is one connected component. Scan every cell; when you find unvisited land, count +1 and sink the whole island so it isn't counted again.

```java
public int numIslands(char[][] grid) {
    int count = 0;
    for (int r = 0; r < grid.length; r++)
        for (int c = 0; c < grid[0].length; c++)
            if (grid[r][c] == '1') {     // found a new, unvisited island
                count++;
                sink(grid, r, c);        // erase the whole island
            }
    return count;
}

private void sink(char[][] grid, int r, int c) {
    if (r < 0 || r >= grid.length || c < 0 || c >= grid[0].length) return;
    if (grid[r][c] != '1') return;       // water or already sunk
    grid[r][c] = '0';                    // mark visited
    sink(grid, r + 1, c);
    sink(grid, r - 1, c);
    sink(grid, r, c + 1);
    sink(grid, r, c - 1);
}
```

```
1 1 0      scan (0,0): land → count=1, sink 4 cells
1 1 0      scan (2,2): land → count=2, sink 1 cell
0 0 1      answer: 2
```

**Time:** O(M·N) **Space:** O(M·N) recursion worst case (BFS version: O(min(M,N))).
**Follow-ups:** distinct island shapes (LC 694), islands added over time (LC 305 → DSU).

### LC 417 — Pacific Atlantic Water Flow (Medium)

> `heights[r][c]` is the height of a cell. Water flows to neighbors with height ≤ its own. The Pacific touches the top and left edges, the Atlantic the bottom and right edges. Return all cells that can reach **both** oceans.

**Trick: reverse the direction.** Asking "can each cell reach the ocean?" is slow (a search from every cell). Instead start **at the ocean borders** and go **uphill** (water in reverse). Do it once for each ocean. The answer = cells reached by both.

```java
private int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};

public List<List<Integer>> pacificAtlantic(int[][] heights) {
    int m = heights.length, n = heights[0].length;
    boolean[][] pacific = new boolean[m][n], atlantic = new boolean[m][n];

    for (int r = 0; r < m; r++) {
        dfs(heights, r, 0, pacific);        // left edge touches the Pacific
        dfs(heights, r, n - 1, atlantic);   // right edge touches the Atlantic
    }
    for (int c = 0; c < n; c++) {
        dfs(heights, 0, c, pacific);        // top edge touches the Pacific
        dfs(heights, m - 1, c, atlantic);   // bottom edge touches the Atlantic
    }

    List<List<Integer>> res = new ArrayList<>();
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            if (pacific[r][c] && atlantic[r][c])    // reaches both oceans
                res.add(List.of(r, c));
    return res;
}

// walk UPHILL from the ocean: if water can flow from the neighbor down to us, mark it
private void dfs(int[][] h, int r, int c, boolean[][] seen) {
    seen[r][c] = true;
    for (int[] d : DIRS) {
        int nr = r + d[0], nc = c + d[1];
        if (nr < 0 || nr >= h.length || nc < 0 || nc >= h[0].length) continue;
        if (seen[nr][nc]) continue;
        if (h[nr][nc] < h[r][c]) continue;          // neighbor is lower → water can't come from it
        dfs(h, nr, nc, seen);
    }
}
```

**Time:** O(M·N) (two sweeps) vs O((M·N)²) for the naive way. **Space:** O(M·N)

### LC 329 — Longest Increasing Path in a Matrix (Hard)

> Find the length of the longest strictly increasing path in a matrix (4 directions).

**Idea:** draw an edge from a cell to each bigger neighbor. Strictly increasing means **no cycles**, so the grid is a **DAG**. That's DFS + memoization (top-down DP).

- `memo[r][c]` = longest increasing path **starting** at `(r, c)`.
- `memo[r][c] = 1 + max(memo of bigger neighbors)`, or `1` if none.

**No visited array needed.** You can never walk back to a smaller-or-equal cell, and the memo stops repeated work.

```java
private int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
private int[][] memo;                       // 0 = not computed yet (every real answer is ≥ 1)

public int longestIncreasingPath(int[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    memo = new int[m][n];
    int best = 0;
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            best = Math.max(best, dfs(matrix, r, c));   // try every cell as the start
    return best;
}

private int dfs(int[][] mat, int r, int c) {
    if (memo[r][c] != 0) return memo[r][c];             // already solved
    int best = 1;                                        // the cell alone
    for (int[] d : DIRS) {
        int nr = r + d[0], nc = c + d[1];
        if (nr < 0 || nr >= mat.length || nc < 0 || nc >= mat[0].length) continue;
        if (mat[nr][nc] <= mat[r][c]) continue;          // must be strictly bigger
        best = Math.max(best, 1 + dfs(mat, nr, nc));
    }
    return memo[r][c] = best;                            // save and return
}
```

**Faster-to-explain alternative — peel layers (Kahn's, no recursion):** a cell's out-degree = number of bigger neighbors. Start from cells with out-degree 0 (local maxima = path ends) and peel layer by layer going to smaller neighbors. **The number of layers = the longest path.**

```java
public int longestIncreasingPath(int[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    int[][] outdeg = new int[m][n];          // number of strictly bigger neighbors
    Queue<int[]> queue = new ArrayDeque<>();

    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++) {
            for (int[] d : DIRS) {
                int nr = r + d[0], nc = c + d[1];
                if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
                if (matrix[nr][nc] > matrix[r][c]) outdeg[r][c]++;
            }
            if (outdeg[r][c] == 0) queue.offer(new int[]{r, c});   // no bigger neighbor = path end
        }

    int layers = 0;
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] cell = queue.poll();
            for (int[] d : DIRS) {                               // go to SMALLER neighbors
                int nr = cell[0] + d[0], nc = cell[1] + d[1];
                if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
                if (matrix[nr][nc] >= matrix[cell[0]][cell[1]]) continue;
                if (--outdeg[nr][nc] == 0) queue.offer(new int[]{nr, nc});   // all its bigger ones are done
            }
        }
        layers++;                                                // one more step of path length
    }
    return layers;
}
```

**Time:** O(M·N) both ways. **Space:** O(M·N). (Row-by-row space saving doesn't work here: a cell can depend on neighbors in any direction.)

---
## 6. Topological Sort

### Idea

A topological sort puts the nodes of a **DAG** (directed, no cycles) in a line so that for every edge `u → v`, `u` comes **before** `v`. Think: *"everything I depend on is placed before me."*

Two ways:

- **Kahn's (BFS):** repeatedly take a node with **indegree 0** (no remaining prerequisites), add it to the order, and remove its outgoing edges. That may free new indegree-0 nodes. If you get stuck with nodes left over, those nodes are in a **cycle**.
- **DFS:** push a node onto a stack when its DFS **finishes**. Reading the stack top to bottom gives a valid order.

```
Edges: 0→1, 0→2, 1→3, 2→3        indegree: [0, 1, 1, 2]

take 0 → order [0]         indegree becomes [_, 0, 0, 2]
take 1 → order [0,1]       indegree becomes [_, _, 0, 1]
take 2 → order [0,1,2]     indegree becomes [_, _, _, 0]
take 3 → order [0,1,2,3]
```

When 1 and 2 are both ready, either can go first. That means **more than one valid order exists** (queue size > 1).

### Template — Kahn's

```java
// Returns a valid order, or an EMPTY list if there is a cycle.
List<Integer> topoSort(int n, List<List<Integer>> adj) {      // adj.get(u) = nodes u points to
    int[] indegree = new int[n];
    for (int u = 0; u < n; u++)
        for (int v : adj.get(u)) indegree[v]++;                // count incoming edges

    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < n; i++)
        if (indegree[i] == 0) queue.offer(i);                  // start with all "roots"

    List<Integer> order = new ArrayList<>();
    while (!queue.isEmpty()) {
        int u = queue.poll();
        order.add(u);
        for (int v : adj.get(u))
            if (--indegree[v] == 0) queue.offer(v);            // v has no prerequisites left
    }
    return order.size() == n ? order : new ArrayList<>();      // fewer than n ⇒ cycle
}
```

### Template — DFS (3 colors)

```java
// state: 0 = not visited, 1 = on the current path, 2 = fully done
int[] state;
Deque<Integer> stack;

List<Integer> topoSortDFS(int n, List<List<Integer>> adj) {
    state = new int[n];
    stack = new ArrayDeque<>();
    for (int i = 0; i < n; i++)
        if (state[i] == 0 && !dfs(adj, i)) return new ArrayList<>();   // cycle found
    return new ArrayList<>(stack);                                       // top → bottom = topo order
}

boolean dfs(List<List<Integer>> adj, int u) {
    state[u] = 1;                              // entering u: it's on the current path
    for (int v : adj.get(u)) {
        if (state[v] == 1) return false;       // edge back to the current path ⇒ CYCLE
        if (state[v] == 0 && !dfs(adj, v)) return false;
    }
    state[u] = 2;                              // leaving u: all descendants done
    stack.push(u);                             // push AFTER descendants ⇒ reverse post-order
    return true;
}
```

**Why 3 colors, not just visited?** In a directed graph, reaching an already-finished node (two paths lead to it) is fine. Only an edge to a node **still on the current path** is a cycle.

**Time:** O(V + E) **Space:** O(V + E)

### Variations

| You need... | Change |
|---|---|
| Detect a cycle | Kahn's: `order.size() != n`. DFS: edge to a state-1 node |
| Is the order **unique**? | In Kahn's, if the queue ever has **more than 1** node → not unique |
| **Lexicographically smallest** order | Replace the `Queue` with a `PriorityQueue` (min-heap): O(V log V + E) |
| Longest path / count paths in a DAG | Process in topo order and do a DP step on each pop |
| Graph is hidden | The work is **building** the edges (Alien Dictionary, Sequence Reconstruction) |
| Undirected tree | Peel **degree-1** nodes, stop at a target count (LC 310) |
| Isolated nodes | Register them with indegree 0 or they vanish from the output |

### LC 207 / 210 — Course Schedule I & II (Medium)

> `prerequisites[i] = [a, b]` means take `b` before `a`.
> 207: can you finish all courses? 210: return one valid order (or empty if impossible).

**Direction trap:** `[a, b]` means the edge is `b → a` (prerequisite points to the course it unlocks).

```java
// LC 207: can finish all courses? ⇔ no cycle ⇔ Kahn's takes every course
public boolean canFinish(int numCourses, int[][] prerequisites) {
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) adj.add(new ArrayList<>());
    int[] indegree = new int[numCourses];
    for (int[] p : prerequisites) {          // p = {course, prereq}
        adj.get(p[1]).add(p[0]);             // prereq → course
        indegree[p[0]]++;
    }

    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < numCourses; i++)
        if (indegree[i] == 0) queue.offer(i);    // courses with no prerequisites

    int taken = 0;
    while (!queue.isEmpty()) {
        int c = queue.poll();
        taken++;                                  // take this course
        for (int next : adj.get(c))
            if (--indegree[next] == 0) queue.offer(next);   // one fewer prerequisite
    }
    return taken == numCourses;                   // all taken = no cycle
}
```

```java
// LC 210: same code, but record the order. Return empty if a cycle blocks some courses.
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
        order[idx++] = c;                         // record the course in order
        for (int next : adj.get(c))
            if (--indegree[next] == 0) queue.offer(next);
    }
    return idx == numCourses ? order : new int[0];   // cycle ⇒ empty array
}
```

**Time:** O(V + E) **Space:** O(V + E)

### LC 444 — Sequence Reconstruction (Medium)

> `nums` is a permutation of 1..n. `sequences` is a list of subsequences. Return true only if `nums` is the **only** shortest sequence that has all of them as subsequences.

**Idea:** each sequence gives edges between **consecutive** elements. The order is unique exactly when, at every step of Kahn's, the queue has **exactly one** node. That one order must also equal `nums`.

```java
public boolean sequenceReconstruction(int[] nums, List<List<Integer>> sequences) {
    int n = nums.length;
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i <= n; i++) adj.add(new ArrayList<>());
    int[] indegree = new int[n + 1];
    Arrays.fill(indegree, -1);                  // -1 = this value never appeared in any sequence

    for (List<Integer> seq : sequences) {
        for (int i = 0; i < seq.size(); i++) {
            int cur = seq.get(i);
            if (cur < 1 || cur > n) return false;       // value outside 1..n
            if (indegree[cur] == -1) indegree[cur] = 0; // mark as "seen"
            if (i > 0) {
                adj.get(seq.get(i - 1)).add(cur);       // edge: previous → current
                indegree[cur]++;
            }
        }
    }

    Queue<Integer> queue = new ArrayDeque<>();
    for (int v = 1; v <= n; v++)
        if (indegree[v] == 0) queue.offer(v);           // unseen values stay at -1 and are skipped

    int idx = 0;
    while (!queue.isEmpty()) {
        if (queue.size() > 1) return false;             // 2+ choices ⇒ order not unique
        int node = queue.poll();
        if (node != nums[idx++]) return false;          // must match nums
        for (int next : adj.get(node))
            if (--indegree[next] == 0) queue.offer(next);
    }
    return idx == n;                                    // all placed ⇒ no cycle, full coverage
}
```

**Time:** O(V + E) **Space:** O(V + E)

### LC 269 — Alien Dictionary (Hard)

> `words` are sorted in an unknown alphabet. Return any valid letter order, or "" if impossible.

**Idea:** the hard part is **building the graph**. Compare each pair of **adjacent words**; the **first** position where they differ gives one edge (`letter in word1 → letter in word2`). Then topo sort the letters.

Three traps:
1. **Prefix case:** `["abc", "ab"]` is impossible → return `""`.
2. **Only the first difference** gives an edge. Later letters tell you nothing.
3. **Register every letter**, even those with no edges, or they vanish from the output.

```java
public String alienOrder(String[] words) {
    Map<Character, Set<Character>> adj = new HashMap<>();
    Map<Character, Integer> indegree = new HashMap<>();

    // 1) register every letter that appears
    for (String word : words)
        for (char c : word.toCharArray()) {
            adj.putIfAbsent(c, new HashSet<>());
            indegree.putIfAbsent(c, 0);
        }

    // 2) build edges from adjacent word pairs
    for (int i = 0; i < words.length - 1; i++) {
        String w1 = words[i], w2 = words[i + 1];
        if (w1.length() > w2.length() && w1.startsWith(w2)) return "";   // prefix trap
        for (int j = 0; j < Math.min(w1.length(), w2.length()); j++) {
            char c1 = w1.charAt(j), c2 = w2.charAt(j);
            if (c1 != c2) {
                if (adj.get(c1).add(c2))                           // add() is false for a duplicate edge
                    indegree.put(c2, indegree.get(c2) + 1);        // count each edge only once
                break;                                             // only the FIRST difference counts
            }
        }
    }

    // 3) Kahn's algorithm
    Deque<Character> queue = new ArrayDeque<>();
    for (char c : indegree.keySet())
        if (indegree.get(c) == 0) queue.offer(c);

    StringBuilder res = new StringBuilder();
    while (!queue.isEmpty()) {
        char c = queue.poll();
        res.append(c);
        for (char next : adj.get(c)) {
            indegree.put(next, indegree.get(next) - 1);
            if (indegree.get(next) == 0) queue.offer(next);
        }
    }
    return res.length() == indegree.size() ? res.toString() : "";   // leftover letters ⇒ cycle
}
```

**The duplicate-edge guard matters.** Example `["ac","ab","zc","zb"]` creates the edge `c→b` twice. If you count the indegree twice but the `Set` stores it once, `b` never reaches 0 and you falsely report a cycle. Rule: **the indegree count must match the number of edges stored.**

**Lexicographically smallest answer?** Use a `PriorityQueue<Character>` instead of the queue. A sorted-seeded FIFO queue is *not* the same: the tie-break must happen at every poll.

**Time:** O(C), C = total characters in all words. **Space:** O(1) (≤ 26 letters).

### LC 802 — Find Eventual Safe States (Medium)

> A node is **terminal** if it has no outgoing edges. A node is **safe** if every path from it ends at a terminal node. Return all safe nodes in ascending order.

**Idea:** *a node is safe if all its successors are safe.* Terminal nodes are safe (base case). That condition is about **out**-edges, but Kahn's peels by **in**-edges, so **reverse the graph** and peel from the terminals.

- `remaining[u]` = number of successors of `u` not yet proven safe.
- When `u` is proven safe, every predecessor of `u` loses one unsafe successor. At 0 → it's safe too.
- Nodes on or leading into a cycle never reach 0.

```
graph = [[1,2],[2,3],[5],[0],[5],[],[]]      cycle: 0 → 1 → 3 → 0
seed terminals: 5, 6
peel 5 → preds 2, 4 drop to 0 → safe
peel 6, 2, 4 ...
0, 1, 3 never reach 0 (they are in the cycle).  Safe = [2, 4, 5, 6]
```

```java
public List<Integer> eventualSafeNodes(int[][] graph) {
    int n = graph.length;

    List<List<Integer>> reverse = new ArrayList<>();    // reverse.get(v) = nodes pointing INTO v
    for (int i = 0; i < n; i++) reverse.add(new ArrayList<>());
    int[] remaining = new int[n];                        // successors not yet proven safe

    for (int u = 0; u < n; u++) {
        remaining[u] = graph[u].length;
        for (int v : graph[u]) reverse.get(v).add(u);    // flip each edge u → v
    }

    Deque<Integer> queue = new ArrayDeque<>();
    for (int u = 0; u < n; u++)
        if (remaining[u] == 0) queue.offer(u);           // terminal nodes are safe

    boolean[] safe = new boolean[n];
    while (!queue.isEmpty()) {
        int u = queue.poll();
        safe[u] = true;
        for (int pred : reverse.get(u))
            if (--remaining[pred] == 0) queue.offer(pred);   // all of pred's successors are safe
    }

    List<Integer> ans = new ArrayList<>();
    for (int u = 0; u < n; u++) if (safe[u]) ans.add(u);     // scanning gives ascending order for free
    return ans;
}
```

**Time:** O(V + E) **Space:** O(V + E)

### LC 310 — Minimum Height Trees (Medium)

> Given a tree, return every node that gives the smallest height when used as the root.

**Idea:** the best roots are the **center** of the tree (the middle of its longest path). A tree's center is **1 or 2 nodes, never 3**. Peel the leaves layer by layer; the last 1-2 nodes left are the center.

Two differences from normal Kahn's:
- A leaf here is **degree 1** (an undirected leaf still has its one edge).
- **Stop when 2 or fewer nodes remain.** If you peel the last 2, you delete the answer.

```
a — b — c — d      leaves a, d. Peel → b, c remain (2 nodes) → STOP → answer [b, c]
```

```java
public List<Integer> findMinHeightTrees(int n, int[][] edges) {
    if (n == 1) return Collections.singletonList(0);      // single node: no edges, no leaves

    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
    for (int[] e : edges) {
        adj.get(e[0]).add(e[1]);
        adj.get(e[1]).add(e[0]);                          // undirected: add both ways
    }

    int[] degree = new int[n];
    Deque<Integer> leaves = new ArrayDeque<>();
    for (int i = 0; i < n; i++) {
        degree[i] = adj.get(i).size();
        if (degree[i] == 1) leaves.offer(i);              // leaf = degree 1
    }

    int remaining = n;
    while (remaining > 2) {                               // stop before peeling the center
        int size = leaves.size();                         // one layer of leaves
        remaining -= size;
        for (int i = 0; i < size; i++) {
            int u = leaves.poll();
            for (int v : adj.get(u))
                if (--degree[v] == 1) leaves.offer(v);    // v just became a leaf
        }
    }
    return new ArrayList<>(leaves);                       // the 1 or 2 center nodes
}
```

**Time:** O(n) **Space:** O(n)

---

## 7. Shortest Path

### Pick the algorithm by edge weights

```
every move costs 1        → BFS (Section 4)
costs are only 0 or 1     → 0-1 BFS
any cost ≥ 0              → Dijkstra
negative costs / ≤ K hops → Bellman-Ford
```

### 7.1 — 0-1 BFS

**Idea:** like BFS, but edges cost 0 or 1. Use a **deque**: a **0-cost** move goes to the **front** (same ring, handle it now), a **1-cost** move goes to the **back** (next ring). The deque stays sorted by distance, so you get Dijkstra's result in O(V + E) with no heap.

**Why plain BFS breaks:** a 0-cost neighbor is at the *same* distance. Putting it at the back would put it behind farther nodes.

Two differences from normal BFS:
1. **No visited-on-enqueue.** A node can be improved later, so the check is `dist[u] + w < dist[v]`.
2. **No ring-size loop.** Distances live in `dist[]`.

```java
int zeroOneBFS(int[][] grid) {      // cost to ENTER a cell = grid value (0 or 1); go top-left → bottom-right
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
            int w = grid[nr][nc];                          // 0 or 1
            if (dist[r][c] + w < dist[nr][nc]) {           // found a cheaper way
                dist[nr][nc] = dist[r][c] + w;
                if (w == 0) deque.offerFirst(new int[]{nr, nc});   // free move → front
                else        deque.offerLast(new int[]{nr, nc});    // paid move → back
            }
        }
    }
    return dist[m - 1][n - 1];
}
```

**Time:** O(V + E) **Space:** O(V)

**Recognize it:** looks like grid BFS but **some moves are free, some cost 1**.

#### LC 2290 — Minimum Obstacle Removal to Reach Corner (Hard)

> Grid: 0 empty, 1 obstacle. Go from top-left to bottom-right, removing obstacles as needed. Return the fewest removals.

**Why 0-1 BFS:** stepping onto an empty cell costs 0, stepping onto an obstacle costs 1 (remove it). The grid value *is* the edge weight, so this is exactly the template above.

```java
public int minimumObstacles(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    int[][] dist = new int[m][n];                 // dist = fewest obstacles removed to reach the cell
    for (int[] row : dist) Arrays.fill(row, Integer.MAX_VALUE);
    Deque<int[]> deque = new ArrayDeque<>();
    dist[0][0] = 0;
    deque.offerFirst(new int[]{0, 0});

    while (!deque.isEmpty()) {
        int[] cur = deque.pollFirst();
        int r = cur[0], c = cur[1];
        for (int[] d : DIRS) {
            int nr = r + d[0], nc = c + d[1];
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
            int w = grid[nr][nc];                  // 1 = remove an obstacle (cost 1), 0 = free
            if (dist[r][c] + w < dist[nr][nc]) {
                dist[nr][nc] = dist[r][c] + w;
                if (w == 0) deque.offerFirst(new int[]{nr, nc});   // free → handle now
                else        deque.offerLast(new int[]{nr, nc});    // costs 1 → later
            }
        }
    }
    return dist[m - 1][n - 1];
}
```

**Time:** O(M·N) **Space:** O(M·N)

#### LC 1368 — Minimum Cost to Make at Least One Valid Path in a Grid (Hard)

> Each cell has an arrow: 1 right, 2 left, 3 down, 4 up. Following the cell's own arrow is free. Changing a cell's arrow costs 1. Min cost to get from top-left to bottom-right?

**Why 0-1 BFS:** from a cell, the move in the direction of its arrow costs **0**; the other 3 directions cost **1** (you rewrite the arrow). Same as 2290, only the way you compute `w` changes.

```java
public int minCost(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{0,1},{0,-1},{1,0},{-1,0}};   // index + 1 matches the arrow value: 1 right, 2 left, 3 down, 4 up
    int[][] dist = new int[m][n];
    for (int[] row : dist) Arrays.fill(row, Integer.MAX_VALUE);
    Deque<int[]> deque = new ArrayDeque<>();
    dist[0][0] = 0;
    deque.offerFirst(new int[]{0, 0});

    while (!deque.isEmpty()) {
        int[] cur = deque.pollFirst();
        int r = cur[0], c = cur[1];
        for (int k = 0; k < 4; k++) {
            int nr = r + DIRS[k][0], nc = c + DIRS[k][1];
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
            int w = (grid[r][c] == k + 1) ? 0 : 1;        // 0 if we follow the arrow, else 1
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

**Time:** O(M·N) **Space:** O(M·N)

### 7.2 — Dijkstra

**Idea:** BFS for weighted graphs. Always expand the **unfinished node with the smallest known distance**, using a **min-heap**.

**Why a heap:** BFS's queue is sorted only because every edge costs 1. With different weights, a cheap 2-hop path can beat an expensive 1-hop path, so you need the truly closest node next.

**Key fact:** the first time a node is **popped** from the heap, its distance is **final**. (Every other path to it goes through nodes that are already at least as far, and weights are ≥ 0.) This is exactly why **negative edges break Dijkstra**.

```
A→B(4)  A→C(1)  C→B(2)  C→D(5)  B→D(1),   start at A

pop A(0): B=4, C=1
pop C(1): B = 1+2 = 3 (better than 4), D = 6
pop B(3): D = 3+1 = 4 (better than 6)
pop D(4): done.   A=0  B=3  C=1  D=4
(the old (4,B) and (6,D) entries are stale and get skipped)
```

```java
int[] dijkstra(int n, List<List<int[]>> adj, int src) {    // adj.get(u) = list of {neighbor, weight}
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);   // {distance, node}, smallest first
    pq.offer(new int[]{0, src});

    while (!pq.isEmpty()) {
        int[] top = pq.poll();
        int d = top[0], u = top[1];
        if (d > dist[u]) continue;              // LAZY DELETION: old entry, a better one was already used
        for (int[] e : adj.get(u)) {
            int v = e[0], w = e[1];
            if (dist[u] + w < dist[v]) {        // relaxation: found a cheaper path to v
                dist[v] = dist[u] + w;
                pq.offer(new int[]{dist[v], v});
            }
        }
    }
    return dist;
}
```

**Lazy deletion:** Java's `PriorityQueue` can't lower a key. So you push a *new* smaller entry and skip the old one at pop time with `if (d > dist[u]) continue;`. Always write it.

**Early exit:** you can return the moment you **pop** the target (never when you push it).

**Time:** O(E log V) **Space:** O(V + E)

**Use it for:** weighted shortest path, weights ≥ 0.
**Don't use it for:** negative weights (Bellman-Ford), all-equal weights (BFS), weights 0/1 (0-1 BFS).

**Two knobs turn Dijkstra into most hard problems:**

| You need... | Change |
|---|---|
| A side limit like "≤ K stops" | **Bigger state** `(node, stops)` |
| Minimax / maximin path | Combine with `max` / `min` instead of `+` |
| Maximize a product (probability) | Max-heap, multiply |
| Count shortest paths | `ways[]`: on a strictly better path `ways[v] = ways[u]`; on an equal path `ways[v] += ways[u]` |

#### LC 743 — Network Delay Time (Medium) · the clean one

> `times[i] = [u, v, w]` is a directed edge with travel time `w`. A signal starts at node `k`. Return the time for **all** `n` nodes to receive it, or -1.

**Idea:** run Dijkstra from `k`. The time for everyone = the **largest** shortest distance. If any node is unreachable, return -1.

```java
public int networkDelayTime(int[][] times, int n, int k) {
    List<List<int[]>> adj = new ArrayList<>();
    for (int i = 0; i <= n; i++) adj.add(new ArrayList<>());      // nodes are 1-indexed → size n+1
    for (int[] t : times) adj.get(t[0]).add(new int[]{t[1], t[2]});   // {to, time}

    int[] dist = new int[n + 1];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[k] = 0;
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, k});

    while (!pq.isEmpty()) {
        int[] top = pq.poll();
        int d = top[0], u = top[1];
        if (d > dist[u]) continue;                // stale entry
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
        if (dist[i] == Integer.MAX_VALUE) return -1;   // some node never got the signal
        ans = Math.max(ans, dist[i]);                  // the slowest node decides the time
    }
    return ans;
}
```

**Time:** O(E log V) **Space:** O(V + E)

#### LC 787 — Cheapest Flights Within K Stops (Medium) · bigger state

> Find the cheapest price from `src` to `dst` with **at most `k` stops** (so at most `k+1` flights), or -1.

**Why plain Dijkstra fails:** the cheapest path may use too many stops. Price alone isn't enough, so the state becomes **`(city, stops)`**. You can't just skip a city you've seen: reaching it again with **fewer stops** can still matter.

```java
public int findCheapestPrice(int n, int[][] flights, int src, int dst, int k) {
    List<List<int[]>> adj = new ArrayList<>();
    for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
    for (int[] f : flights) adj.get(f[0]).add(new int[]{f[1], f[2]});   // {to, price}

    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);   // {cost, city, stopsUsed}
    pq.offer(new int[]{0, src, 0});
    int[] minStops = new int[n];                       // fewest stops seen so far when popping each city
    Arrays.fill(minStops, Integer.MAX_VALUE);

    while (!pq.isEmpty()) {
        int[] top = pq.poll();
        int cost = top[0], city = top[1], stops = top[2];
        if (city == dst) return cost;                  // first pop of dst = cheapest valid path
        if (stops > k) continue;                       // no stops left to continue
        if (stops >= minStops[city]) continue;         // an earlier (cheaper) visit already had ≤ stops here
        minStops[city] = stops;
        for (int[] e : adj.get(city))
            pq.offer(new int[]{cost + e[1], e[0], stops + 1});
    }
    return -1;
}
```

Bellman-Ford solves this more simply (see 7.3).
**Time:** O(E·K·log) **Space:** O(V + E)

#### LC 778 — Swim in Rising Water (Hard) · minimax

> `grid[r][c]` is the elevation. At time `t` you can enter cells with elevation ≤ `t`; swimming is instant. Return the least time to go from top-left to bottom-right.

**Key insight:** the time for a path = its **highest** cell. You want the path whose highest cell is **smallest** (minimax). So it's Dijkstra with **`max` instead of `+`**.

```java
public int swimInWater(int[][] grid) {
    int n = grid.length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    int[][] time = new int[n][n];                 // best possible "highest cell on the path" to reach each cell
    for (int[] row : time) Arrays.fill(row, Integer.MAX_VALUE);
    time[0][0] = grid[0][0];
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);   // {pathMax, row, col}
    pq.offer(new int[]{grid[0][0], 0, 0});

    while (!pq.isEmpty()) {
        int[] top = pq.poll();
        int t = top[0], r = top[1], c = top[2];
        if (t > time[r][c]) continue;                         // stale entry
        if (r == n - 1 && c == n - 1) return t;               // popped the target = optimal
        for (int[] d : DIRS) {
            int nr = r + d[0], nc = c + d[1];
            if (nr < 0 || nr >= n || nc < 0 || nc >= n) continue;
            int nt = Math.max(t, grid[nr][nc]);               // MAX, not sum: the only change
            if (nt < time[nr][nc]) {
                time[nr][nc] = nt;
                pq.offer(new int[]{nt, nr, nc});
            }
        }
    }
    return time[n - 1][n - 1];
}
```

**Why Dijkstra still works:** `max` never makes a path better when you extend it, so "first pop is final" still holds.
**Three solutions to name:** Dijkstra (above), **DSU** adding cells in sorted order (Section 8), **binary search** on `t` + BFS ("can I cross using only cells ≤ t?").
**Time:** O(n² log n) **Space:** O(n²)

#### LC 1102 — Path With Maximum Minimum Value (Medium) · maximin

> Path score = the **smallest** value on the path. Find the path from top-left to bottom-right with the **largest** score.

**Mirror of 778:** max-heap, and combine with **`min`**.

```java
public int maximumMinimumPath(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
    int[][] best = new int[m][n];                  // best score (path minimum) found for each cell
    for (int[] row : best) Arrays.fill(row, -1);
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> b[2] - a[2]);   // MAX-heap on score
    pq.offer(new int[]{0, 0, grid[0][0]});
    best[0][0] = grid[0][0];

    while (!pq.isEmpty()) {
        int[] cur = pq.poll();
        int r = cur[0], c = cur[1], s = cur[2];
        if (s < best[r][c]) continue;                          // stale entry
        if (r == m - 1 && c == n - 1) return s;                // first pop of the end = optimal
        for (int[] d : DIRS) {
            int nr = r + d[0], nc = c + d[1];
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
            int ns = Math.min(s, grid[nr][nc]);                // path score = weakest cell so far
            if (ns > best[nr][nc]) {
                best[nr][nc] = ns;
                pq.offer(new int[]{nr, nc, ns});
            }
        }
    }
    return best[m - 1][n - 1];
}
```

**Time:** O(mn log mn) **Space:** O(mn)

### 7.3 — Bellman-Ford

**Idea:** relax **every edge**, `V-1` times. After round `i`, each node holds its cheapest cost using **at most `i` edges**.

That "at most `i` edges" property is why it's useful: it handles **negative weights** and **hop limits**, which Dijkstra can't.

```java
int[] dist = new int[n];
Arrays.fill(dist, Integer.MAX_VALUE);
dist[src] = 0;

for (int i = 0; i < n - 1; i++) {                       // V-1 rounds
    for (int[] e : edges) {                              // relax EVERY edge each round
        int u = e[0], v = e[1], w = e[2];
        if (dist[u] == Integer.MAX_VALUE) continue;      // unreached: MAX_VALUE + w would overflow
        if (dist[u] + w < dist[v]) dist[v] = dist[u] + w;
    }
}
// Run one extra round: if anything still improves, there is a negative cycle.
```

**Time:** O(V·E) **Space:** O(V)

**Use it for:** negative weights, "at most K edges", negative-cycle detection.
**Dijkstra vs Bellman-Ford:** same relaxation `dist[u] + w < dist[v]`. Dijkstra picks the next node with a heap (fast, no negatives, no hop count). Bellman-Ford blindly sweeps all edges (slower, handles both).

#### LC 787 — Cheapest Flights Within K Stops (again, the easy way)

**Why it fits:** "at most `k` stops" = "at most `k+1` edges" = exactly `k+1` rounds of Bellman-Ford.

```java
public int findCheapestPrice(int n, int[][] flights, int src, int dst, int k) {
    int[] prices = new int[n];
    Arrays.fill(prices, Integer.MAX_VALUE);
    prices[src] = 0;

    for (int i = 0; i <= k; i++) {                         // k+1 rounds = at most k+1 flights
        int[] tmp = Arrays.copyOf(prices, n);              // write to a COPY, read from last round
        for (int[] f : flights) {
            int from = f[0], to = f[1], price = f[2];
            if (prices[from] == Integer.MAX_VALUE) continue;          // not reachable yet
            if (prices[from] + price < tmp[to]) tmp[to] = prices[from] + price;
        }
        prices = tmp;                                      // finish the round
    }
    return prices[dst] == Integer.MAX_VALUE ? -1 : prices[dst];
}
```

**The copy is the one trap.** Without it, `A→B` and then `B→C` could both be used in the **same round**, so a path would use 2 edges in 1 round and break the stop limit.
**Time:** O(k·E) **Space:** O(V)

---
## 8. DSU (Union-Find)

### Idea

DSU keeps **groups of nodes** and supports two operations:

- `find(x)` → the group's "leader" (root) of `x`
- `union(a, b)` → merge the groups of `a` and `b`

"Are `a` and `b` connected?" becomes `find(a) == find(b)`. Each group is a tree and only the **root** matters, so two speed-ups are allowed:

- **Path compression:** while going up in `find`, point nodes closer to the root.
- **Union by size:** attach the smaller tree under the bigger one.

Together: **O(α(n)) ≈ O(1)** per operation.

**DSU vs BFS/DFS:** BFS/DFS answers connectivity on a graph you already have. DSU handles **edges arriving over time with queries in between**. But DSU only **merges**. It can't delete an edge. (For deletions, replay the operations **in reverse**, so deletions become additions.)

### Template

```java
class UnionFind {
    private int[] parent, size;
    private int count;                           // number of separate groups

    UnionFind(int n) {
        parent = new int[n];
        size = new int[n];
        count = n;                               // at first, everyone is alone
        for (int i = 0; i < n; i++) { parent[i] = i; size[i] = 1; }
    }

    // iterative find with path halving (no recursion, no stack overflow)
    int find(int x) {
        while (parent[x] != x) {
            parent[x] = parent[parent[x]];       // jump to the grandparent
            x = parent[x];
        }
        return x;
    }

    // returns false if a and b were ALREADY in the same group
    boolean union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return false;              // already connected → this edge is redundant
        if (size[ra] < size[rb]) { int t = ra; ra = rb; rb = t; }   // make ra the bigger root
        parent[rb] = ra;                         // hang the smaller tree under the bigger
        size[ra] += size[rb];
        count--;                                 // two groups became one
        return true;
    }

    boolean connected(int a, int b) { return find(a) == find(b); }
    int componentCount() { return count; }
}
```

**The return value of `union` is the key.** `false` = "already connected" = this edge closes a **cycle**. That answers cycle detection, redundant edges, counting merges, and Kruskal.

**Non-integer nodes:** grid cell → `r * n + c`. Strings → map them to ints with a `HashMap`.

**Time:** O(α(n)) per op, O(E·α(n)) for E edges. **Space:** O(n)

### When to use it

- Edges added over time with queries in between (DSU's special strength)
- Many connectivity queries on a fixed graph
- Counting components (start `count = n`, each successful union does `count--`)
- Cycle detection in an **undirected** graph (not directed: use Kahn's or DFS 3-color)
- Kruskal's MST

**Not for:** distances or shortest paths, or deleting edges.

### LC 547 — Number of Provinces (Medium)

> `isConnected[i][j] == 1` means cities `i` and `j` are directly connected. Count the provinces (connected groups).

**Idea:** union every connected pair. Each **successful** union merges two groups, so the group count drops by 1.

```java
public int findCircleNum(int[][] isConnected) {
    int n = isConnected.length;
    UnionFind uf = new UnionFind(n);
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)           // upper triangle only: the matrix is symmetric
            if (isConnected[i][j] == 1) uf.union(i, j);
    return uf.componentCount();                   // number of groups left
}
```

**Time:** O(n² · α(n)) **Space:** O(n)

### LC 684 — Redundant Connection (Medium)

> A tree with `n` nodes got one extra edge. Return the edge that can be removed (the last one in the input if several work).

**Idea:** go through the edges in order. The first edge whose `union` returns `false` joins two nodes that are already connected, so it closes the cycle.

```java
public int[] findRedundantConnection(int[][] edges) {
    UnionFind uf = new UnionFind(edges.length + 1);   // nodes are 1-indexed → size n+1
    for (int[] e : edges)
        if (!uf.union(e[0], e[1])) return e;          // already connected → this edge makes the cycle
    return new int[0];
}
```

**Time:** O(E · α(n)) **Space:** O(n)

### LC 947 — Most Stones Removed with Same Row or Column (Medium)

> Stones are at integer coordinates. You can remove a stone if another stone **still on the board** shares its row or column. Max stones you can remove?

**Answer = `n - (number of connected groups)`**, where two stones are connected if they share a row **or** a column (transitively).

**Why:** in a group of `k` stones, remove them leaf-first (leaves of a spanning tree). Each removal is legal because its parent still stands. You can remove `k - 1` and one stone stays.

**Approach A — stones are the nodes** (easy to explain):

```java
public int removeStones(int[][] stones) {
    int n = stones.length;
    UnionFind uf = new UnionFind(n);
    int components = n;                                  // start: every stone alone
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)
            if (stones[i][0] == stones[j][0]             // same row
             || stones[i][1] == stones[j][1])            // OR same column
                if (uf.union(i, j)) components--;        // only a REAL merge counts
    return n - components;
}
```

**Time:** O(n² · α(n))

**Approach B — rows and columns are the nodes** (faster). Each **stone is an edge** joining its row-node and its column-node. Stones in the same row then share a row-node automatically, so one union per stone is enough.

```java
public int removeStones(int[][] stones) {
    final int OFFSET = 10001;                    // coordinates ≤ 10^4; keeps row ids and column ids apart
    UnionFind uf = new UnionFind(2 * OFFSET);    // ids 0..10000 = rows, 10001..20001 = columns

    for (int[] s : stones)
        uf.union(s[0], s[1] + OFFSET);           // connect this stone's row to its column

    Set<Integer> roots = new HashSet<>();
    for (int[] s : stones)
        roots.add(uf.find(s[0]));                // loop over STONES, not all ids (empty rows would count)
    return stones.length - roots.size();         // stones − groups
}
```

**Time:** O(n · α(n)). If coordinates have no bound, use a `HashMap`-based DSU and use `~c` (bitwise NOT, always negative) as the column id so it never collides with a row id.

**Pattern: "items linked by a shared attribute"** (same row, same email, same prime factor). Don't compare items pairwise (O(n²)). **Index by the attribute.** Same family: LC 721 Accounts Merge, LC 952 Largest Component by Common Factor, LC 128 Longest Consecutive Sequence.

### LC 3532 — Path Existence Queries in a Graph I (Medium)

> `nums` is **sorted**. An edge exists between any `i, j` with `|nums[i] - nums[j]| <= maxDiff`. For each query `[u, v]`: is there a path?

**Why the obvious way fails:** up to O(n²) edges, and BFS per query is too slow.

**Key insight — "sorted" matters.** If `nums[j] - nums[i] <= maxDiff`, then every adjacent gap between them is also ≤ `maxDiff`. So `i → i+1 → … → j` already works, and **long edges are redundant**. Only the `n-1` adjacent edges matter. A gap bigger than `maxDiff` is a **wall**.

```
nums = [2, 5, 6, 8]   maxDiff = 2
gaps =    3  1  2
          ↑ wall (3 > 2)
comp = [0, 1, 1, 1]
query [1,3] → same comp → true      query [0,2] → different → false
```

```java
// DSU version: union neighbors if the gap fits
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

Groups are always **contiguous intervals**, so you can drop DSU and just **label** each index:

```java
// Simpler: a running counter labels each component in one pass
public boolean[] pathExistenceQueries(int n, int[] nums, int maxDiff, int[][] queries) {
    int[] comp = new int[n];                              // comp[i] = component id of index i
    for (int i = 1; i < n; i++)                           // a wall starts a new component
        comp[i] = comp[i - 1] + (nums[i] - nums[i - 1] <= maxDiff ? 0 : 1);

    boolean[] ans = new boolean[queries.length];
    for (int i = 0; i < queries.length; i++)
        ans[i] = comp[queries[i][0]] == comp[queries[i][1]];   // same label = connected
    return ans;
}
```

**Lesson:** DSU merges arbitrary sets. If only **neighbors** ever merge, a simple counter does the job. Always ask whether the input's structure makes DSU unnecessary.
**Time:** O(n + q) **Space:** O(n)

### Bottleneck paths with DSU — LC 778 & LC 1102 (the "Kruskal engine")

Both look like shortest path but are really **connectivity with a threshold**: *"keep only cells that beat `t`. Are start and end connected?"* Connectivity only grows as you add cells, so **add cells in sorted order and stop the moment start meets end**. That value is the answer. No binary search needed.

#### LC 778 — Swim in Rising Water (DSU)

Add cells **low → high**. The highest cell added when start meets end is the answer.

```java
public int swimInWater(int[][] grid) {
    int n = grid.length;
    UnionFind uf = new UnionFind(n * n);                // cell (r,c) → index r*n + c
    int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};

    Integer[] order = new Integer[n * n];                // all cell indexes...
    for (int i = 0; i < n * n; i++) order[i] = i;
    Arrays.sort(order, (a, b) -> grid[a / n][a % n] - grid[b / n][b % n]);   // ...lowest elevation first

    boolean[] active = new boolean[n * n];               // cells already "under water"
    int ans = 0;
    for (int idx : order) {
        int r = idx / n, c = idx % n;
        ans = Math.max(ans, grid[r][c]);                 // water level so far
        active[idx] = true;
        for (int[] d : DIRS) {
            int nr = r + d[0], nc = c + d[1];
            if (nr < 0 || nr >= n || nc < 0 || nc >= n) continue;
            if (active[nr * n + nc]) uf.union(idx, nr * n + nc);   // join with active neighbors
        }
        if (uf.connected(0, n * n - 1)) return ans;      // start meets end → done
    }
    return ans;
}
```

#### LC 1102 — Path With Maximum Minimum Value

The same code with **three flips**:

| | Swim (minimax) | Max-Min (maximin) |
|---|---|---|
| Sort order | **ascending** | **descending** |
| Running answer | `Math.max` | `Math.min` |
| Finds | smallest possible max | largest possible min |

```java
// only these lines change
Arrays.sort(order, (a, b) -> grid[b / n][b % n] - grid[a / n][a % n]);   // highest first
int ans = Integer.MAX_VALUE;
ans = Math.min(ans, grid[r][c]);                                          // weakest cell added so far
```

**Three solutions to name:** Dijkstra with `max`/`min` (Section 7.2), DSU in sorted order (here), binary search on the threshold + BFS.
**Time:** O(mn log mn) **Space:** O(mn)

---

## 9. MST (Minimum Spanning Tree)

### Idea

An MST connects **all** vertices with the **cheapest** set of edges: exactly `V-1` edges, no cycles.

**Why the greedy algorithms work (cut property):** split the vertices into any two groups. The cheapest edge crossing the split belongs to some MST.

Not shortest path. MST is "connect everything cheaply", not "get from A to B cheaply".

### Kruskal — sort edges, add if no cycle

It's just DSU. `union` returning `false` is the cycle check.

```java
Arrays.sort(edges, (a, b) -> a[2] - b[2]);       // edges[i] = {u, v, weight}, cheapest first
UnionFind uf = new UnionFind(n);
int cost = 0, used = 0;
for (int[] e : edges)
    if (uf.union(e[0], e[1])) {                  // false → would form a cycle → skip
        cost += e[2];
        if (++used == n - 1) break;              // a tree has exactly n-1 edges
    }
return used == n - 1 ? cost : -1;                // fewer than n-1 → graph is disconnected
```

### Prim — grow a tree from one vertex

**Array version, O(V²), no heap** (best for dense graphs):

```java
int[] minDist = new int[n];                       // cheapest edge connecting each vertex to the tree
boolean[] inTree = new boolean[n];
Arrays.fill(minDist, Integer.MAX_VALUE);
minDist[0] = 0;                                   // start from vertex 0
int cost = 0;
for (int iter = 0; iter < n; iter++) {
    int u = -1;
    for (int v = 0; v < n; v++)                   // pick the cheapest vertex still outside the tree
        if (!inTree[v] && (u == -1 || minDist[v] < minDist[u])) u = v;
    inTree[u] = true;
    cost += minDist[u];
    for (int v = 0; v < n; v++)                   // update the cheapest connection of the others
        if (!inTree[v]) minDist[v] = Math.min(minDist[v], weight(u, v));
}
```

The heap version is like Dijkstra, but the heap key is the **edge weight**, not the distance from the source. That one word is the only difference between Prim and Dijkstra.

### Kruskal vs Prim: density decides

| | Time | Best when |
|---|---|---|
| Kruskal (sort + DSU) | O(E log E) | **sparse** graph, or edges given as a list |
| Prim + heap | O(E log V) | sparse, adjacency list |
| Prim + array | **O(V²)** | **dense** graph, e.g. complete graph |

Compare `E log V` with `V²`. On a complete graph `E ≈ V²`, so Kruskal costs `V² log V` and has to sort ~500k edges, while Prim's array version is `V²` with nothing to sort (measured at n = 1000: Kruskal ≈ 197 ms, Prim ≈ 3 ms).

**But Kruskal's *pattern* matters more:** "sort by weight, union step by step, stop at a condition" also solves offline-query problems and LC 1489.

### LC 1584 — Min Cost to Connect All Points (Medium)

> Points on a plane. Connect them all with the smallest total Manhattan distance.

**Recognize:** "connect everything at min cost" = MST. No edge list is given: every pair is an edge (a **complete graph**, ~500k edges at n = 1000).

**Kruskal version** (easy, passes):

```java
public int minCostConnectPoints(int[][] points) {
    int n = points.length;
    UnionFind uf = new UnionFind(n);

    int[][] edges = new int[n * (n - 1) / 2][3];          // {weight, i, j} for every pair
    int k = 0;
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)
            edges[k++] = new int[]{
                Math.abs(points[i][0] - points[j][0]) + Math.abs(points[i][1] - points[j][1]), i, j};
    Arrays.sort(edges, (a, b) -> a[0] - b[0]);            // cheapest first

    int cost = 0, used = 0;
    for (int[] e : edges)
        if (uf.union(e[1], e[2])) {                       // false → cycle → skip
            cost += e[0];
            if (++used == n - 1) break;                   // tree complete
        }
    return cost;
}
```

**Time:** O(n² log n) **Space:** O(n²)

**Prim version** (better for a complete graph):

```java
public int minCostConnectPoints(int[][] points) {
    int n = points.length;
    int[] minDist = new int[n];                            // cheapest known link from the tree to each point
    boolean[] inTree = new boolean[n];
    Arrays.fill(minDist, Integer.MAX_VALUE);
    minDist[0] = 0;
    int cost = 0;

    for (int iter = 0; iter < n; iter++) {
        int u = -1;
        for (int v = 0; v < n; v++)                        // cheapest point still outside the tree
            if (!inTree[v] && (u == -1 || minDist[v] < minDist[u])) u = v;
        inTree[u] = true;
        cost += minDist[u];                                // pay for the edge that brings u in

        for (int v = 0; v < n; v++) {                      // maybe u gives a cheaper link to others
            if (inTree[v]) continue;
            int d = Math.abs(points[u][0] - points[v][0])
                  + Math.abs(points[u][1] - points[v][1]); // edge weight computed on the fly
            if (d < minDist[v]) minDist[v] = d;
        }
    }
    return cost;
}
```

**Time:** O(n²) **Space:** O(n) (no edge list is ever built).

**What to say in an interview:** *"This is an MST. Kruskal is natural, but every pair is an edge, so that's ~500k edges to build and sort. The graph is complete, so Prim with a `minDist` array is O(n²) with no heap and no sorting. I'll use that."*

### LC 1489 — Find Critical and Pseudo-Critical Edges in MST (Hard)

> **Critical** edge: removing it makes the MST heavier (or disconnects the graph), so it's in **every** MST. **Pseudo-critical:** in **some** MST but not all. Return both lists.

**Why Kruskal:** you need to test edges one by one, which Prim can't do. Two Kruskal runs per edge:

| Test | How | Meaning |
|---|---|---|
| Critical? | Run Kruskal with the edge **banned** | Weight goes up (or can't span) ⇒ critical |
| Pseudo-critical? | Run Kruskal with the edge **forced in first** | Weight equals the base MST ⇒ in some MST |

**Test critical first.** A critical edge also passes the "forced" test, so checking pseudo-critical first would misclassify it.

```java
public List<List<Integer>> findCriticalAndPseudoCriticalEdges(int n, int[][] edges) {
    int m = edges.length;
    int[][] e = new int[m][4];
    for (int i = 0; i < m; i++)
        e[i] = new int[]{edges[i][0], edges[i][1], edges[i][2], i};   // keep the ORIGINAL index in slot 3
    Arrays.sort(e, (a, b) -> a[2] - b[2]);                            // sort by weight

    int base = mstWeight(n, e, -1, -1);                // normal MST weight (nothing banned or forced)
    List<Integer> critical = new ArrayList<>(), pseudo = new ArrayList<>();

    for (int i = 0; i < m; i++) {
        if (mstWeight(n, e, i, -1) > base)             // banning it makes the MST worse → critical
            critical.add(e[i][3]);
        else if (mstWeight(n, e, -1, i) == base)       // forcing it still gives the best weight → pseudo
            pseudo.add(e[i][3]);
    }
    return List.of(critical, pseudo);
}

// Kruskal with one edge banned (skip) or forced in first (force). Use -1 for "none".
// Returns MAX_VALUE if the graph can't be spanned.
private int mstWeight(int n, int[][] e, int skip, int force) {
    UnionFind uf = new UnionFind(n);                   // fresh DSU for every run
    int weight = 0, used = 0;
    if (force >= 0 && uf.union(e[force][0], e[force][1])) {   // put the forced edge in first
        weight += e[force][2];
        used++;
    }
    for (int i = 0; i < e.length; i++) {
        if (i == skip) continue;                       // banned edge
        if (uf.union(e[i][0], e[i][1])) {              // the forced edge is skipped naturally (already joined)
            weight += e[i][2];
            used++;
        }
    }
    return used == n - 1 ? weight : Integer.MAX_VALUE; // MAX_VALUE makes "> base" catch disconnection too
}
```

**Time:** O(E² · α(V)) (2E Kruskal runs). **Space:** O(V + E)

---
## 10. Other Graph Concepts

### 10.1 Functional graphs (every node has at most 1 outgoing edge)

**Idea:** if each node points to **at most one** node, walking forward is **deterministic** (no branching). No queue, no reverse graph, no recursion: just a `while` loop.

**How it shows up** (the input rarely says "graph"):
- `edges[i] = j` (or `-1` for none)
- `nums` is a **permutation** of `0..n-1` → pure cycles, no tails
- "from index `i` jump to `i + nums[i]`"
- any "follow the next pointer" chain

**Shape:** every piece is a "rho": a tail leading into exactly one cycle.

```
tail:  7 → 8 → 3 ─┐
                  ↓
       cycle:  3 → 4 → 5 → 3
```

**Walk-and-timestamp template:** stamp each node with *when* you visited it. One comparison tells you if you found a **new** cycle:

```java
int[] visitedAt = new int[n];         // 0 = unvisited, otherwise the time it was stamped
int timer = 1;

for (int i = 0; i < n; i++) {
    if (visitedAt[i] != 0) continue;
    int start = timer;                // the time THIS walk began
    int u = i;
    while (u != -1 && visitedAt[u] == 0) {
        visitedAt[u] = timer++;
        u = next(u);                  // the one outgoing edge
    }
    // the walk stopped: dead end (-1), or we ran into a stamped node
    if (u != -1 && visitedAt[u] >= start) {          // stamped during THIS walk → new cycle
        int cycleLength = timer - visitedAt[u];
        // ... use it
    }
}
```

`visitedAt[u] >= start` is the whole idea. If `u` was stamped by an **earlier** walk, you just joined old territory (no new cycle). If it was stamped by **this** walk, you closed a loop.

**Never recurse on a walk.** The depth can be 10^5 and Java has no tail-call optimization, so it overflows. Use a loop.

**Time:** O(n) **Space:** O(n)

#### LC 2360 — Longest Cycle in a Graph (Hard)

> `edges[i]` is the one node `i` points to, or -1. Return the length of the longest cycle, or -1.

```java
public int longestCycle(int[] edges) {
    int n = edges.length;
    int[] visitedAt = new int[n];
    int timer = 1, longest = -1;                       // -1 covers the "no cycle" case

    for (int i = 0; i < n; i++) {
        if (visitedAt[i] != 0) continue;               // already part of an earlier walk
        int start = timer, u = i;
        while (u != -1 && visitedAt[u] == 0) {
            visitedAt[u] = timer++;                    // stamp with the current time
            u = edges[u];                              // follow the single edge
        }
        if (u != -1 && visitedAt[u] >= start)          // closed a loop inside THIS walk
            longest = Math.max(longest, timer - visitedAt[u]);   // length = nodes stamped since u
    }
    return longest;
}
```

Alternative: peel with Kahn's (remove every node that reaches indegree 0), then walk what's left. Same O(n); the timestamp one is a single pass.
**Time:** O(n) **Space:** O(n)

#### LC 565 — Array Nesting (Medium)

> `nums` is a **permutation** of `0..n-1`. `s[k] = {nums[k], nums[nums[k]], ...}` until a repeat. Return the longest set.

**Idea:** a permutation = every node has indegree 1 and outdegree 1 → the graph is a set of **separate cycles**, no tails. `s[k]` is just the cycle containing `k`. Find the longest cycle. A plain `boolean[]` is enough.

```java
public int arrayNesting(int[] nums) {
    int n = nums.length;
    boolean[] visited = new boolean[n];
    int longest = 0;
    for (int i = 0; i < n; i++) {
        if (visited[i]) continue;               // this cycle was already measured
        int len = 0, cur = i;
        while (!visited[cur]) {                 // walk the whole cycle
            visited[cur] = true;
            len++;
            cur = nums[cur];
        }
        longest = Math.max(longest, len);
    }
    return longest;
}
```

**Why O(n), not O(n²):** each element is visited once across all starts. A cycle already walked is skipped.
**Time:** O(n) **Space:** O(n)

#### LC 457 — Circular Array Loop (Medium)

> `nums[i]` is a jump length (circular). Is there a cycle of length > 1 where all jumps go the **same direction**?

**Idea:** fold the two extra rules into the `next()` function (return `-1` for an invalid step), then use **Floyd's tortoise and hare** (slow moves 1, fast moves 2; they meet inside a cycle). Failed paths are zeroed so each cell is walked once.

```java
// next index from i, or -1 if the step is invalid
private int next(int[] nums, int i, int n) {
    int j = ((i + nums[i]) % n + n) % n;               // fix Java's negative modulo
    if (j == i) return -1;                              // self-loop = length 1, not allowed
    if ((long) nums[j] * nums[i] < 0) return -1;        // direction changed
    return j;
}

public boolean circularArrayLoop(int[] nums) {
    int n = nums.length;
    for (int i = 0; i < n; i++) {
        if (nums[i] == 0) continue;                     // already proven to lead nowhere
        int slow = i, fast = i;
        while (true) {                                  // Floyd's: slow = 1 step, fast = 2 steps
            slow = next(nums, slow, n);
            if (slow == -1) break;
            fast = next(nums, fast, n);
            if (fast != -1) fast = next(nums, fast, n);
            if (fast == -1) break;
            if (slow == fast) return true;              // they met → valid cycle
        }
        // this start failed: mark the whole same-direction path as dead (zero it)
        int j = i, sign = nums[i];
        while (nums[j] != 0 && (long) nums[j] * sign > 0) {
            int nj = ((j + nums[j]) % n + n) % n;
            nums[j] = 0;
            j = nj;
        }
    }
    return false;
}
```

**Three traps:** negative modulo needs `((x % n) + n) % n`; the self-loop check enforces length > 1; zeroing failed paths keeps it O(n).
**Time:** O(n) **Space:** O(1)

**Related:** Linked List Cycle II (LC 142) and Find the Duplicate Number (LC 287) are also functional graphs solved with Floyd's in O(1) space.

### 10.2 Remodel the graph: 7 moves for hard problems

Most hard graph problems are a normal template plus one of these:

1. **Reverse the edges.** Many-to-many becomes few-to-many. *LC 417*: DFS from the ocean borders inward.
2. **Add an imaginary super-source.** Multi-source BFS is exactly this: one node joined to all sources by 0-cost edges.
3. **Redefine what a node is.** A node can be a 4-digit string (LC 752), or `(position, visitedSet)`, or a row/column index (LC 947).
4. **Swap the operator.** `+` → `max` / `min` / `×` in Dijkstra (LC 778, 1102).
5. **Reverse time.** Deletions become additions, which DSU can handle (LC 803 Bricks Falling When Hit).
6. **Sort to create an order.** In LC 329 the cell values *are* the topological order. Offline queries: sort by threshold so edges are only added.
7. **Binary search on the answer.** "Minimize the maximum" → *for a fixed X, is it feasible?* (a plain BFS/DFS check).

> **"Minimize the maximum" or "maximize the minimum" always has two solutions:** (a) binary search + reachability, (b) Dijkstra with swapped operator (or DSU in sorted order). Naming both is strong signal.

### 10.3 Cycle detection: which tool?

| Graph | Tool |
|---|---|
| Undirected | DSU (`union` returns false), or DFS with "visited neighbor that isn't the parent" |
| Directed | Kahn's (`order.size() < n`) or DFS 3-color (edge to a node on the current path) |
| Out-degree ≤ 1 | Timestamp walk, or Floyd's tortoise and hare for O(1) space |
| Negative cycle (weighted) | Bellman-Ford: a V-th round still improves something |

### 10.4 Names to know (usually not coded in interviews)

- **Bridges, articulation points, SCCs (Tarjan/Kosaraju):** need DFS-tree structure (low-link values). BFS can't do these.
- **Floyd-Warshall:** all-pairs shortest path in O(V³). Simple 3 nested loops. Use for small dense graphs.
- **Dial's algorithm:** shortest path with weights 0..k, a bucket version of 0-1 BFS.

---

## 11. Tips & Tricks

### General

- **Name nodes and edges out loud** before coding. It buys thinking time and reads as senior.
- **The verb of the question picks the algorithm,** not the input shape (Section 2).
- **Take the cheapest algorithm that fits** the edge weights: BFS → 0-1 BFS → Dijkstra → Bellman-Ford.
- **Say both approaches** when two exist (Dijkstra and Bellman-Ford for LC 787; Dijkstra, DSU and binary search for LC 778).
- **Ask what the input guarantees you haven't used yet** (sorted? permutation? out-degree ≤ 1?). That is often the whole trick (LC 3532, LC 565).
- **Write the simple solution first,** get it correct, then offer the optimization (LC 947 approach A then B; LC 1584 Kruskal then Prim).
- **Large inputs:** use `long` for summed costs if weights are big. Don't recurse on 10^5-deep structures.

### BFS

- **Mark visited when you enqueue,** not when you dequeue.
- **Freeze `queue.size()`** at the start of each ring to count distances.
- **Multi-source:** put all sources in the queue first. Don't run BFS per source (unless you need the *sum* over sources, like LC 317).
- **Bidirectional BFS** when start and target are both known and branching is big. Always expand the smaller side.
- **Read the distance definition:** edges or nodes? (LC 127 counts words; LC 1091 counts cells.)

### DFS

- **Guards at the top of the function** (bounds, then visited/wall) keep grid DFS short.
- **Deep graph?** Use an iterative stack or BFS.
- **DFS + memo on a DAG** is just top-down DP (LC 329). The DAG property replaces the visited set.
- **Reverse the direction** to turn many queries into a few sweeps (LC 417).

### Topological sort

- **Default to Kahn's.** It's iterative and cycle detection is free (`order.size() < n`).
- **Get the edge direction right.** `[a, b]` meaning "b before a" gives edge `b → a`. Write one example edge.
- **Queue size > 1 ⇔ order not unique.**
- **`PriorityQueue` ⇒ lexicographically smallest order.**
- **Register all nodes first,** so isolated nodes still appear.
- **"u is OK iff all its successors are OK"** → reverse the graph, then peel (LC 802).
- **Answer must be sorted but the peel order isn't?** Mark a `boolean[]` and scan `0..n-1`.
- **Undirected tree peeling:** degree-1 leaves, stop at 2 nodes left (LC 310).
- **Topo order turns a DAG into a DP tape:** longest path and path counting become one pass.

### Shortest path

- **Always write the lazy-deletion guard** `if (d > dist[u]) continue;` in Dijkstra.
- **Exit early on pop, never on push.**
- **Negative weight anywhere ⇒ not Dijkstra.** Say "Bellman-Ford".
- **Two knobs:** side constraint → bigger state; max/min/product → swap the operator.
- **Bellman-Ford with a hop limit:** copy the array each round.
- **Guard `dist[u] == MAX_VALUE`** before adding a weight, or you overflow.
- **1-indexed nodes** → size arrays `n + 1`.

### DSU

- **Always return a boolean from `union`.** `false` = cycle / redundant edge.
- **Write `find` iteratively** (path halving, 2 lines).
- **1-indexed nodes** → size `n + 1` (Redundant Connection).
- **Undirected graphs only** for cycle detection.
- **Deletions ⇒ process in reverse.**
- **Grid cells:** flatten with `r * n + c`.
- **`n - components`** is the answer when you keep one item per group (LC 947).
- **Shared attribute?** Index by the attribute instead of comparing pairs.
- **Ask if the structure makes DSU unnecessary** (sorted input, only neighbors merge).

### MST

- **Kruskal = sort + DSU,** skip the `false` unions.
- **Count edges used:** `used == n - 1`, otherwise disconnected.
- **Points / no edge list ⇒ complete graph ⇒ Prim O(V²).** Compute weights on the fly.
- **Prim vs Dijkstra:** the heap key is the *edge weight* (Prim) vs the *distance from source* (Dijkstra).
- **"In every / in some MST?"** ⇒ ban-it and force-it probes; test critical first.

### If you get stuck in an interview

Silence is the worst outcome. A wrong idea said out loud is fine.

1. **Say where you are.** *"I'm confident this is shortest path, but not sure yet if the K limit goes into the state."*
2. **Solve an easier version.** Drop a constraint, solve it, add it back. Try `n = 2`.
3. **State the brute force and its complexity,** then improve it. Never open with "I don't know".
4. **Trace an example by hand.** Ask *"why is the answer 4 and not 3?"*. The reason is usually the insight.
5. **Run the 7 remodeling moves** (Section 10.2) out loud.
6. **Ask a narrow question** by about the 5-minute mark. Show your thinking first: *"I see two directions: A for X, B for Y. I lean toward B because ... does that sound right?"*

**If you chose a wrong approach,** say so and switch: *"Plain BFS isn't enough because revisiting with fewer removals can be better, so I'll use `(cell, used)` as the state."* Catching your own mistake is a good signal.

**Rough timing for a 45-minute screen:** ~5 min clarify + brute force, ~10 min pick approach + complexity, ~30 min working code, ~35 min dry run, rest for follow-ups. Stuck 15 minutes without an approach → ask for a hint.

### Questions to ask the interviewer

Ask 2-3 that could change your code:

- Directed or undirected?
- Can weights be negative?
- How large are `n` and the number of queries? (the most useful one)
- Can the graph be disconnected? Self-loops or duplicate edges?
- Are nodes `0..n-1` or arbitrary labels?
- Any tie-break rule (lexicographically smallest)?
- Is the input sorted or otherwise guaranteed?

When you get a hint, use it visibly and say what it unlocked.
