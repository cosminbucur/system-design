Bit manipulation works directly on a number's binary representation instead of its decimal value — useful whenever a problem is really about individual on/off flags, or when an operation can be done in `O(1)` at the bit level instead of `O(n)` any other way.

## 1. The Core Operators

| Operator | Symbol | What it does |
| --- | --- | --- |
| AND | `&` | 1 only where both bits are 1 — used to check or clear specific bits |
| OR | `\|` | 1 where either bit is 1 — used to set specific bits |
| XOR | `^` | 1 where bits differ — used to toggle bits or find differences |
| NOT | `~` | Flips every bit |
| Left shift | `<<` | Shifts bits left, equivalent to multiplying by `2^n` |
| Right shift | `>>` | Shifts bits right, equivalent to dividing by `2^n` (rounding toward negative infinity) |

```java
int a = 5;  // 0101
int b = 3;  // 0011

int and = a & b;  // 0001 = 1
int or  = a | b;  // 0111 = 7
int xor = a ^ b;  // 0110 = 6
int shiftLeft = a << 1;  // 1010 = 10 — same as a * 2
```

## 2. Checking, Setting, and Clearing a Specific Bit

The recurring low-level template most bit-manipulation problems build on.

```java
boolean isBitSet(int num, int i) {
    return (num & (1 << i)) != 0; // shift a single 1 into position i, AND checks if that position is also 1 in num
}

int setBit(int num, int i) {
    return num | (1 << i); // OR-ing with a single 1 at position i forces that bit on, leaves everything else unchanged
}

int clearBit(int num, int i) {
    return num & ~(1 << i); // AND-ing with everything-except-position-i forces that bit off, leaves the rest unchanged
}
```

## 3. XOR's Special Property: Self-Cancellation

`x ^ x = 0` and `x ^ 0 = x` — XOR-ing a value with itself cancels it out, which is the basis for a surprisingly large number of clever bit tricks.

```java
// Find the single number that appears once, when every other number appears exactly twice
int singleNumber(int[] nums) {
    int result = 0;
    for (int num : nums) {
        result ^= num; // every paired number cancels itself out; only the unpaired one survives
    }
    return result;
}
```

```java
// Swap two variables without a temporary variable, using XOR
a = a ^ b;
b = a ^ b; // b becomes original a
a = a ^ b; // a becomes original b
```

## 4. Counting Set Bits, and the `n & (n-1)` Trick

`n & (n - 1)` clears the lowest set bit of `n` — a compact trick used both for counting set bits and for checking if a number is a power of two.

```java
int countSetBits(int n) {
    int count = 0;
    while (n != 0) {
        n &= (n - 1); // clears the lowest set bit each iteration
        count++;
    }
    return count; // loop runs once per set bit, not once per total bit — faster than checking every bit position
}

boolean isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0; // a power of two has exactly one set bit; clearing it leaves 0
}
```

Java's built-in `Integer.bitCount(n)` does the same job as `countSetBits` and should be preferred in real code — the manual version above is worth understanding because the same `n & (n-1)` idea reappears in other bit tricks.

## 5. Bitmasking: Representing a Set as an Integer

When there's a small, fixed number of possible items (say, up to 32), a single integer can represent an entire subset — each bit position is one item's "included or not" flag. This turns subset-related DP/backtracking problems into simple integer operations instead of manipulating actual sets.

```java
// Iterate over every subset of n items, represented as bitmasks from 0 to 2^n - 1
for (int mask = 0; mask < (1 << n); mask++) {
    for (int i = 0; i < n; i++) {
        if ((mask & (1 << i)) != 0) {
            // item i is included in this subset
        }
    }
}
```

This is the standard technique behind "traveling salesman with DP" and similar small-`n` combinatorial problems — `dp[mask]` represents "the best answer for exactly this subset of items," and transitions between masks are just bit operations.

## 6. Recognizing When Bit Manipulation Applies

| Signal in the problem | Why bit manipulation fits |
| --- | --- |
| "Find the number that appears once/odd number of times" | XOR's self-cancellation solves this in one pass, `O(1)` extra space |
| "Is this a power of two/four" | The `n & (n-1)` trick checks this in `O(1)` |
| A small fixed set of flags/options (≤ 32 items) | Represent the whole subset as one integer — bitmask DP |
| Constant-time set/clear/check of individual flags | Direct bit operations beat a boolean array for compactness and speed |

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Prefer Java's built-in bit methods over hand-rolled loops in real code | `Integer.bitCount`, `Integer.numberOfTrailingZeros`, etc. are optimized and clearer than reimplementing them. |
| Use `1 << i` (not a hardcoded power of two) to reference a specific bit position | Keeps the code readable and correct regardless of which position is being referenced. |
| Remember XOR cancels pairs, not general duplicates | The "find the single number" trick only works when every other number appears an even number of times — verify this actually matches the problem before applying it. |
| Use bitmasking only when the item count is genuinely small | An `int` bitmask tops out at 32 items (a `long` at 64) — this technique doesn't scale past that without switching to a `BitSet` or similar. |
| Watch for the difference between `>>` and `>>>` in Java | `>>` sign-extends (preserves the sign bit), `>>>` doesn't — using the wrong one silently corrupts results on negative numbers. |
