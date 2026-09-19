Math and geometry problems don't share one single technique the way the other patterns do — instead, they share a common trap: an approach that looks correct can silently fail on overflow, floating-point imprecision, or an edge case (zero, negative numbers, a degenerate shape) that's easy to overlook. Recognizing the handful of recurring building blocks below covers most of what shows up.

## 1. Overflow: The Silent Killer of Otherwise-Correct Math Code

Java's `int` maxes out at ~2.1 billion — a calculation that's logically correct can still produce a wrong answer if an intermediate value exceeds that, silently wrapping around instead of throwing an error.

```java
// BAD: mid can overflow if lo and hi are both close to Integer.MAX_VALUE
int mid = (lo + hi) / 2;

// GOOD: this ordering avoids the intermediate sum ever exceeding the range
int mid = lo + (hi - lo) / 2;

// Multiplying two large ints: cast to long BEFORE multiplying, not after
long product = (long) a * b; // casting only 'a' is enough — it forces the whole expression to widen
```

The rule that prevents almost every overflow bug: whenever multiplying or adding values that could individually be large, deliberately widen to `long` (or check bounds) before the operation, not after it's already wrapped around.

## 2. GCD and LCM

Greatest common divisor and least common multiple show up constantly in problems about ratios, simplifying fractions, or finding a common cycle length.

```java
int gcd(int a, int b) {
    return b == 0 ? a : gcd(b, a % b); // Euclidean algorithm — O(log(min(a,b)))
}

long lcm(int a, int b) {
    return (long) a * b / gcd(a, b); // cast to long first — the product can overflow before the division shrinks it back down
}
```

## 3. Prime Numbers: Sieve of Eratosthenes

Checking primality of one number is `O(√n)`; finding *all* primes up to `n` is far more efficient done once with a sieve than by checking each number individually.

```java
boolean[] sieveOfEratosthenes(int n) {
    boolean[] isPrime = new boolean[n + 1];
    Arrays.fill(isPrime, true);
    isPrime[0] = isPrime[1] = false;
    for (int i = 2; (long) i * i <= n; i++) {
        if (isPrime[i]) {
            for (int multiple = i * i; multiple <= n; multiple += i) {
                isPrime[multiple] = false; // every multiple of a prime is, by definition, not prime
            }
        }
    }
    return isPrime;
}
```

Starting the inner loop at `i * i` (not `2 * i`) is the key efficiency trick — every smaller multiple of `i` was already marked false by a smaller prime factor earlier in the sieve, so re-marking them again would be wasted work.

## 4. Fast Exponentiation

Computing `base^exp` by multiplying in a loop is `O(exp)` — repeated squaring gets the same answer in `O(log exp)` by halving the exponent at each step.

```java
long power(long base, long exp, long mod) {
    long result = 1;
    base %= mod;
    while (exp > 0) {
        if ((exp & 1) == 1) { // if the current lowest bit of exp is 1, fold this power of base into the result
            result = (result * base) % mod;
        }
        base = (base * base) % mod; // square the base each step — this is what halves the remaining exponent
        exp >>= 1;
    }
    return result;
}
```

Taking `% mod` at every multiplication (not just at the end) is what keeps intermediate values from overflowing on large exponents — this pattern is standard whenever a problem asks for "the answer modulo some large number."

## 5. Geometry: Distance, Area, and Orientation

Most geometry problems reduce to a small set of formulas applied carefully.

```java
// Euclidean distance between two points
double distance(int[] p1, int[] p2) {
    double dx = p1[0] - p2[0], dy = p1[1] - p2[1];
    return Math.sqrt(dx * dx + dy * dy);
}

// Cross product — tells you the turn direction (clockwise, counter-clockwise, or straight) at point b
// going from a -> b -> c
long crossProduct(int[] a, int[] b, int[] c) {
    return (long) (b[0] - a[0]) * (c[1] - a[1]) - (long) (b[1] - a[1]) * (c[0] - a[0]);
    // positive: counter-clockwise turn, negative: clockwise turn, zero: collinear
}
```

The cross product's sign is the workhorse behind many geometry problems (convex hull, checking if a point is inside a polygon, determining if a set of points is sorted around a center) — recognizing "this is really asking about turn direction" often turns an intimidating geometry problem into a simple sign check.

## 6. Avoiding Floating-Point Comparison Bugs

Floating-point arithmetic is inherently imprecise — comparing two computed doubles with `==` frequently fails even when the values are mathematically equal, because of how the values are rounded internally.

```java
// BAD: this comparison can fail even for values that are mathematically equal
if (computedDistance == expectedDistance) { ... }

// GOOD: compare within a small tolerance (epsilon)
double EPSILON = 1e-9;
if (Math.abs(computedDistance - expectedDistance) < EPSILON) { ... }
```

Where possible, avoid floating-point entirely by staying in integer/rational arithmetic (e.g., comparing squared distances instead of taking a square root, or cross-multiplying fractions instead of dividing) — this sidesteps the precision problem completely rather than just tolerating it.

## 7. Recognizing Which Building Block Applies

| Signal in the problem | Likely building block |
| --- | --- |
| "Simplify a fraction," "common cycle length" | GCD / LCM |
| "Count/find primes up to n" | Sieve of Eratosthenes |
| "Result modulo 10^9+7," large exponent | Fast exponentiation with modular arithmetic |
| Points, turns, polygons, "is this point inside/on the line" | Cross product / orientation check |
| Any comparison between computed decimal values | Epsilon-tolerant comparison, or avoid floating-point entirely |

## 8. Best Practices

| Practice | Recommendation |
| --- | --- |
| Widen to `long` before multiplying or adding values that could be large | Casting after the operation is too late — the overflow already happened in `int` arithmetic. |
| Never compare floating-point values with `==` | Use an epsilon tolerance, or better, restructure the problem to stay in integer arithmetic entirely. |
| Take the modulus at every intermediate step, not just at the end | Prevents overflow in fast-exponentiation and similar large-number problems. |
| Recognize the cross product's sign as an orientation/turn-direction check | Unlocks a large class of geometry problems (convex hull, point-in-polygon) without needing trigonometry. |
| Start a sieve's inner loop at `i * i`, not `2 * i` | Smaller multiples were already eliminated by smaller prime factors — re-marking them wastes work. |
