Backtracking is systematic trial and error: build a candidate solution piece by piece, and the moment it can't possibly lead anywhere valid, undo the last step and try a different option instead. It's how you correctly enumerate "all possible X" (subsets, permutations, valid boards) without missing any or duplicating any.

## 1. The Core Template

Every backtracking solution follows the same shape: choose an option, recurse with that choice made, then explicitly undo the choice before trying the next option — the "undo" step is what makes it backtracking rather than plain recursion.

```java
void backtrack(/* current partial state */) {
    if (/* current state is a complete, valid solution */) {
        results.add(new ArrayList<>(currentState)); // record a copy — the list keeps changing after this
        return;
    }
    for (/* each option available at this point */) {
        currentState.add(option);      // choose
        backtrack(/* advance state */); // explore
        currentState.remove(currentState.size() - 1); // un-choose — this is the "back" in backtracking
    }
}
```

The "un-choose" line is the part beginners most often forget, and forgetting it means every branch after the first shares polluted state from a previous branch that was never cleaned up.

## 2. Subsets — The Simplest Shape

For every element, you either include it or don't — this binary choice at every position is the simplest backtracking template.

```java
List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}

void backtrack(int[] nums, int start, List<Integer> current, List<List<Integer>> result) {
    result.add(new ArrayList<>(current)); // every partial state along the way is itself a valid subset
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);
        backtrack(nums, i + 1, current, result); // only consider elements after i — avoids duplicate subsets
        current.remove(current.size() - 1);
    }
}
```

## 3. Permutations — Every Order Matters

Unlike subsets, permutations need every element used exactly once, in every possible order — so instead of a `start` index, you track which elements have already been used.

```java
List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, new ArrayList<>(), new boolean[nums.length], result);
    return result;
}

void backtrack(int[] nums, List<Integer> current, boolean[] used, List<List<Integer>> result) {
    if (current.size() == nums.length) {
        result.add(new ArrayList<>(current));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue; // already placed this element earlier in this branch
        used[i] = true;
        current.add(nums[i]);
        backtrack(nums, current, used, result);
        current.remove(current.size() - 1); // undo
        used[i] = false;                     // undo
    }
}
```

## 4. Pruning: The Difference Between Backtracking and Pure Brute Force

Backtracking's real power isn't just "try everything" — it's stopping early the moment a partial choice can't possibly lead to a valid solution, which avoids ever fully constructing the branches that were doomed from the start.

```java
// N-Queens: place queens so none attack each other
void solve(int n, int row, int[] positions, List<List<String>> result) {
    if (row == n) {
        result.add(buildBoard(positions, n));
        return;
    }
    for (int col = 0; col < n; col++) {
        if (isValid(positions, row, col)) { // prune immediately — don't even recurse into an invalid placement
            positions[row] = col;
            solve(n, row + 1, positions, result);
            // no explicit undo needed here — positions[row] just gets overwritten on the next iteration
        }
    }
}

boolean isValid(int[] positions, int row, int col) {
    for (int r = 0; r < row; r++) {
        int c = positions[r];
        if (c == col || Math.abs(c - col) == Math.abs(r - row)) return false; // same column or same diagonal
    }
    return true;
}
```

Checking `isValid` *before* recursing (rather than recursing all the way down and checking at the end) is what keeps this efficient — a bad queen placement in row 2 is rejected immediately, instead of wasting time exploring every possible arrangement of rows 3 through n that could never have worked anyway.

## 5. Combination Sum — Choices With Repetition

A variant where the same element can be reused, and the stopping condition is a target being reached exactly (or exceeded, requiring a prune).

```java
List<List<Integer>> combinationSum(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(candidates, target, 0, new ArrayList<>(), result);
    return result;
}

void backtrack(int[] candidates, int remaining, int start, List<Integer> current, List<List<Integer>> result) {
    if (remaining == 0) {
        result.add(new ArrayList<>(current));
        return;
    }
    if (remaining < 0) return; // prune — overshot the target, no point continuing
    for (int i = start; i < candidates.length; i++) {
        current.add(candidates[i]);
        backtrack(candidates, remaining - candidates[i], i, current, result); // "i" not "i+1" — allows reusing this element
        current.remove(current.size() - 1);
    }
}
```

## 6. Recognizing When Backtracking Applies

| Signal in the problem | Why backtracking fits |
| --- | --- |
| "All possible subsets/permutations/combinations" | Systematic enumeration is exactly what the choose/explore/un-choose template produces |
| "Valid arrangement" with constraints (N-Queens, Sudoku) | Pruning invalid partial states early avoids exploring dead branches fully |
| Word search on a grid | Explore in all directions, mark visited, un-mark on backtrack |
| Problem size is small, explicitly hinting exponential exploration is expected | Backtracking's worst case is exponential — a large input size in the constraints usually rules it out |

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Always undo the choice after recursing | Forgetting to remove/un-mark leaves polluted state for every subsequent sibling branch. |
| Prune before recursing, not after | Checking validity before descending avoids wasted exploration of branches that were already doomed. |
| Copy the current state when recording a result | Adding the live, mutable list/array directly means every later mutation silently changes already-recorded answers. |
| Choose `start`/`used[]`/`remaining` bookkeeping to match the problem's constraint | Subsets need a start index (no reuse before position), permutations need a used-tracker (each element once, any order), combination-sum-with-repetition needs neither restriction. |
| Recognize backtracking's exponential worst case up front | It's the right tool for small search spaces with pruning — not a substitute for a polynomial-time algorithm when one exists. |
