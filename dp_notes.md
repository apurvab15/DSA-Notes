# Dynamic Programming — Study Notes

> **Convention:** prefix / low-to-high. `dp[i][j]` = answer for the *first `i`* and *first `j`* elements; fill low → high; answer at `dp[m][n]`. Base case lives in the empty-prefix row/column.

---

# 1D Sequence DP

`dp[i]` depends on a **fixed, small look-back** (usually `dp[i-1]`, `dp[i-2]`). Backward recurrence → fill low→high. Because only the last 1–2 cells are ever read, these **space-optimize to a couple of rolling variables** (O(1) space) — the defining trait of this family vs. the "keep the whole array" families.

**Knob checklist:** *type?* (count `+` / optimize min-max / feasible OR) sets the operator and base case · *look-back depth?* (2 → two rolling vars, 3 → three).

---

## Climbing Stairs

**Description:** You climb `n` stairs, taking 1 or 2 steps at a time. Return the number of distinct ways to reach the top.

**Example:** `n = 3` → `3`. The ways: `1+1+1`, `1+2`, `2+1`.

**Intuition:** `dp[i]` = number of ways to reach step `i`. You arrive at `i` either from `i-1` (one step) or `i-2` (two steps), so the ways **sum**: `dp[i] = dp[i-1] + dp[i-2]`. It's a *counting* problem, so combine with `+`. Base cases: `dp[0] = 1` (one way to be at the start — do nothing), `dp[1] = 1`. It's Fibonacci.

**Concrete — `dp[3]`:** ways to reach step 3 = (ways to reach step 2, then take 1 step) + (ways to reach step 1, then take 2 steps) = `dp[2] + dp[1] = 2 + 1 = 3`.

**Signature to remember:** counting → `+`, base `dp[0]=dp[1]=1`; look-back 2 → two rolling vars.

**Type / knobs:** counting (`+`) · look-back 2.

**Time:** O(n) (every rung).
**Space:** memo/tabulation O(n) · space-optimized O(1).

### Recurrence ‖ Memo

```java
// ── RECURRENCE ── (exponential)                  // ── MEMO ── O(n)
int climbStairs(int n) {                         Integer[] memo;
    return dfs(n);                               int climbStairs(int n) {
}                                                    memo = new Integer[n + 1];
                                                     return dfs(n);
int dfs(int i) {                                 }
    if (i <= 1) return 1;   // base              int dfs(int i) {
    return dfs(i-1) + dfs(i-2);                      if (i <= 1) return 1;
}                                                    if (memo[i] != null) return memo[i];  // hit
                                                     return memo[i] = dfs(i-1) + dfs(i-2);
                                                 }
```

### Tabulation ‖ Space-optimized

```java
// ── TABULATION ──                                // ── SPACE-OPTIMIZED (two vars) ──
int climbStairs(int n) {                         int climbStairs(int n) {
    int[] dp = new int[n + 1];                       int prev2 = 1, prev1 = 1;  // dp[0], dp[1]
    dp[0] = 1; dp[1] = 1;                            for (int i = 2; i <= n; i++) {
    for (int i = 2; i <= n; i++)                         int curr = prev1 + prev2;
        dp[i] = dp[i-1] + dp[i-2];                       prev2 = prev1;   // slide (oldest first)
    return dp[n];                                        prev1 = curr;
}                                                    }
                                                     return prev1;
                                                 }
```

**Tips:**
- Base case is `1` (counting: one way to be at the start), not 0.
- Space-opt: two rolling vars, slide oldest-first (`prev2 = prev1` before `prev1 = curr`).
- `n <= 1` handled by the base returning 1 directly.

**Trick / clever variant:** it's literally Fibonacci — same recurrence, base `1,1`.

---

## Min Cost Climbing Stairs

**Description:** `cost[i]` is the price to step off stair `i`. You start at stair 0 or 1 (free), and each move goes up 1 or 2 stairs. Return the min cost to reach the top (past the last stair).

**Example:** `cost = [10,15,20]` → `15`. Start at stair 1 (free), pay 15 to step off it, reach the top.

**Intuition:** `dp[i]` = min cost to *reach* step `i`. You pay for the step you **came from**, so reaching `i` costs `min(dp[i-1] + cost[i-1], dp[i-2] + cost[i-2])`. Optimizing → `min`. Base cases: `dp[0] = dp[1] = 0` — **reaching** steps 0 and 1 is free (you pay only when you *leave* a step). Answer is `dp[n]` (the top, past the array).

**Concrete — `dp[3]` for `cost=[10,15,20]`** (`dp[0]=dp[1]=0, dp[2]=10`): came from step 2 → `dp[2]+cost[2] = 10+20 = 30`; came from step 1 → `dp[1]+cost[1] = 0+15 = 15`. Min → `dp[3] = 15`.

**Signature to remember:** pay for the step you *came from* (`cost[i-1]`/`cost[i-2]`); reaching 0/1 is free (`dp=0`); answer at `dp[n]`.

**Type / knobs:** optimizing (`min`) · look-back 2 · you pay on *leaving* a step (base cases are 0, not the cost).

**Time:** O(n).
**Space:** memo/tabulation O(n) · space-optimized O(1).

### Recurrence ‖ Memo

```java
// ── RECURRENCE ──                                // ── MEMO ── O(n)
int minCost(int[] cost) {                        Integer[] memo;
    return dfs(cost.length, cost);               int minCost(int[] cost) {
}                                                    memo = new Integer[cost.length + 1];
                                                     return dfs(cost.length, cost);
int dfs(int i, int[] cost) {                     }
    if (i <= 1) return 0;   // reaching 0/1 free  int dfs(int i, int[] cost) {
    return Math.min(                                 if (i <= 1) return 0;
        dfs(i-1, cost) + cost[i-1],                  if (memo[i] != null) return memo[i];  // hit
        dfs(i-2, cost) + cost[i-2]);                 return memo[i] = Math.min(
}                                                        dfs(i-1, cost) + cost[i-1],
                                                         dfs(i-2, cost) + cost[i-2]);
                                                 }
```

### Tabulation ‖ Space-optimized

```java
// ── TABULATION ──                                // ── SPACE-OPTIMIZED (two vars) ──
int minCost(int[] cost) {                        int minCost(int[] cost) {
    int n = cost.length;                             int n = cost.length;
    int[] dp = new int[n + 1];                       int prev2 = 0, prev1 = 0;  // dp[0], dp[1]
    dp[0] = 0; dp[1] = 0;   // reaching 0/1 free     for (int i = 2; i <= n; i++) {
    for (int i = 2; i <= n; i++)                         int curr = Math.min(prev1 + cost[i-1],
        dp[i] = Math.min(dp[i-1] + cost[i-1],                               prev2 + cost[i-2]);
                         dp[i-2] + cost[i-2]);           prev2 = prev1;
    return dp[n];                                        prev1 = curr;
}                                                    }
                                                     return prev1;   // dp[n]
                                                 }
```

**Tips:**
- **You pay to LEAVE a step, so reaching 0/1 is free** (`dp[0]=dp[1]=0`), and the cost index is `cost[i-1]`/`cost[i-2]` (the step you came from), NOT `cost[i]`. This is the #1 confusion (vs. Climbing Stairs' base of 1).
- Answer is `dp[n]` (the top, one past the last stair) — array sized `n+1`.
- Seeds for the space-opt are `0, 0` (not `cost[0], cost[1]`).

**Trick / clever variant:** none.

---

## House Robber

**Description:** Houses in a row, `nums[i]` money in each. You can't rob two **adjacent** houses. Return the max money.

**Example:** `nums = [2,7,9,3,1]` → `12` (rob houses 0, 2, 4 → 2+9+1).

**Intuition:** `dp[i]` = max money robbable from houses `0..i`. At house `i`: **rob it** (take `nums[i]` + best up to `i-2`, since `i-1` is adjacent) or **skip it** (carry `dp[i-1]`). Take the better → optimizing `max`: `dp[i] = max(nums[i] + dp[i-2], dp[i-1])`. Base cases: `dp[0] = nums[0]`; `dp[1] = max(nums[0], nums[1])`.

**Concrete — `dp[2]` for `nums=[2,7,9,...]`** (`dp[0]=2, dp[1]=max(2,7)=7`): rob house 2 → `nums[2] + dp[0] = 9 + 2 = 11`; skip → `dp[1] = 7`. Max → `dp[2] = 11`.

**Signature to remember:** rob (`nums[i] + dp[i-2]`) vs skip (`dp[i-1]`), take max; base `dp[1] = max(nums[0], nums[1])`.

**Type / knobs:** optimizing (`max`) · look-back 2 · the choice is rob-vs-skip (rob forces the `i-2` jump due to adjacency).

**Time:** O(n).
**Space:** memo/tabulation O(n) · space-optimized O(1).

### Recurrence ‖ Memo

```java
// ── RECURRENCE ──                                // ── MEMO ── O(n)
int rob(int[] nums) {                            Integer[] memo;
    return dfs(nums.length - 1, nums);           int rob(int[] nums) {
}                                                    memo = new Integer[nums.length];
                                                     return dfs(nums.length - 1, nums);
int dfs(int i, int[] nums) {                     }
    if (i < 0) return 0;                          int dfs(int i, int[] nums) {
    // rob i (skip i-1) vs skip i                     if (i < 0) return 0;
    return Math.max(                                 if (memo[i] != null) return memo[i];  // hit
        nums[i] + dfs(i-2, nums),                    return memo[i] = Math.max(
        dfs(i-1, nums));                                 nums[i] + dfs(i-2, nums, memo),
}                                                        dfs(i-1, nums, memo));
                                                 }
```

### Tabulation ‖ Space-optimized

```java
// ── TABULATION ──                                // ── SPACE-OPTIMIZED (two vars) ──
int rob(int[] nums) {                            int rob(int[] nums) {
    int n = nums.length;                             int n = nums.length;
    if (n == 1) return nums[0];                      if (n == 1) return nums[0];
    int[] dp = new int[n];                           int prev2 = nums[0];              // dp[0]
    dp[0] = nums[0];                                 int prev1 = Math.max(nums[0],     // dp[1]
    dp[1] = Math.max(nums[0], nums[1]);                                  nums[1]);
    for (int i = 2; i < n; i++)                      for (int i = 2; i < n; i++) {
        dp[i] = Math.max(nums[i] + dp[i-2],              int curr = Math.max(nums[i] + prev2,
                         dp[i-1]);                                          prev1);
    return dp[n-1];                                      prev2 = prev1;
}                                                        prev1 = curr;
                                                     }
                                                     return prev1;   // dp[n-1]
                                                 }
```

**Tips:**
- **`dp[1] = max(nums[0], nums[1])`**, not `nums[1]` — with two houses (adjacency forbids both), take the richer. This is the #1 House Robber bug.
- Answer is `dp[n-1]` (the last house), NOT `dp[n]` — there's no "top past the end" like the stair problems.
- `dp[i]` is monotonic (best-through-i ≥ best-through-(i-1)), so `return prev1` alone is correct — no need for `max(prev1, prev2)`.
- Guard `n == 1` (so `dp[1]` doesn't index out of bounds).

**Trick / clever variant:** **House Robber II** (circular street): houses 0 and n-1 are adjacent, so run this twice — once excluding the first house, once excluding the last — and take the max. Reuse a `robRange(nums, start, end)` helper with `0,0` seeds over an exclusive `[start,end)` window.

---

## Decode Ways

**Description:** A message of digits, where `1→A … 26→Z`. Return the number of ways to decode it. A `0` can't stand alone; two digits decode only if they form `10`–`26`.

**Example:** `s = "226"` → `3`: `"2 2 6"` (BBF), `"22 6"` (VF), `"2 26"` (BZ).

**Intuition:** `dp[i]` = number of ways to decode the first `i` characters. Counting → `+`. At position `i`: the single digit `s[i-1]` decodes alone if it's not `'0'` (→ add `dp[i-1]`); the two digits `s[i-2..i-1]` decode together if they form `10`–`26` (→ add `dp[i-2]`). Base case `dp[0] = 1` (empty string = one way). The **semantics** (zeros!) are the whole challenge, not the DP.

**Concrete — `dp[3]` for `s="226"`:** single digit `'6'` valid → add `dp[2]` (= 2 ways); two digits `"26"` (≤26) valid → add `dp[1]` (= 1 way). Total `dp[3] = 2 + 1 = 3`.

**Signature to remember:** counting → `+`; single digit valid (not `'0'`) → `+dp[i-1]`; two digits `10`–`26` → `+dp[i-2]`; base `dp[0]=1`.

**Type / knobs:** counting (`+`) · look-back 2 · **semantics-heavy** (zeros are the trap).

**Time:** O(n).
**Space:** memo/tabulation O(n) · space-optimized O(1).

### Recurrence ‖ Memo

```java
// ── RECURRENCE ── (index i = "from i onward")     // ── MEMO ── O(n)
int numDecodings(String s) {                     Integer[] memo;
    return dfs(0, s);                            int numDecodings(String s) {
}                                                    memo = new Integer[s.length()];
                                                     return dfs(0, s);
int dfs(int i, String s) {                       }
    if (i == s.length()) return 1;   // end       int dfs(int i, String s) {
    if (s.charAt(i) == '0') return 0; // dead         if (i == s.length()) return 1;
                                                     if (s.charAt(i) == '0') return 0;
    int ways = dfs(i+1, s);          // 1 digit       if (memo[i] != null) return memo[i];   // hit
    if (i+1 < s.length() &&                          int ways = dfs(i+1, s, memo);
        Integer.parseInt(                            if (i+1 < s.length() &&
            s.substring(i, i+2)) <= 26)                  Integer.parseInt(s.substring(i,i+2)) <= 26)
        ways += dfs(i+2, s);         // 2 digits          ways += dfs(i+2, s, memo);
    return ways;                                     return memo[i] = ways;
}                                                }
```

### Tabulation ‖ Space-optimized

```java
// ── TABULATION (dp[i] = first i chars) ──         // ── SPACE-OPTIMIZED (two vars) ──
int numDecodings(String s) {                     int numDecodings(String s) {
    int n = s.length();                              int n = s.length();
    int[] dp = new int[n + 1];                       int prev2 = 1;               // dp[0]
    dp[0] = 1;                                       int prev1 = s.charAt(0)=='0' ? 0 : 1; // dp[1]
    dp[1] = s.charAt(0)=='0' ? 0 : 1;               for (int i = 2; i <= n; i++) {
    for (int i = 2; i <= n; i++) {                       int curr = 0;
        if (s.charAt(i-1) != '0')                        if (s.charAt(i-1) != '0') curr += prev1;
            dp[i] += dp[i-1];       // 1 digit           int two = Integer.parseInt(
        int two = Integer.parseInt(                                  s.substring(i-2, i));
                    s.substring(i-2, i));               if (two >= 10 && two <= 26) curr += prev2;
        if (two >= 10 && two <= 26)                      prev2 = prev1;
            dp[i] += dp[i-2];       // 2 digits           prev1 = curr;
    }                                                }
    return dp[n];                                    return prev1;
}                                                }
```

**Tips:**
- **Zeros are the whole difficulty** (semantics, not DP). A `'0'` can't stand alone (dead end); it's only legal as the second digit of `"10"`/`"20"`.
- **Recursion:** check `s.charAt(i)` (the CURRENT position), not `s.charAt(0)`. Base case (`i == length` → 1) before the `'0'` check so `charAt(i)` stays in bounds.
- **Tabulation:** here `dp[i]` = "first i chars" (prefix), so the single digit is `s[i-1]` and the pair is `s[i-2..i-1]`.
- Test set: `"0"`→0, `"06"`→0, `"10"`→1, `"100"`→0, `"120"`→1, `"226"`→3.

**Trick / clever variant:** none — it's a semantics exercise on House Robber's look-back-2 skeleton (counting variant with validity gates instead of rob/skip).

---

# 2D String DP

The shared shape: a table `dp[i][j]` over two strings, filled row by row. Match does one thing (usually the diagonal), mismatch does another (best of the neighbours). LCS is the template; Edit Distance and most string problems are variations on it.

---

## Longest Common Subsequence

**Description:** Given two strings `text1` and `text2`, return the length of their longest common subsequence — characters in the same relative order, not necessarily contiguous. No common subsequence → 0.

**Example:** `text1 = "abcde"`, `text2 = "ace"` → `3`. The LCS is `"ace"` (skip `b` and `d`).

**Intuition:** `dp[i][j]` = length of the LCS of the first `i` chars of `text1` and first `j` chars of `text2`. At each pair of characters: if they **match**, that character is in the LCS — count it and shrink both prefixes (diagonal). If they **don't match**, you can't use both, so drop one character from either string and take the better result. Base case: an empty prefix shares nothing → 0.

**Signature to remember:** match → diagonal `+ 1`; mismatch → `max` of the two orthogonal neighbours (up, left).

**Type / knobs:** optimizing (`max`) · answer at `dp[m][n]`.

**Time:** `O(m·n)` (every rung).
**Space:** recurrence/memo `O(m·n)` · tabulation `O(m·n)` · space-optimized `O(n)` (two rows).

### Recurrence ‖ Memo

```java
// ── RECURRENCE ──                              // ── MEMO ──  (recurrence + cache lines)
int lcs(String a, String b) {                    int lcs(String a, String b) {
    return dfs(a.length(),                           Integer[][] memo =
              b.length(), a, b);                         new Integer[a.length()+1][b.length()+1];
}                                                    return dfs(a.length(), b.length(), a, b, memo);
                                                 }
int dfs(int i, int j,                            int dfs(int i, int j,
        String a, String b) {                            String a, String b, Integer[][] memo) {
    // base: empty prefix -> LCS 0                    // base: empty prefix -> LCS 0
    if (i == 0 || j == 0) return 0;                  if (i == 0 || j == 0) return 0;
                                                     if (memo[i][j] != null) return memo[i][j];   // hit
    if (a.charAt(i-1) == b.charAt(j-1)) {            if (a.charAt(i-1) == b.charAt(j-1)) {
        // match -> take char, shrink both               // match -> take char, shrink both
        return 1 + dfs(i-1, j-1, a, b);                  memo[i][j] = 1 + dfs(i-1, j-1, a, b, memo);
    } else {                                         } else {
        // mismatch -> drop one, take best               // mismatch -> drop one, take best
        return Math.max(dfs(i-1, j, a, b),               memo[i][j] = Math.max(dfs(i-1, j, a, b, memo),
                        dfs(i, j-1, a, b));                                    dfs(i, j-1, a, b, memo));
    }                                                }
}                                                    return memo[i][j];   // store before returning
                                                 }
```

### Tabulation ‖ Space-optimized

```java
// ── TABULATION (2D) ──                          // ── SPACE-OPTIMIZED (two rows) ──
int lcs(String a, String b) {                    int lcs(String a, String b) {
    int m = a.length(), n = b.length();              int m = a.length(), n = b.length();
    int[][] dp = new int[m+1][n+1];                  int[] prev = new int[n+1];   // row i-1
    // row 0 & col 0 = 0 (empty prefix), auto         int[] curr = new int[n+1];   // row i

    for (int i = 1; i <= m; i++) {                   for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {                   for (int j = 1; j <= n; j++) {
            if (a.charAt(i-1)==b.charAt(j-1)) {              if (a.charAt(i-1)==b.charAt(j-1)) {
                dp[i][j] = 1 + dp[i-1][j-1];                     curr[j] = 1 + prev[j-1];   // diagonal
            } else {                                         } else {
                dp[i][j] = Math.max(dp[i-1][j],                  curr[j] = Math.max(prev[j],      // up
                                    dp[i][j-1]);                                    curr[j-1]);  // left
            }                                                }
        }                                                }
    }                                                    int[] t=prev; prev=curr; curr=t;   // slide
    return dp[m][n];                                 }
}                                                    return prev[n];
                                                 }
```

**Cell mapping (2D → two rows):** `dp[i-1][j-1]` → `prev[j-1]` · `dp[i-1][j]` → `prev[j]` · `dp[i][j-1]` → `curr[j-1]`.

**Tips:**
- Prefix off-by-one: compare `charAt(i-1)` / `charAt(j-1)`, since `dp[i][j]` covers the first `i`/`j` chars (last char sits at index `i-1`).
- Table is `[m+1][n+1]` — the extra row/col is the empty-prefix base case (auto-zeros, no manual seeding).
- Space-opt needs **two rows, not one**, because the recurrence reads the **diagonal** (`dp[i-1][j-1]`); a single in-place row would overwrite the diagonal before use. (Doable with one row + a temp holding the diagonal, but two rows is cleaner and less bug-prone.)
- This is the template for the whole 2D-string family — Edit Distance reuses the same grid and fill direction.

**Trick / clever variant:** none specific to LCS.

---

## Longest Palindromic Subsequence

**Description:** Given a string `s`, return the length of the longest subsequence that is a palindrome (reads the same forwards and backwards; skips allowed, not contiguous).

**Example:** `s = "bbbab"` → `4`. The subsequence `"bbbb"` is a palindrome (drop the `a`).

**Intuition (the trick):** A palindromic subsequence reads the same forwards and backwards — which means it's a subsequence that appears in **both `s` and `reverse(s)`**. So:

> **LPS(s) = LCS(s, reverse(s))**

Reverse the string and run plain LCS against the original. No new recurrence needed — it reduces to a problem you already have.

**⚠️ Trick applies to SUBSEQUENCE only:** this works because subsequences allow skips. It does **not** work for the palindromic *substring* problems (contiguous) — LCS ignores contiguity, so it would match non-adjacent characters. For substrings, use expand-around-center or interval DP instead.

**Type / knobs:** optimizing (`max`) · reduces to LCS.

**Time:** `O(n²)`.
**Space:** tabulation `O(n²)` · space-optimized `O(n)` (two rows). (Reversing the string is an extra `O(n)`.)

### Tabulation ‖ Space-optimized

```java
// ── TABULATION (2D) ── LCS(s, reverse(s))       // ── SPACE-OPTIMIZED (two rows) ──
int lps(String s) {                              int lps(String s) {
    String a = s;                                    String a = s;
    String b = new StringBuilder(s)                  String b = new StringBuilder(s)
                   .reverse().toString();                           .reverse().toString();
    int n = s.length();                              int n = s.length();
    int[][] dp = new int[n+1][n+1];                  int[] prev = new int[n+1];
    // row 0 & col 0 = 0 (empty), auto               int[] curr = new int[n+1];

    for (int i = 1; i <= n; i++) {                   for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= n; j++) {                   for (int j = 1; j <= n; j++) {
            if (a.charAt(i-1)==b.charAt(j-1)) {              if (a.charAt(i-1)==b.charAt(j-1)) {
                dp[i][j] = 1 + dp[i-1][j-1];                     curr[j] = 1 + prev[j-1];   // diagonal
            } else {                                         } else {
                dp[i][j] = Math.max(dp[i-1][j],                  curr[j] = Math.max(prev[j],      // up
                                    dp[i][j-1]);                                    curr[j-1]);  // left
            }                                                }
        }                                                }
    }                                                    int[] t=prev; prev=curr; curr=t;   // slide
    return dp[n][n];                                 }
}                                                    return prev[n];
                                                 }
```

**Tips:**
- It's literally LCS with `b = reverse(a)` — same grid, same fill, same two-row collapse. Answer at `dp[n][n]`.
- There's also a **direct interval DP**: `dp[i][j]` = LPS of `s[i..j]`; if `s[i]==s[j]` then `2 + dp[i+1][j-1]`, else `max(dp[i+1][j], dp[i][j-1])`; base `dp[i][i]=1`. But it fills by **increasing substring length** (a different fill order than our prefix convention), so the reverse+LCS reduction is the cleaner fit here.

**Trick / clever variant:** the whole entry *is* the trick — **LPS = LCS(s, reverse(s))**. Reduce-to-known-problem is the pattern to remember.

---

## Edit Distance

**Description:** Given two strings `word1` and `word2`, return the minimum number of operations to convert `word1` into `word2`. Allowed operations: **insert** a character, **delete** a character, **replace** a character.

**Example:** `word1 = "horse"`, `word2 = "ros"` → `3`. (`horse` → `rorse` (replace h→r) → `rose` (delete r) → `ros` (delete e).)

**Intuition:** `dp[i][j]` = min edits to turn the first `i` chars of `word1` into the first `j` chars of `word2`. Same prefix grid as LCS, but minimizing. At each pair of characters: if they **match**, no edit is needed — just carry the diagonal (`dp[i-1][j-1]`) with **no `+1`**. If they **don't match**, do the cheapest of three operations, each `+1`: **replace** (diagonal), **delete** from word1 (up), **insert** into word1 (left). Base cases count *up*: turning an empty string into a length-`k` string costs `k` inserts, and vice-versa `k` deletes.

**Signature to remember:** match → carry diagonal (no cost); mismatch → `1 + min(diagonal, up, left)` = `1 + min(replace, delete, insert)`.

**Type / knobs:** optimizing (`min`) · three choices per mismatched cell · base row/col count UP (not zeros) · answer at `dp[m][n]`.

**Time:** `O(m·n)` (every rung).
**Space:** recurrence/memo `O(m·n)` · tabulation `O(m·n)` · space-optimized `O(n)` (two rows).

### Recurrence ‖ Memo

```java
// ── RECURRENCE ──                              // ── MEMO ──  (recurrence + cache lines)
int edit(String a, String b) {                   int edit(String a, String b) {
    return dfs(a.length(),                           Integer[][] memo =
              b.length(), a, b);                         new Integer[a.length()+1][b.length()+1];
}                                                    return dfs(a.length(), b.length(), a, b, memo);
                                                 }
int dfs(int i, int j,                            int dfs(int i, int j,
        String a, String b) {                            String a, String b, Integer[][] memo) {
    // base: empty prefix                            // base: empty prefix
    if (i == 0) return j;  // j inserts              if (i == 0) return j;  // j inserts
    if (j == 0) return i;  // i deletes              if (j == 0) return i;  // i deletes
                                                     if (memo[i][j]!=null) return memo[i][j];  // hit
    if (a.charAt(i-1)==b.charAt(j-1)) {              if (a.charAt(i-1)==b.charAt(j-1)) {
        // match -> carry diagonal, no cost              // match -> carry diagonal, no cost
        return dfs(i-1, j-1, a, b);                      memo[i][j] = dfs(i-1, j-1, a, b, memo);
    } else {                                         } else {
        // mismatch -> 1 + min(rep, del, ins)            // mismatch -> 1 + min(rep, del, ins)
        return 1 + Math.min(dfs(i-1,j-1,a,b),            memo[i][j] = 1 + Math.min(dfs(i-1,j-1,a,b,memo),
                   Math.min(dfs(i-1,j,a,b),                            Math.min(dfs(i-1,j,a,b,memo),
                            dfs(i,j-1,a,b)));                                    dfs(i,j-1,a,b,memo)));
    }                                                }
}                                                    return memo[i][j];  // store before returning
                                                 }
```

### Tabulation ‖ Space-optimized

```java
// ── TABULATION (2D) ──                          // ── SPACE-OPTIMIZED (two rows) ──
int edit(String a, String b) {                   int edit(String a, String b) {
    int m = a.length(), n = b.length();              int m = a.length(), n = b.length();
    int[][] dp = new int[m+1][n+1];                  int[] prev = new int[n+1];
                                                     int[] curr = new int[n+1];
    for (int j=0;j<=n;j++) dp[0][j]=j; //inserts     for (int j=0;j<=n;j++) prev[j]=j; // row0: inserts
    for (int i=0;i<=m;i++) dp[i][0]=i; //deletes
    for (int i = 1; i <= m; i++) {                   for (int i = 1; i <= m; i++) {
                                                         curr[0] = i;   // col0: deletes (set each row!)
        for (int j = 1; j <= n; j++) {                   for (int j = 1; j <= n; j++) {
            if (a.charAt(i-1)==b.charAt(j-1)) {              if (a.charAt(i-1)==b.charAt(j-1)) {
                dp[i][j] = dp[i-1][j-1];                        curr[j] = prev[j-1];           // diag
            } else {                                         } else {
                dp[i][j]=1+Math.min(dp[i-1][j-1],               curr[j]=1+Math.min(prev[j-1],  // replace
                        Math.min(dp[i-1][j],                            Math.min(prev[j],      // delete
                                 dp[i][j-1]));                                   curr[j-1]));  // insert
            }                                                }
        }                                                }
    }                                                    int[] t=prev; prev=curr; curr=t;   // slide
    return dp[m][n];                                 }
}                                                    return prev[n];
                                                 }
```

**Cell mapping (2D → two rows):** `dp[i-1][j-1]` → `prev[j-1]` · `dp[i-1][j]` → `prev[j]` · `dp[i][j-1]` → `curr[j-1]`.

**Tips:**
- **Base row/col count UP** (`0,1,2,3,…`), NOT zeros — this is the #1 Edit Distance bug. `dp[0][j]=j` (j inserts), `dp[i][0]=i` (i deletes). Unlike LCS whose base row/col are all 0.
- **Match carries the diagonal with NO `+1`** — matching is free. (LCS *added* 1 on match; here you add 0.) The `+1` only appears on mismatch.
- Three neighbours map to operations: **diagonal = replace, up = delete, left = insert**.
- **Space-opt wrinkle:** set `curr[0] = i` at the **start of every row** — the col-0 base (delete count) changes per row, so it can't be seeded once. Same wrinkle as Min Path Sum's `dp[0]`. (In LCS col-0 stayed 0, so this wasn't needed.)
- Prefix off-by-one: compare `charAt(i-1)` / `charAt(j-1)`.

**Trick / clever variant:** none standard. Worth noting the **symmetry with LCS** — same prefix grid and fill; LCS *maximizes* over 1 match rule + 2 neighbours, Edit Distance *minimizes* over 3 operations. Same skeleton, different cell logic.

---

## Longest Common Substring

**Description:** Given two strings `a` and `b`, return the length of the longest **substring** (contiguous) common to both.

**Example:** `a = "abcde"`, `b = "abfce"` → `2`. The longest common contiguous substring is `"ab"`. (`"ce"` isn't contiguous in `a`; the subsequence LCS would give a longer `"abce"` = 4, but that's *not* contiguous — hence the difference.)

**Intuition:** Almost identical to LCS, but *contiguous* — so `dp[i][j]` = length of the common substring **ending exactly at** `a[i-1]` and `b[j-1]`. If the two characters **match**, extend the diagonal run: `1 + dp[i-1][j-1]`. If they **don't match**, the run is broken — reset to `0` (you can't skip and continue like LCS does). The answer is the **maximum value anywhere in the table**, since the longest run can end at any position — not `dp[m][n]`.

**Signature to remember:** match → diagonal `+ 1`; mismatch → `0` (reset); answer → `max` over all cells.

**Type / knobs:** optimizing (`max`) · "ending at (i,j)" state · answer = max over table (NOT `dp[m][n]`).

**Time:** `O(m·n)` (every rung).
**Space:** tabulation `O(m·n)` · space-optimized `O(n)` (two rows).

### Tabulation ‖ Space-optimized

```java
// ── TABULATION (2D) ──                          // ── SPACE-OPTIMIZED (two rows) ──
int lcsub(String a, String b) {                  int lcsub(String a, String b) {
    int m = a.length(), n = b.length();              int m = a.length(), n = b.length();
    int[][] dp = new int[m+1][n+1];                  int[] prev = new int[n+1];
    int best = 0;                                    int[] curr = new int[n+1];
    // row 0 & col 0 = 0 (empty), auto               int best = 0;

    for (int i = 1; i <= m; i++) {                   for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {                   for (int j = 1; j <= n; j++) {
            if (a.charAt(i-1)==b.charAt(j-1)) {              if (a.charAt(i-1)==b.charAt(j-1)) {
                dp[i][j] = 1 + dp[i-1][j-1];                     curr[j] = 1 + prev[j-1]; // extend
                best = Math.max(best, dp[i][j]);                 best = Math.max(best, curr[j]);
            } else {                                         } else {
                dp[i][j] = 0;  // run broken                     curr[j] = 0;   // run broken
            }                                                }
        }                                                }
    }                                                    int[] t=prev; prev=curr; curr=t; // slide
    return best;                                     }
}                                                    return best;
                                                 }
```

**Cell mapping (2D → two rows):** `dp[i-1][j-1]` → `prev[j-1]` (only the diagonal is ever read).

**Tips:**
- **The two differences from LCS:** (1) mismatch resets to `0` instead of `max(up, left)`; (2) answer is the running `max` over all cells, not `dp[m][n]`. Everything else — grid, prefix indexing, fill direction — is identical.
- The state is "common substring **ending at** `i,j`" (like LIS / Maximal Square's "ending here"), which is *why* a mismatch resets: the run ending here is broken the moment the characters differ.
- **Space-opt note:** on mismatch you must explicitly set `curr[j] = 0` (don't skip it), or a stale value from the previous row leaks through. Only the diagonal (`prev[j-1]`) is ever read, so two rows handles it cleanly.
- Recurrence/memo rungs exist but are rarely used — the "ending at (i,j)" state is awkward to express top-down (you'd thread the run length through the recursion). Tabulation is the natural form, so only the two bottom rungs are shown.

**Trick / clever variant:** it's essentially **LCS with two tweaks** (mismatch → 0, answer → max cell). Recognizing "substring = contiguous = mismatch breaks the run" is the whole insight — the same subarray-vs-subsequence distinction from LIS.

---

# Interval DP

**Defining trait:** `dp[i][j]` describes a **range `[i, j]`**, and depends on **smaller ranges inside/around it** — so you fill by **increasing range length** (shortest ranges first), NOT the prefix row-by-row order. Base cases are the length-1 (or empty) ranges.

**Two sub-patterns to recognize:**

**1. Ends-inward** — `dp[i][j]` looks at the two ends and the *inner* range `dp[i+1][j-1]`.
- Shape: `dp[i][j]` = (something about `s[i]`, `s[j]`) combined with `dp[i+1][j-1]` (and sometimes `dp[i+1][j]`, `dp[i][j-1]`).
- Examples: Longest Palindromic Substring, Palindromic Substrings, Longest Palindromic Subsequence.
- Complexity: `O(n²)` (two loops: length, then start).

**2. Split-point** — `dp[i][j]` = combine over every split `k` inside the range of the two sub-ranges `dp[i][k]` and `dp[k][j]`.
- Shape: `dp[i][j] = best over k in (i, j) of ( dp[i][k] + dp[k][j] + cost(i, k, j) )`.
- The mental move: pick the **last thing handled** in the range (last balloon burst, last cut made, last multiply), which splits the range into two *independent* halves you already solved.
- Examples: **Burst Balloons** (canonical), Matrix Chain Multiplication, Min Cost to Cut a Stick, Strange Printer, Guess Number Higher/Lower II. Game-on-a-range (Stone Game) is a cousin.
- Complexity: `O(n³)` — a **third loop over the split point `k`** inside the `(i, j)` loops.

**Fill-order template (both sub-patterns):**
```java
for (int len = 1; len <= n; len++)            // ranges shortest -> longest
    for (int i = 0; i + len - 1 < n; i++) {
        int j = i + len - 1;
        // ends-inward:  use dp[i+1][j-1], dp[i+1][j], dp[i][j-1]
        // split-point:  for (int k = i; k < j; k++) combine dp[i][k], dp[k+1][j]
    }
```

**Why it can't use the prefix convention:** an interval cell needs *smaller intervals* (inner or split sub-ranges), and those aren't "the first i / first j elements" — they're arbitrary sub-ranges. So the fill is driven by **range length**, and there's usually **no clean space-optimization** (you need the whole 2D table of sub-ranges).

---

## Longest Palindromic Substring

**Description:** Given a string `s`, return the longest **substring** (contiguous) that is a palindrome.

**Example:** `s = "babad"` → `"bab"` (or `"aba"`; both length 3 are valid). For `s = "cbbd"` → `"bb"`.

**Intuition:** `isPalin(i, j)` = is `s[i..j]` a palindrome? A range is a palindrome iff its **two ends match** AND the **inside is also a palindrome**:

> `isPalin(i, j) = (s[i] == s[j]) AND isPalin(i+1, j-1)`

Base cases: a single char (`i == j`) or empty range (`i > j`) is a palindrome. To get the *longest*, check every range `(i, j)` and keep the widest one that's a palindrome.

**⚠️ Why NOT flip + Longest Common Substring:** tempting (mirrors the subsequence trick), but it **fails**. `LCSubstring(s, reverse(s))` can match a substring of `s` whose reverse appears at a *different* position — not the same span — so it isn't actually a palindrome. Counterexample: `s = "abacdfgdcaba"` → the trick returns `"abacd"` (len 5), but the true answer is `"aba"` (len 3). (The subsequence version works because subsequences are position-free; substrings aren't.)

**Signature to remember:** ends match AND inner range is a palindrome → this range is a palindrome. Interval DP fills by **length**.

**Type / knobs:** feasibility (`isPalin` boolean) driving an optimizing search (longest) · interval DP (depends on inner `dp[i+1][j-1]`) · fills by substring length.

**Time:** `O(n²)` (memo/tabulation and expand-around-center).
**Space:** memo/tabulation `O(n²)` · expand-around-center `O(1)`. (Interval DP does **not** collapse to a couple of rows — the inner-range dependency doesn't reduce cleanly, so no clean space-opt rung.)

### Recurrence ‖ Memo

```java
// ── RECURRENCE ──  O(n^3) unmemoized             // ── MEMO ──  O(n^2)  (recurrence + cache)
String longestPalindrome(String s) {             Boolean[][] memo;
    int n = s.length(),                          String longestPalindrome(String s) {
        start = 0, maxLen = 0;                        int n = s.length(), start = 0, maxLen = 0;
                                                      memo = new Boolean[n][n];
    for (int i = 0; i < n; i++)                       for (int i = 0; i < n; i++)
      for (int j = i; j < n; j++)                       for (int j = i; j < n; j++)
        if (isPalin(i, j, s)                              if (isPalin(i, j, s)
            && j-i+1 > maxLen) {                              && j-i+1 > maxLen) {
            start = i; maxLen = j-i+1;                        start = i; maxLen = j-i+1;
        }                                                 }
    return s.substring(start,                        return s.substring(start, start+maxLen);
                       start+maxLen);            }
}
                                                 boolean isPalin(int i, int j, String s) {
boolean isPalin(int i, int j, String s) {            if (i >= j) return true;   // empty/1 char
    if (i >= j) return true;                         if (memo[i][j] != null) return memo[i][j];
    if (s.charAt(i) != s.charAt(j))                  if (s.charAt(i) != s.charAt(j))
        return false;                                    return memo[i][j] = false;
    // ends match -> check inner range               // ends match -> check inner range
    return isPalin(i+1, j-1, s);                     return memo[i][j] = isPalin(i+1, j-1, s);
}                                                }
```

### Tabulation (interval DP — fill by length) ‖ Expand-around-center (practical, O(1) space)

```java
// ── TABULATION (fill by length) ──               // ── EXPAND-AROUND-CENTER (what to actually write) ──
String longestPalindrome(String s) {             String longestPalindrome(String s) {
    int n = s.length();                              if (s.length() < 2) return s;
    boolean[][] dp = new boolean[n][n];              int start = 0, maxLen = 1;
    int start = 0, maxLen = 1;
    for (int i=0;i<n;i++) dp[i][i]=true; //len1      for (int c = 0; c < s.length(); c++) {
                                                         int a = expand(s, c, c);     // odd center
    for (int len = 2; len <= n; len++)                   int b = expand(s, c, c+1);   // even center
      for (int i = 0; i+len-1 < n; i++) {                int len = Math.max(a, b);
        int j = i+len-1;                                 if (len > maxLen) {
        if (s.charAt(i)==s.charAt(j)                          maxLen = len;
            && (len==2 || dp[i+1][j-1])) {                    start = c - (len-1)/2;
            dp[i][j] = true;                             }
            if (len > maxLen){maxLen=len;start=i;}   }
        }                                            return s.substring(start, start+maxLen);
      }                                            }
    return s.substring(start,start+maxLen);        // expand while chars match; return palindrome length
}                                                int expand(String s, int l, int r) {
                                                     while (l>=0 && r<s.length()
                                                            && s.charAt(l)==s.charAt(r)) { l--; r++; }
                                                     return r - l - 1;
                                                 }
```

**Tips:**
- **Expand-around-center is the interview answer** — O(n²) time but **O(1) space**, and it directly grows palindromes so there's no reduction to get wrong. `2n-1` centers: `n` single-char (odd) + `n-1` gap (even).
- The `start = c - (len-1)/2` line backs the start index out of a center + length — the one fiddly bit; trace it once on `"bab"`.
- **Interval DP fills by substring length** because `dp[i][j]` depends on the *inner* `dp[i+1][j-1]` — shorter ranges must exist first. This is the signature of interval DP and why it breaks the prefix/low-to-high convention. It also has **no clean space-opt** (the inner-range dependency doesn't collapse to two rows).
- **Palindromic Substrings (LC 647)** — "count all palindromic substrings" — is the *same* techniques: with expand-around-center, increment a counter on each successful expansion; with interval DP, count every `true` cell. One-line change.

**Trick / clever variant:** **Manacher's algorithm** solves it in `O(n)`, but it's rarely expected — expand-around-center's `O(n²)` is the standard answer. The reduce-to-known **flip + LCSubstring does NOT work** here (see warning above) — a good example of a reduction that fails because it drops a constraint (contiguity/same-span).

---

## Matrix Chain Multiplication  *(split-point interval DP — the canonical one)*

**Description:** Given `n` matrices whose dimensions are in `arr[]` of size `n+1` (matrix `i` is `arr[i-1] × arr[i]`), find the minimum number of scalar multiplications needed to multiply the whole chain. You choose the *order* of multiplication (the parenthesization); the result is the same, but the cost differs wildly.

**Example:** `arr = [40, 20, 30, 10, 30]` (4 matrices) → `26000`. The grouping you pick changes the total cost by a large factor; this is the cheapest.

**Intuition:** `solve(i, j)` = min cost to multiply the slice of matrices `[i..j]` into a single matrix. Whatever order you use, there's **one final multiplication** that combines two already-collapsed halves. That final multiply splits the range at some point `k`: multiply `[i..k]` into one matrix, `[k+1..j]` into one matrix, then multiply those two. Once you fix the split `k`, the two halves are **independent** (each collapses on its own), so their costs simply add. You don't know the best `k`, so try every split and take the min:

> `solve(i, j) = min over k in [i, j-1] of ( solve(i, k) + solve(k+1, j) + arr[i-1]*arr[k]*arr[j] )`

The `arr[i-1]*arr[k]*arr[j]` term is the cost of the final multiply: left result is `arr[i-1] × arr[k]`, right result is `arr[k] × arr[j]`, so combining them costs `arr[i-1]·arr[k]·arr[j]`. Base case: `i >= j` → 0 (single matrix, nothing to multiply).

**Concrete — `solve(1, 4)` (multiply M1·M2·M3·M4):** try each split point for the *last* multiplication —
- `k=1`: `(M1) × (M2·M3·M4)` → `solve(1,1) + solve(2,4) + arr[0]*arr[1]*arr[4]`
- `k=2`: `(M1·M2) × (M3·M4)` → `solve(1,2) + solve(3,4) + arr[0]*arr[2]*arr[4]`
- `k=3`: `(M1·M2·M3) × (M4)` → `solve(1,3) + solve(4,4) + arr[0]*arr[3]*arr[4]`

Each = cost(left half) + cost(right half) + cost(combine the two). Take the min. The left/right halves are the same subproblem, one size smaller — recursion bottoms out at single matrices (cost 0).

**Signature to remember:** fix the **last** multiplication → range splits into two independent halves → `min over k of (left + right + combine)`. "Last, not first" is what makes the halves independent.

**Type / knobs:** optimizing (`min`) · **split-point interval DP** (loop over split `k`) · fill by range length.

**Time:** `O(n³)` — `O(n²)` ranges `(i,j)` × `O(n)` split loop `k`.
**Space:** `O(n²)` for the table (memo/tabulation). No clean space-opt — interval DP needs the full table of sub-ranges.

### Recurrence ‖ Memo

```java
// ── RECURRENCE ── (exponential)                  // ── MEMO ── O(n^3)  (recurrence + cache)
int mcm(int[] arr) {                             int[][] memo;
    int n = arr.length - 1;                      int mcm(int[] arr) {
    return solve(arr, 1, n);                         int n = arr.length - 1;
}                                                    memo = new int[n+1][n+1];
                                                     for (int[] r : memo) Arrays.fill(r, -1); // -1=uncomputed
int solve(int[] arr, int i, int j) {             int solve(int[] arr, int i, int j) {
    if (i >= j) return 0;   // single matrix         if (i >= j) return 0;
                                                     if (memo[i][j] != -1) return memo[i][j];   // hit
    int min = Integer.MAX_VALUE;                     int min = Integer.MAX_VALUE;
    for (int k = i; k < j; k++) {                    for (int k = i; k < j; k++) {
        int cost = solve(arr, i, k)                      int cost = solve(arr, i, k)
                 + solve(arr, k+1, j)                             + solve(arr, k+1, j)
                 + arr[i-1]*arr[k]*arr[j];                        + arr[i-1]*arr[k]*arr[j];
        min = Math.min(min, cost);                       min = Math.min(min, cost);
    }                                                }
    return min;                                      return memo[i][j] = min;   // store before return
}                                                }
```

### Tabulation  *(interval DP — fill by range length; no clean space-opt)*

```java
int mcm(int[] arr) {
    int n = arr.length - 1;
    int[][] dp = new int[n + 1][n + 1];
    // dp[i][i] = 0 (single matrix) — array init already 0, no seeding

    for (int len = 2; len <= n; len++) {              // range LENGTH, shortest -> longest
        for (int i = 1; i + len - 1 <= n; i++) {      // left end
            int j = i + len - 1;                      // right end (derived)
            dp[i][j] = Integer.MAX_VALUE;
            for (int k = i; k < j; k++) {             // split point
                int cost = dp[i][k]                   // left half  (shorter -> already filled)
                         + dp[k + 1][j]               // right half (shorter -> already filled)
                         + arr[i - 1] * arr[k] * arr[j];   // final multiply
                dp[i][j] = Math.min(dp[i][j], cost);
            }
        }
    }
    return dp[1][n];   // whole chain
}
```

**Tips:**
- **Outer loop is `len`, not `i`** — this is the interval-DP signature. Filling shortest ranges first guarantees the split sub-ranges `dp[i][k]`, `dp[k+1][j]` are already computed when you reach `dp[i][j]`.
- Base case is the **diagonal** `dp[i][i] = 0` (auto-zero), so `len` starts at 2.
- **"Last, not first"** is the whole trick: fixing the *last* multiplication splits the range into two independent halves. Trying to fix the *first* multiply tangles the sub-ranges — don't.
- Sentinel `-1` in the memo is safe (a real cost is always ≥ 0).
- **Gap-based fill** is an equivalent alternative: `for (i = n..1) for (j = i+1..n)` — also fills the diagonal outward. Length-based is clearer about *why* (shortest first).
- This is the **template for the whole split-point family** — Palindrome Partitioning, Boolean Parenthesization, Min Cost to Cut a Stick, Burst Balloons all reuse this `i / j / k` skeleton.

**Trick / clever variant:** the reframe *is* the trick — **think about the last operation, which splits the range into two independent sub-ranges.** Master this recurrence and the rest of the split-point family is variations on it.

---

## Burst Balloons  *(split-point interval DP — the hard one)*

**Description:** `n` balloons, each with a value `nums[i]`. Bursting balloon `i` earns `nums[left] * nums[i] * nums[right]` coins, where `left`/`right` are the currently-adjacent *un-burst* balloons (out-of-bounds neighbours count as `1`). Choose a burst order to maximize total coins.

**Example:** `nums = [3, 1, 5, 8]` → `167`. (Best order: burst 1, then 5, then 3, then 8.)

**Intuition — the "burst LAST" reframe (this is the whole problem):** You can't think about which balloon to burst *first*, because bursting it makes its two neighbours adjacent — the left and right sides become *entangled*, so they aren't independent subproblems. Instead, think about which balloon is burst **last** in a range. Pad the array with `1`s at both ends: `arr = [1] + nums + [1]`. Define `dp[i][j]` = max coins from bursting all balloons **strictly between** indices `i` and `j` (open interval). If balloon `k` is the *last* one burst in `(i, j)`, then at that moment every other balloon in the range is already gone — so `k`'s neighbours are exactly the fixed boundaries `i` and `j`. It earns `arr[i] * arr[k] * arr[j]`. And because `k` stood as a **wall** between the two sides until the very end, the left part `(i, k)` and right part `(k, j)` were burst **independently**:

> `dp[i][j] = max over k in (i, j) of ( dp[i][k] + dp[k][j] + arr[i]*arr[k]*arr[j] )`

Base case: `j - i < 2` → 0 (no balloons strictly between). Answer: `dp[0][n+1]` (the full padded range).

**Concrete — full range `dp[0][5]` for `nums=[3,1,5,8]`, padded `arr=[1,3,1,5,8,1]`** (balloons at indices 1–4). Which balloon is burst **last**?
- `k=1` last: `dp[0][1] + dp[1][5] + arr[0]*arr[1]*arr[5]` = `0 + dp[1][5] + 1*3*1`
- `k=2` last: `dp[0][2] + dp[2][5] + arr[0]*arr[2]*arr[5]` = `dp[0][2] + dp[2][5] + 1*1*1`
- `k=3` last: `dp[0][3] + dp[3][5] + arr[0]*arr[3]*arr[5]` = `dp[0][3] + dp[3][5] + 1*5*1`
- `k=4` last: `dp[0][4] + dp[4][5] + arr[0]*arr[4]*arr[5]` = `dp[0][4] + 0 + 1*8*1`

Each = coins(left sub-range) + coins(right sub-range) + coins(bursting `k` against the fixed walls `i`, `j`). Take the max. The `+1` padding is what makes the boundary multiplications clean (`arr[i]`/`arr[j]` are real values or the padded `1`s).

**Signature to remember:** pad with 1s · `dp[i][j]` = open interval `(i,j)` · fix the **last** balloon `k` → its neighbours are the fixed walls `i,j` → left/right sub-ranges independent · `max over k of (left + right + arr[i]*arr[k]*arr[j])`.

**Type / knobs:** optimizing (`max`) · **split-point interval DP** · open-interval state + boundary padding · fill by range length.

**Time:** `O(n³)` — `O(n²)` intervals × `O(n)` split loop.
**Space:** `O(n²)` for the table. No clean space-opt (interval DP).

### Recurrence ‖ Memo

```java
// ── RECURRENCE ── (exponential)                  // ── MEMO ── O(n^3)  (recurrence + cache)
int maxCoins(int[] nums) {                       int[][] memo;
    int n = nums.length;                         int maxCoins(int[] nums) {
    int[] arr = new int[n + 2];                      int n = nums.length;
    arr[0] = arr[n+1] = 1;                           int[] arr = new int[n + 2];
    for (int i=0;i<n;i++) arr[i+1]=nums[i];          arr[0] = arr[n+1] = 1;
    return solve(arr, 0, n + 1);                     for (int i=0;i<n;i++) arr[i+1]=nums[i];
}                                                    memo = new int[n+2][n+2];
                                                     for (int[] r:memo) Arrays.fill(r,-1); //uncomputed
int solve(int[] arr, int i, int j) {                 return solve(arr, 0, n + 1);
    if (j - i < 2) return 0;  // none between    }
                                                 int solve(int[] arr, int i, int j) {
    int max = 0;                                     if (j - i < 2) return 0;
    for (int k = i+1; k < j; k++) {  // k last       if (memo[i][j] != -1) return memo[i][j];  // hit
        int coins = solve(arr, i, k)                 int max = 0;
                  + solve(arr, k, j)                 for (int k = i+1; k < j; k++) {  // k last
                  + arr[i]*arr[k]*arr[j];                int coins = solve(arr, i, k)
        max = Math.max(max, coins);                                + solve(arr, k, j)
    }                                                              + arr[i]*arr[k]*arr[j];
    return max;                                          max = Math.max(max, coins);
}                                                    }
                                                     return memo[i][j] = max;   // store before return
                                                 }
```

### Tabulation  *(interval DP — fill by range length; no clean space-opt)*

```java
int maxCoins(int[] nums) {
    int n = nums.length;
    int[] arr = new int[n + 2];
    arr[0] = arr[n + 1] = 1;
    for (int i = 0; i < n; i++) arr[i + 1] = nums[i];

    int[][] dp = new int[n + 2][n + 2];
    // len = gap (j - i); need >= 2 for a balloon to sit strictly between i and j
    for (int len = 2; len <= n + 1; len++) {          // gap, small -> large
        for (int i = 0; i + len <= n + 1; i++) {      // left boundary
            int j = i + len;                          // right boundary
            for (int k = i + 1; k < j; k++) {         // last balloon burst
                int coins = dp[i][k]                  // left sub-range (smaller gap, filled)
                          + dp[k][j]                  // right sub-range (smaller gap, filled)
                          + arr[i] * arr[k] * arr[j]; // burst k against walls i, j
                dp[i][j] = Math.max(dp[i][j], coins);
            }
        }
    }
    return dp[0][n + 1];   // full padded range
}
```

**Tips:**
- **"Burst LAST, not first"** is the entire insight. Last-burst balloon `k` has fixed neighbours `i` and `j` (the range boundaries), which makes the two sides independent. Thinking "first" entangles the sides (neighbours change) and doesn't split cleanly.
- **Pad the array with 1s** at both ends so boundary bursts multiply by 1 cleanly — avoids special-casing the edges. `dp[i][j]` is the **open** interval `(i, j)`: balloons strictly between, `i` and `j` are walls, never burst.
- The split term is `dp[i][k] + dp[k][j]` (note `dp[k][j]`, not `dp[k+1][j]` — because `k` itself is *in* the range and burst last, so both sub-intervals still use `k` as a wall).
- Same `i / j / k` skeleton as MCM, fill by range length. The differences: open-interval state, boundary padding, and the "last" element sits *inside* both split terms.
- Sentinel `-1` is safe (coins ≥ 0). A real interval can legitimately be 0, so don't use 0 as the sentinel.

**Trick / clever variant:** the reframe *is* the trick — **fix the LAST balloon burst**, so its neighbours are the fixed range boundaries and the two sub-ranges become independent. This "reason about the last operation" move is the signature of hard split-point interval DP; if "first" won't decompose cleanly, try "last."

---

# Subsequence DP (LIS family)

State is usually **"best ending exactly at index `i`"** (like Kadane / Maximal Square). The recurrence is **loop-inside-step**: to fill `dp[i]`, scan *all* earlier elements you could extend from. Answer is the **max over all cells**, not `dp[n-1]`. Keeps the whole array; O(n²) time (an O(n log n) trick exists for LIS itself).

---

## Longest Increasing Subsequence

**Description:** Given `nums`, return the length of the longest strictly increasing **subsequence** (skips allowed, not contiguous).

**Example:** `nums = [10,9,2,5,3,7,101,18]` → `4`. One LIS is `[2,3,7,101]` (or `[2,5,7,18]`).

**Intuition:** `dp[i]` = length of the longest increasing subsequence **ending exactly at** index `i` (`nums[i]` is its last element). To end at `i`, the previous element must be some earlier, smaller `nums[j]` (`j < i`, `nums[j] < nums[i]`) — and the best chain ending at `j` is already `dp[j]`. So `nums[i]` glues onto the best earlier chain it can legally extend: `dp[i] = 1 + max(dp[j])` over valid `j`, or `1` if none. Answer = max over all `dp[i]` (the longest chain can end anywhere). *Skippable → look back at ALL earlier elements (the inner loop), which is why it's O(n²).*

**Concrete — `dp[4]` for `nums = [3,1,4,2,5]`, computing the element `5` at index 4:** earlier elements smaller than 5 are `3`(dp=1), `1`(dp=1), `4`(dp=2), `2`(dp=2). Each offers to be 5's predecessor: extend the best of them (dp=2) → `dp[4] = 2 + 1 = 3`. (Chains like `[1,4,5]` or `[1,2,5]`.) `5` hunts backward for the longest chain it's allowed to sit on top of.

**Signature to remember:** "ending at `i`" · `dp[i] = 1 + max(dp[j] : j<i, nums[j]<nums[i])` · answer = **max over all cells**.

**Type / knobs:** optimizing (`max`) · loop-inside-step (inner scan over `j<i`) · answer = max over table.

**Time:** O(n²) (DP) · O(n log n) (patience sorting).
**Space:** O(n) (the `dp` array — doesn't reduce further).

### Recurrence ‖ Memo

```java
// ── RECURRENCE ── (exponential-ish)              // ── MEMO ── O(n^2)  (recurrence + cache)
int lengthOfLIS(int[] nums) {                    int[] memo;
    int n = nums.length, best = 0;               int lengthOfLIS(int[] nums) {
    for (int i = 0; i < n; i++)                      int n = nums.length, best = 0;
        best = Math.max(best,                        memo = new int[n];
                        solve(nums, i));             Arrays.fill(memo, -1);   // -1 = uncomputed
    return best;                                     for (int i = 0; i < n; i++)
}                                                        best = Math.max(best, solve(nums, i));
                                                     return best;
int solve(int[] nums, int i) {                   }
    // LIS ending exactly at i                    int solve(int[] nums, int i) {
    int len = 1;                                     if (memo[i] != -1) return memo[i];
    for (int j = 0; j < i; j++)                      int len = 1;
        if (nums[j] < nums[i])                       for (int j = 0; j < i; j++)
            len = Math.max(len,                          if (nums[j] < nums[i])
                           1 + solve(nums, j));              len = Math.max(len, 1 + solve(nums, j));
    return len;                                      return memo[i] = len;
}                                                }
```

### Tabulation ‖ Patience sorting (O(n log n), the clever variant)

```java
// ── TABULATION O(n^2) ──                          // ── PATIENCE SORTING O(n log n) ──
int lengthOfLIS(int[] nums) {                    int lengthOfLIS(int[] nums) {
    int n = nums.length;                             int[] tails = new int[nums.length];
    int[] dp = new int[n];                           int size = 0;   // length of LIS so far
    Arrays.fill(dp, 1);  // each alone = 1
    int best = 1;                                    for (int x : nums) {
    for (int i = 0; i < n; i++) {                        int lo = 0, hi = size;
        for (int j = 0; j < i; j++)                      while (lo < hi) {        // first tail >= x
            if (nums[j] < nums[i])                           int mid = (lo + hi) / 2;
                dp[i] = Math.max(dp[i], dp[j]+1);            if (tails[mid] < x) lo = mid + 1;
        best = Math.max(best, dp[i]);                        else hi = mid;
    }                                                    }
    return best;                                         tails[lo] = x;   // extend or replace
}                                                        if (lo == size) size++;
                                                     }
                                                     return size;
                                                 }
```

**Tips:**
- **Answer = max over all `dp[i]`, NOT `dp[n-1]`** — the longest chain can end anywhere (same "ending here" trap as Maximal Square / Kadane). #1 LIS mistake.
- **Strict vs non-strict:** `nums[j] < nums[i]` for *strictly* increasing; use `<=` for non-decreasing. Read the problem carefully (semantics check).
- **Patience sorting (O(n log n)):** `tails[k]` = smallest possible tail of an increasing subsequence of length `k+1`. For each `x`, binary-search the first tail `>= x` and overwrite it (or append if `x` is bigger than all). The **length of `tails` is the LIS length** — but note `tails` itself is *not* a valid subsequence, just a bookkeeping array. Use "first tail `>= x`" (lower bound) for strict; "first `> x`" (upper bound) for non-decreasing.

**Trick / clever variant:** **patience sorting / binary search → O(n log n)** (shown above). The insight: maintain the smallest tail per subsequence length; a new element either extends the longest pile or improves (lowers) an existing tail.

---

## Russian Doll Envelopes

**Description:** Each envelope is `[width, height]`. Envelope A fits inside B only if **both** `A.width < B.width` and `A.height < B.height` (strict). Return the max number of envelopes you can nest (Russian-doll).

**Example:** `envelopes = [[5,4],[6,4],[6,7],[2,3]]` → `3`. Nesting: `[2,3] → [5,4] → [6,7]`.

**Intuition (LIS in disguise):** It's 2D nesting, but you can collapse one dimension by **sorting**. Sort by **width ascending**; for ties (equal width), sort by **height descending**. After sorting, width is non-decreasing left-to-right, so any strictly-increasing chain of *heights* automatically has strictly-increasing widths too — **except** equal-width pairs, which the descending-height tie-break prevents from both being chosen (their heights go down, so a strictly-increasing height LIS can't pick two of them). So the answer = **LIS on the heights array**.

**Concrete — why height DESC for ties:** take `[3,3]` and `[3,4]` (same width, can't nest).
- Height *ascending* tie → heights `[3, 4]` → LIS picks both = 2. **WRONG** (they can't nest).
- Height *descending* tie → heights `[4, 3]` → LIS picks one = 1. **Correct.**
The descending tie-break is what stops equal-width envelopes from being counted together.

**Signature to remember:** sort width ↑, height ↓ on ties → LIS on heights. The tie-break direction is the whole trick.

**Type / knobs:** optimizing (`max`) · reduces to LIS after sorting · use O(n log n) LIS (n can be large).

**Time:** O(n log n) — sort + patience-sorting LIS.
**Space:** O(n).

### Solution (sort → LIS on heights, O(n log n))

```java
int maxEnvelopes(int[][] envelopes) {
    // width ascending; on equal width, height DESCENDING (so equal widths can't both be chosen)
    Arrays.sort(envelopes, (a, b) ->
        a[0] == b[0] ? b[1] - a[1] : a[0] - b[0]);

    // LIS on the heights (patience sorting)
    int[] tails = new int[envelopes.length];
    int size = 0;
    for (int[] e : envelopes) {
        int h = e[1];
        int lo = 0, hi = size;
        while (lo < hi) {                 // first tail >= h  (strict increase)
            int mid = (lo + hi) / 2;
            if (tails[mid] < h) lo = mid + 1;
            else hi = mid;
        }
        tails[lo] = h;
        if (lo == size) size++;
    }
    return size;
}
```

**Tips:**
- **The tie-break is the whole problem.** Width ascending is obvious; height *descending* on equal widths is the subtle part — it prevents equal-width envelopes (which can't nest) from forming a fake increasing height run.
- After sorting, you've reduced a 2D nesting problem to plain **LIS on heights** — a clean example of "reduce to a known problem by removing a dimension via sorting."
- Use the **O(n log n) LIS** here (patience sorting), since envelope counts can be large; the O(n²) LIS may TLE.

**Trick / clever variant:** **sort to collapse one dimension, then LIS on the other.** The reduce-to-known move (like LPS = LCS(s, reverse(s))) — recognizing "this is LIS wearing a 2D costume" is the insight.

---

# Unbounded Knapsack

Each item can be used **unlimited times** (0, 1, 2, … copies). Signature move: when you "use" an item you **stay on the same item** (can reuse it), unlike 0/1 knapsack where taking an item moves you past it. The recurrence is **loop-inside-step** (loop over items/coins). Reach-back distance varies (any item value), so it **keeps the whole array** — no rolling-variable collapse.

**Knob checklist for this family:** *type?* (min/count/feasible → operator) · *item reuse?* unlimited (stay on item) · *order matter?* (combinations vs permutations → loop nesting).

---

## Coin Change  *(min coins — optimizing)*

**Description:** Given coin denominations `coins` and a `amount`, return the **fewest coins** that sum to `amount`, or `-1` if impossible. Unlimited coins of each type.

**Example:** `coins = [1,2,5]`, `amount = 11` → `3` (5 + 5 + 1).

**Intuition:** `dp[a]` = fewest coins to make amount `a`. The choice is "which coin do I use *last*?" — one option per coin. Using coin `c` means: one coin now (`+1`) plus the best way to make the leftover `a - c` (already solved). Try every coin, take the min:

> `dp[a] = min over coins c (c ≤ a) of ( dp[a - c] + 1 )`

Base case `dp[0] = 0` (zero coins make amount 0). "Impossible" needs an explicit sentinel (there's no valid min) — use a big value and translate to `-1` at the end.

**Concrete — `dp[6]` for `coins = [1,2,5]`** (say `dp[5]=1, dp[4]=2, dp[1]=1` already known):
- use coin 1 → `dp[5] + 1 = 1 + 1 = 2`
- use coin 2 → `dp[4] + 1 = 2 + 1 = 3`
- use coin 5 → `dp[1] + 1 = 1 + 1 = 2`

Take the min → `dp[6] = 2` (e.g. 5 + 1). Each coin proposes "spend one of me + cheapest way to make the rest"; min keeps the best.

**Signature to remember:** optimizing → `min`, base `dp[0]=0`; loop over coins; impossible → sentinel → `-1`.

**Type / knobs:** optimizing (`min`) · unbounded (reuse coins) · loop-inside-step · keeps whole array.

**Time:** O(amount × coins).
**Space:** O(amount) (the `dp` array — doesn't collapse; reach-back varies by coin value).

### Recurrence ‖ Memo

```java
// ── RECURRENCE ── (exponential)                  // ── MEMO ── O(amount·coins)
int coinChange(int[] coins, int amount) {        Integer[] memo;
    int r = dfs(coins, amount);                  int coinChange(int[] coins, int amount) {
    return r == Integer.MAX_VALUE ? -1 : r;          memo = new Integer[amount + 1];
}                                                    int r = dfs(coins, amount);
                                                     return r == Integer.MAX_VALUE ? -1 : r;
int dfs(int[] coins, int a) {                    }
    if (a == 0) return 0;                         int dfs(int[] coins, int a) {
    if (a < 0) return Integer.MAX_VALUE; // dead      if (a == 0) return 0;
                                                     if (a < 0) return Integer.MAX_VALUE;
    int best = Integer.MAX_VALUE;                    if (memo[a] != null) return memo[a];   // hit
    for (int c : coins) {                            int best = Integer.MAX_VALUE;
        int sub = dfs(coins, a - c);                 for (int c : coins) {
        if (sub != Integer.MAX_VALUE)                    int sub = dfs(coins, a - c, memo);
            best = Math.min(best, sub + 1);              if (sub != Integer.MAX_VALUE)
    }                                                        best = Math.min(best, sub + 1);
    return best;                                      }
}                                                    return memo[a] = best;
                                                 }
```

### Tabulation  *(no clean space-opt — reach-back varies by coin value)*

```java
int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);        // "impossible" sentinel (bigger than any real answer)
    dp[0] = 0;                          // base case

    for (int a = 1; a <= amount; a++) {         // fill low -> high
        for (int c : coins) {                   // loop-inside-step over coins
            if (c <= a)
                dp[a] = Math.min(dp[a], dp[a - c] + 1);
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

**Tips:**
- **Impossible sentinel:** fill with `amount + 1` (no real answer exceeds `amount`), not `MAX_VALUE` — avoids overflow on the `+1`. Final check `dp[amount] > amount` → `-1`.
- **Nested loop** (amounts outer, coins inner) is the loop-inside-step signature. Order doesn't matter here (counting *coins*, not combinations), so either nesting works.
- **No space-opt to O(1):** `dp[a]` can reach back by *any* coin value, so you need the whole array — the defining trait of this family vs. the rolling-variable 1D problems.
- Memo sentinel: `Integer[]`/`null` (a real answer can be 0, and `MAX_VALUE` is a real "impossible" value you cache).

**Trick / clever variant:** none — but note it's a **BFS** in disguise (shortest path over reachable amounts). Min-coins = fewest edges to reach `amount` from 0.

---

## Coin Change II  *(count ways — counting)*

**Description:** Given `coins` and an `amount`, return the **number of distinct combinations** that sum to `amount`. Unlimited coins. `{1,2}` and `{2,1}` are the **same** combination (count once).

**Example:** `coins = [1,2,5]`, `amount = 5` → `4`: `{5}`, `{1,1,1,2}`, `{1,2,2}`, `{1,1,1,1,1}`.

**Intuition:** counting, so combine with `+` and base case `1`. The **trap** is combinations vs permutations — `{1,2}` and `{2,1}` must count once. The 2D recurrence makes this explicit: `dfs(i, a)` = ways to make `a` using coins from index `i` onward. Two choices: **skip** coin `i` (move to `i+1`) or **use** coin `i` (stay on `i`, since unlimited). Sum them.

> `dfs(i, a) = dfs(i+1, a)  +  dfs(i, a - coins[i])`

Base cases: `a == 0` → 1 (one way — take nothing more); `a < 0` or `i == n` → 0. The coin index only moves *forward* (skip advances, use stays) — that one-directional movement enforces a fixed coin order, so no permutation double-counting.

**Concrete — `dfs(coin1, amt3)` for `coins=[1,2]`, amount 3:**
- **skip coin 1** → `dfs(coin2, amt3)`: only coin 2 left → makes 3? no (2 doesn't divide into 3 cleanly with one coin type)… `2→ dfs(coin2,1)→` dead → 0 ways via pure 2s.
- **use coin 1** → `dfs(coin1, amt2)`: keep using coin 1 → leads to `{1,1,1}` and `{1,2}`.

Total = 2 ways: `{1,1,1}` and `{1,2}`. Note `{2,1}` never appears — using coin 2 requires having *skipped* coin 1 first, so coin 1 can't come after coin 2.

**Signature to remember:** counting → `+`, base `1`; 2D state `(coin index, amount)`; skip advances `i`, use stays on `i`. 1D collapse = "coins on the OUTER loop."

**Type / knobs:** counting (`+`) · unbounded · **combinations** (coins-outer loop) · impossible → 0 (free, no sentinel).

**Time:** O(amount × coins).
**Space:** O(amount × coins) memo · O(amount) 1D.

### Recurrence ‖ Memo

```java
// ── RECURRENCE ── (exponential)                  // ── MEMO ── O(amount·coins)  2D state (i, a)
int change(int amount, int[] coins) {            Integer[][] memo;
    return dfs(0, amount, coins);                int change(int amount, int[] coins) {
}                                                    memo = new Integer[coins.length][amount + 1];
                                                     return dfs(0, amount, coins);
int dfs(int i, int a, int[] coins) {             }
    if (a == 0) return 1;   // one way            int dfs(int i, int a, int[] coins) {
    if (a < 0) return 0;                             if (a == 0) return 1;
    if (i == coins.length) return 0;                 if (a < 0) return 0;
                                                     if (i == coins.length) return 0;
    int skip = dfs(i+1, a, coins);                   if (memo[i][a] != null) return memo[i][a]; //hit
    int use  = dfs(i, a - coins[i], coins);          int skip = dfs(i+1, a, coins, memo);
    return skip + use;                               int use  = dfs(i, a - coins[i], coins, memo);
}                                                    return memo[i][a] = skip + use;
                                                 }
```

### Tabulation ‖ Space-optimized (1D, coins-outer)

```java
// ── TABULATION (2D) ──                            // ── SPACE-OPTIMIZED (1D) — coins OUTER ──
int change(int amount, int[] coins) {            int change(int amount, int[] coins) {
    int n = coins.length;                            int[] dp = new int[amount + 1];
    int[][] dp = new int[n + 1][amount + 1];         dp[0] = 1;   // one way to make 0
    for (int i=0;i<=n;i++) dp[i][0]=1; // amt0=1
                                                     for (int c : coins) {          // coins OUTER
    for (int i = 1; i <= n; i++) {                       for (int a = c; a <= amount; a++) {
        for (int a = 0; a <= amount; a++) {                  dp[a] += dp[a - c];    // add ways using c
            dp[i][a] = dp[i-1][a];  // skip              }
            if (a >= coins[i-1])                     }
                dp[i][a] += dp[i][a-coins[i-1]];     return dp[amount];
        }                                        }
    }
    return dp[n][amount];
}
```

**Tips:**
- **Coins on the OUTER loop** (1D version) is the crux — it processes one coin fully before the next, so each combination is built in a fixed coin order → no permutation double-counting. Flipping to amount-outer would count *permutations* (that's Combination Sum IV).
- **Impossible is free:** if an amount can't be made, its count is just `0` — no sentinel needed (counting handles "no solution" naturally, unlike optimizing).
- 2D memo state is `(coin index, amount)` — the coin dimension is *why* no double-counting; it collapses to 1D via the outer coin loop.
- Contrast Coin Change I: there order didn't matter (counting coins), so loop nesting was free; here order matters (counting combinations), so nesting is fixed.

**Trick / clever variant:** the **coins-outer loop order** is the clever bit — it's what turns "count sequences" into "count sets." Recognizing combinations-vs-permutations decides the loop nesting.

---

# 0/1 Knapsack

Each item is used **at most once** (0 or 1 copies). Signature move: when you "take" an item you **move past it** (to `i-1`/`i+1`), unlike unbounded where you stay. State is 2D: `dp[i][budget]` = best/count/feasible using the first `i` items for a given budget/target.

**Knob checklist:** *type?* (min/count/feasible → operator) · *item reuse?* **once** (take → move past item) · *(1D space-opt only)* iterate the budget **backward** (high→low) so an item isn't reused within one pass — the famous 0/1 gotcha.

---

## Partition Equal Subset Sum

**Description:** Given `nums`, return `true` if the array can be split into two subsets with **equal sum**.

**Example:** `nums = [1,5,11,5]` → `true`. Split `{11}` and `{1,5,5}`, both sum to 11.

**Intuition (the reduction):** Two equal halves means each sums to `total/2`. But you only need to find **one** subset summing to `total/2` — the leftover *automatically* sums to `total/2` too (they're complements). So it reduces to: **"can a subset sum to exactly `total/2`?"** If `total` is odd → immediately `false`. It's a *feasibility* 0/1 knapsack: `dp[i][t]` = "can I make sum `t` using the first `i` numbers?", combine with `OR`, base `dp[i][0] = true` (sum 0 = pick nothing).

**Concrete — `dp[i][t]` choice for number `nums[i-1]`:** can I make sum `t` from the first `i` numbers?
- **skip** it → can I make `t` from the first `i-1`? → `dp[i-1][t]`
- **take** it (if `t >= nums[i-1]`) → can I make the leftover `t - nums[i-1]` from the first `i-1`? → `dp[i-1][t - nums[i-1]]`

`OR` them: reachable if *either* works. (Take moves to `i-1` — past the item — enforcing "each used once.")

**Signature to remember:** reduce to "subset sums to `total/2`?"; odd total → false; feasibility `OR`, base `true`; take → move past item.

**Type / knobs:** feasibility (`OR`) · 0/1 (each number once) · target = `total/2` · impossible → just `false`.

**Time:** O(n × target) where target = total/2.
**Space:** memo/tabulation O(n × target) · space-optimized O(target) 1D.

### Recurrence ‖ Memo

```java
// ── RECURRENCE ── (exponential)                  // ── MEMO ── O(n·target)  2D state (i, t)
boolean canPartition(int[] nums) {               Boolean[][] memo;
    int sum = 0; for (int x : nums) sum += x;     boolean canPartition(int[] nums) {
    if (sum % 2 != 0) return false;                   int sum = 0; for (int x : nums) sum += x;
    return dfs(nums, 0, sum / 2);                     if (sum % 2 != 0) return false;
}                                                     memo = new Boolean[nums.length][sum/2 + 1];
                                                      return dfs(nums, 0, sum / 2);
boolean dfs(int[] nums, int i, int t) {          }
    if (t == 0) return true;   // made target     boolean dfs(int[] nums, int i, int t) {
    if (i == nums.length || t < 0)                    if (t == 0) return true;
        return false;                                 if (i == nums.length || t < 0) return false;
    // skip i-th  OR  take i-th (move past)            if (memo[i][t] != null) return memo[i][t]; //hit
    return dfs(nums, i+1, t) ||                       return memo[i][t] =
           dfs(nums, i+1, t - nums[i]);                   dfs(nums, i+1, t, memo) ||
}                                                            dfs(nums, i+1, t - nums[i], memo);
                                                 }
```

### Tabulation ‖ Space-optimized (1D — iterate target BACKWARD)

```java
// ── TABULATION (2D) ──                            // ── SPACE-OPTIMIZED (1D, target backward) ──
boolean canPartition(int[] nums) {               boolean canPartition(int[] nums) {
    int sum = 0; for (int x:nums) sum += x;          int sum = 0; for (int x:nums) sum += x;
    if (sum % 2 != 0) return false;                  if (sum % 2 != 0) return false;
    int T = sum / 2, n = nums.length;                int T = sum / 2;
    boolean[][] dp = new boolean[n+1][T+1];          boolean[] dp = new boolean[T + 1];
    for (int i=0;i<=n;i++) dp[i][0]=true; //sum0     dp[0] = true;   // sum 0 always reachable

    for (int i = 1; i <= n; i++) {                   for (int x : nums) {
        for (int t = 0; t <= T; t++) {                   for (int t = T; t >= x; t--) {  // BACKWARD
            dp[i][t] = dp[i-1][t];  // skip                  dp[t] = dp[t] || dp[t - x];
            if (t >= nums[i-1])                          }
                dp[i][t] = dp[i][t]                  }
                    || dp[i-1][t-nums[i-1]]; //take  return dp[T];
        }                                        }
    }
    return dp[n][T];
}
```

**Tips:**
- **The reduction is the insight:** "partition into two equal halves" → "does a subset sum to `total/2`?" (the other half is forced). Odd total → instantly false.
- **1D space-opt iterates the target BACKWARD** (`t` from `T` down to `x`). This is *the* 0/1 gotcha: going forward would let the same number be added twice in one pass (turning it into unbounded knapsack). Backward guarantees each item is used at most once.
- Feasibility handles "impossible" for free — the answer is just `false`, no sentinel.
- **Set trick:** since it's feasibility, a `Set<Integer>` of reachable sums works instead of the boolean array (store only the `true` cells). Only valid for *feasibility* types (not min/count).

**Trick / clever variant:** **set-of-reachable-sums** — maintain a `Set` of achievable subset sums, expanding by each number; if `total/2` appears, return true. Compact form of feasibility DP. (Also: **Target Sum** reduces here — "assign ± to hit target" becomes "count subsets summing to `(target+total)/2`", a *counting* 0/1 knapsack.)
