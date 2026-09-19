2D dynamic programming is the same overlapping-subproblems idea already covered in 1D DP, extended to problems whose subproblems need *two* pieces of state to describe — comparing two strings, moving through a grid, or choosing among items with a capacity constraint. The recurrence-finding process is identical; the table just gains a second dimension.

## 1. Why a Second Dimension Is Needed

A 1D DP table answers "what's the best/count/possibility up to position i." Some problems can't be described by one index alone — "up to character i of string A *and* character j of string B" genuinely needs both numbers to know where you are.

```java
int[][] dp = new int[m + 1][n + 1]; // dp[i][j] describes progress along two independent dimensions at once
```

## 2. Grid Traversal: Unique Paths

The simplest 2D shape: moving through a grid, where each cell's answer depends on the cells you could have come from (usually the one above and the one to the left).

```java
// Number of distinct paths from top-left to bottom-right, moving only right or down
int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];
    for (int i = 0; i < m; i++) dp[i][0] = 1; // only one way to reach any cell in the first column: straight down
    for (int j = 0; j < n; j++) dp[0][j] = 1; // only one way to reach any cell in the first row: straight across

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1]; // arrived here either from above or from the left
        }
    }
    return dp[m - 1][n - 1];
}
```

The base cases (first row, first column) are the anchor the rest of the table builds on — getting these wrong is the most common 2D DP bug, exactly like a wrong base case in 1D DP.

## 3. Two-String Problems: Longest Common Subsequence

Whenever a problem compares two strings/sequences, `dp[i][j]` almost always means "the answer considering the first `i` characters of string A and the first `j` characters of string B."

```java
int longestCommonSubsequence(String a, String b) {
    int[][] dp = new int[a.length() + 1][b.length() + 1];
    for (int i = 1; i <= a.length(); i++) {
        for (int j = 1; j <= b.length(); j++) {
            if (a.charAt(i - 1) == b.charAt(j - 1)) {
                dp[i][j] = 1 + dp[i - 1][j - 1]; // characters match — extend the subsequence found so far
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]); // no match — take the better of skipping from either string
            }
        }
    }
    return dp[a.length()][b.length()];
}
```

The `i - 1`/`j - 1` indexing offset (table is sized `(length+1) x (length+1)`) is a deliberate, common convention — it lets row/column 0 cleanly represent "zero characters considered," avoiding negative-index special cases for empty prefixes.

## 4. Edit Distance: A Richer Two-String Recurrence

A more involved version of the same shape — instead of just matching or skipping, there are three possible operations (insert, delete, replace) to choose between at each mismatch.

```java
int editDistance(String a, String b) {
    int[][] dp = new int[a.length() + 1][b.length() + 1];
    for (int i = 0; i <= a.length(); i++) dp[i][0] = i; // deleting all of a's first i characters
    for (int j = 0; j <= b.length(); j++) dp[0][j] = j; // inserting all of b's first j characters

    for (int i = 1; i <= a.length(); i++) {
        for (int j = 1; j <= b.length(); j++) {
            if (a.charAt(i - 1) == b.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1]; // characters already match — no operation needed here
            } else {
                dp[i][j] = 1 + Math.min(dp[i - 1][j - 1],  // replace
                                 Math.min(dp[i - 1][j],     // delete from a
                                          dp[i][j - 1]));   // insert into a
            }
        }
    }
    return dp[a.length()][b.length()];
}
```

## 5. Knapsack: Choice Plus a Capacity Constraint

A different flavor of 2D DP — the second dimension isn't a second string, it's a constraint (remaining capacity) that changes as items are chosen.

```java
// 0/1 Knapsack: maximize value without exceeding weight capacity, each item usable at most once
int knapsack(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[][] dp = new int[n + 1][capacity + 1]; // dp[i][c] = best value using first i items, capacity c

    for (int i = 1; i <= n; i++) {
        for (int c = 0; c <= capacity; c++) {
            dp[i][c] = dp[i - 1][c]; // baseline: don't take item i
            if (weights[i - 1] <= c) {
                dp[i][c] = Math.max(dp[i][c], dp[i - 1][c - weights[i - 1]] + values[i - 1]); // take item i
            }
        }
    }
    return dp[n][capacity];
}
```

The recurrence's shape — "best without this item" vs. "best with this item, using its cost against remaining capacity" — is the template for almost every constrained-choice DP problem, whether the constraint is weight, budget, or time.

## 6. Space Optimization: Collapsing to a 1D Rolling Array

When `dp[i][j]` only ever depends on row `i-1` (never rows further back), the full 2D table can be collapsed into a single 1D array reused across iterations — the same principle as the 1D DP space optimization, one dimension up.

```java
// Knapsack, space-optimized: one row reused, iterated capacity in reverse to avoid overwriting needed values
int knapsackOptimized(int[] weights, int[] values, int capacity) {
    int[] dp = new int[capacity + 1];
    for (int i = 0; i < weights.length; i++) {
        for (int c = capacity; c >= weights[i]; c--) { // reverse order matters — see below
            dp[c] = Math.max(dp[c], dp[c - weights[i]] + values[i]);
        }
    }
    return dp[capacity];
}
```

Iterating capacity in *reverse* is essential here: it guarantees `dp[c - weights[i]]` still holds last iteration's value (item `i` not yet applied) rather than this iteration's already-updated value — iterating forward would let item `i` be counted more than once, silently turning this into an unbounded-knapsack solution instead of 0/1.

## 7. Recognizing When 2D DP Applies

| Signal in the problem | Why 2D DP fits |
| --- | --- |
| Comparing two strings/sequences | `dp[i][j]` = answer considering prefixes of length `i` and `j` |
| Grid traversal with a running count/optimum | `dp[i][j]` = answer at that cell, built from adjacent cells |
| A choice per item, constrained by a shared budget/capacity | `dp[i][c]` = best value using first `i` items within capacity `c` |
| The state needed to describe "progress so far" genuinely needs two numbers | If one number would do, it's 1D DP, not 2D |

## 8. Best Practices

| Practice | Recommendation |
| --- | --- |
| Identify what each dimension of `dp[i][j]` actually represents before coding | Naming the two dimensions clearly ("i = prefix of A, j = prefix of B") prevents subtle indexing mistakes later. |
| Get base cases (first row and first column) right first | These anchor every other cell — a wrong base case propagates a wrong answer through the entire table. |
| Watch the direction of iteration when space-optimizing to 1D | Some recurrences (like 0/1 knapsack) require reverse iteration to avoid reusing a value that should still be "last iteration's." |
| Use a `(length+1) x (length+1)` table for two-string problems | Reserves row/column 0 for "zero characters considered," avoiding negative-index special cases for empty prefixes. |
| Derive the recurrence the same way as in 1D DP, just with two coordinates | "What was the last decision made, and what state did it come from" still applies — it now just has two axes to track. |
