A greedy algorithm makes the locally best choice at every step, without reconsidering it later, and trusts that this sequence of locally optimal choices adds up to a globally optimal answer. The hard part isn't writing greedy code — it's proving (or at least convincing yourself) that the greedy choice never needs to be undone. When that trust is misplaced, greedy gives a wrong answer with no warning; dynamic programming is the fallback when it doesn't hold.

## 1. The Core Idea

At each decision point, take whatever looks best *right now*, and never go back to reconsider it — no backtracking, no trying alternatives.

```java
// Coin change with US-style denominations {25, 10, 5, 1} — greedy happens to work here
int minCoinsGreedy(int amount, int[] denominations) {
    int count = 0;
    for (int coin : denominations) { // must be sorted largest to smallest
        count += amount / coin;
        amount %= coin;
    }
    return count;
}
```

This specific coin system works with greedy because of a property of these particular denominations (each is enough of a multiple of the smaller ones) — but greedy coin-change is a trap in general: with denominations `{1, 3, 4}` and a target of 6, greedy picks `4 + 1 + 1` (3 coins) when `3 + 3` (2 coins) is actually optimal. This is the single most important lesson about greedy: it must be proven correct for the specific problem, never assumed correct by analogy to a similar-looking one.

## 2. When Greedy Is Provably Correct: Interval Scheduling

Maximizing the number of non-overlapping intervals you can select is a case where greedy is actually provably optimal: always pick the interval that ends soonest among the remaining valid options.

```java
// Maximum number of non-overlapping intervals
int maxNonOverlapping(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[1] - b[1]); // sort by END time, not start — this is the key choice
    int count = 0, lastEnd = Integer.MIN_VALUE;
    for (int[] interval : intervals) {
        if (interval[0] >= lastEnd) {
            count++;
            lastEnd = interval[1];
        }
    }
    return count;
}
```

The proof intuition: picking the interval that frees up time soonest always leaves at least as much room for future choices as picking any other valid interval would — no future scenario is ever made worse by this choice, so it never needs to be reconsidered. This is the "exchange argument" pattern behind most correct greedy proofs: show that swapping any other valid first choice for the greedy one is never worse.

## 3. Jump Game: Track the Farthest Reachable Point

A different flavor of greedy: instead of picking discrete items, greedily track the best possible "reach" seen so far, and fail fast the moment even that can't move forward.

```java
// Can you reach the last index, where arr[i] is the max jump length from i?
boolean canJump(int[] nums) {
    int farthestReachable = 0;
    for (int i = 0; i < nums.length; i++) {
        if (i > farthestReachable) return false; // this index is unreachable — nothing before it could get here
        farthestReachable = Math.max(farthestReachable, i + nums[i]);
    }
    return true;
}
```

## 4. Gas Station: A Greedy Reset Trick

Some greedy problems are solved by tracking a running total and resetting the starting point the moment that total goes negative — the insight being that no station *before* the failure point could have been a valid start either.

```java
// Find the starting gas station that allows completing a full circuit
int canCompleteCircuit(int[] gas, int[] cost) {
    int total = 0, tank = 0, start = 0;
    for (int i = 0; i < gas.length; i++) {
        int diff = gas[i] - cost[i];
        total += diff;
        tank += diff;
        if (tank < 0) {
            start = i + 1; // this station, and everything up to it, can't be the start — try the next one
            tank = 0;
        }
    }
    return total >= 0 ? start : -1; // total >= 0 guarantees some valid start exists
}
```

## 5. Recognizing When Greedy Applies (and When It Doesn't)

| Signal in the problem | Likely approach |
| --- | --- |
| "Maximize/minimize count of non-overlapping choices" | Often greedy — sort by the property that frees up future options soonest |
| A small counterexample to a proposed greedy rule exists | Greedy is wrong here — the problem likely needs dynamic programming instead |
| "Optimal substructure with overlapping subproblems, and choices interact" | Usually DP, not greedy — the locally best choice may need to be revisited |
| Simple resource allocation with a clear "always prefer X" rule you can prove | A strong greedy candidate — try to construct an exchange argument before trusting it |

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Never trust a greedy approach without at least attempting a counterexample | The coin-change trap ({1,3,4}, target 6) shows greedy can look obviously right and still be wrong. |
| Look for an exchange argument before committing to a greedy solution | If you can show any alternative first choice is never strictly better than the greedy one, that's a real correctness proof, not just intuition. |
| Sort by the property that matters for the greedy choice, not by habit | Interval scheduling needs sorting by *end* time specifically — sorting by start time instead gives a wrong answer. |
| Fall back to dynamic programming when greedy's correctness isn't provable | DP explores/remembers more of the solution space and is the correct fallback whenever local optimality doesn't guarantee global optimality. |
| Treat "reset on failure" tracking (like the gas station trick) as a distinct greedy sub-pattern | Recognizing it by name saves re-deriving it from scratch on similar problems. |
