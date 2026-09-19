Dynamic programming (DP) solves a problem by breaking it into overlapping subproblems, solving each subproblem exactly once, and reusing that answer instead of recomputing it — the fix for exactly the kind of redundant recursive work seen in naive Fibonacci. 1D DP problems are the ones where a single index (or single running quantity) is enough to describe "where you are" in the subproblem space.

## 1. The Two Ingredients DP Needs

DP only applies when a problem has both of these properties — without either, DP either doesn't help or doesn't apply at all.

| Property | Meaning |
| --- | --- |
| Overlapping subproblems | The same smaller subproblem gets solved multiple times if approached with plain recursion |
| Optimal substructure | The optimal answer to the whole problem can be built from optimal answers to its subproblems |

Recognizing these two properties in a new problem — usually by first writing the naive recursive solution and noticing it recomputes the same call repeatedly — is the actual skill; the coding template afterward is mostly mechanical.

## 2. Top-Down (Memoization): Recursion Plus a Cache

Start with the natural recursive definition, then add a cache so each distinct subproblem is computed once.

```java
// Fibonacci — the canonical example of turning O(2^n) into O(n) with a cache
int fib(int n, Map<Integer, Integer> memo) {
    if (n <= 1) return n;
    if (memo.containsKey(n)) return memo.get(n);
    int result = fib(n - 1, memo) + fib(n - 2, memo);
    memo.put(n, result);
    return result;
}
```

Top-down is usually the easier direction to *derive* — you write the recursive relationship first, in whatever form feels natural, then mechanically add memoization on top.

## 3. Bottom-Up (Tabulation): Build the Answer Iteratively

Once the recursive relationship is clear, it can almost always be rewritten as a loop that fills a table from the base cases upward — avoiding recursion overhead and stack depth entirely.

```java
int fibBottomUp(int n) {
    if (n <= 1) return n;
    int[] dp = new int[n + 1];
    dp[0] = 0;
    dp[1] = 1;
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2]; // this line IS the recurrence relation from the recursive version
    }
    return dp[n];
}
```

The recurrence relation (`dp[i] = dp[i-1] + dp[i-2]`) is identical in spirit to the recursive call (`fib(n-1) + fib(n-2)`) — bottom-up DP is really just "run the recursion's logic forward instead of backward," filling in known small answers first instead of recursively asking for big answers first.

## 4. Climbing Stairs: The Same Shape, a New Story

A huge number of 1D DP problems are Fibonacci wearing a different costume — recognizing the underlying recurrence is the real work.

```java
// Number of distinct ways to climb n stairs, taking 1 or 2 steps at a time
int climbStairs(int n) {
    if (n <= 2) return n;
    int[] dp = new int[n + 1];
    dp[1] = 1;
    dp[2] = 2;
    for (int i = 3; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2]; // ways to reach step i = ways to reach (i-1) + ways to reach (i-2)
    }
    return dp[n];
}
```

The reasoning that produces the recurrence: to be standing at step `i`, your last move was either a 1-step from `i-1` or a 2-step from `i-2` — so the total ways to reach `i` is the sum of the ways to reach each of those. Deriving the recurrence is always this same move: "what was the last decision made, and what state did it come from?"

## 5. House Robber: A Decision at Each Step

A slightly richer shape: at each position, you have an actual choice to make (rob this house or don't), and the recurrence reflects taking whichever choice is better.

```java
// Maximum sum of non-adjacent elements — can't rob two adjacent houses
int rob(int[] nums) {
    int[] dp = new int[nums.length + 1];
    dp[0] = 0;
    dp[1] = nums[0];
    for (int i = 2; i <= nums.length; i++) {
        // either skip house i-1 (keep dp[i-1]), or rob it (dp[i-2] + its value)
        dp[i] = Math.max(dp[i - 1], dp[i - 2] + nums[i - 1]);
    }
    return dp[nums.length];
}
```

## 6. Space Optimization: You Rarely Need the Whole Table

Once the recurrence only looks back a fixed number of steps (here, just `i-1` and `i-2`), the full array is unnecessary — two variables suffice, dropping space from `O(n)` to `O(1)`.

```java
int robOptimized(int[] nums) {
    int prev2 = 0, prev1 = 0; // dp[i-2], dp[i-1]
    for (int num : nums) {
        int current = Math.max(prev1, prev2 + num);
        prev2 = prev1;
        prev1 = current;
    }
    return prev1;
}
```

This optimization is available whenever the recurrence only depends on a constant number of previous states — worth checking for after getting a correct tabulated solution, rather than trying to find it upfront.

## 7. Recognizing When 1D DP Applies

| Signal in the problem | Why 1D DP fits |
| --- | --- |
| "Number of ways to reach/do X" | Recurrence sums the ways from each valid previous state |
| "Maximum/minimum value achievable up to position i" | Recurrence takes the best of several choices ending at i |
| A naive recursive solution recomputes the same smaller call repeatedly | The textbook sign that memoization (or its bottom-up equivalent) will help |
| The state needed to describe "progress so far" is just one number/index | If two numbers are needed, you're likely looking at 2D DP instead |

## 8. Best Practices

| Practice | Recommendation |
| --- | --- |
| Derive the recurrence by asking "what was the last decision made" | This question almost always reveals the relationship between `dp[i]` and earlier states. |
| Write the recursive (top-down) version first if the bottom-up form isn't obvious | It's usually easier to state the relationship recursively, then mechanically convert to a loop. |
| Always define and initialize base cases explicitly | An incorrect or missing base case (`dp[0]`, `dp[1]`) is the most common source of a wrong final answer despite correct-looking logic elsewhere. |
| Check whether the recurrence only needs a constant number of previous states | If so, replace the full `dp` array with a few variables to cut space from `O(n)` to `O(1)`. |
| Don't reach for DP before confirming overlapping subproblems actually exist | If subproblems don't overlap, plain recursion or a greedy/direct approach is simpler and just as fast. |
