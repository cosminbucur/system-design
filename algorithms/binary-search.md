Binary search is the pattern for finding an answer inside a sorted (or monotonic) search space in `O(log n)` by repeatedly cutting the space in half, instead of scanning it linearly. The key requirement isn't "the array is sorted" specifically — it's that the space has some monotonic property that lets you always know which half the answer must be in.

## 1. The Core Idea

At every step, check the middle. If it's not the answer, the sorted property tells you which entire half can be thrown away — the answer, if it exists, is guaranteed to be in the remaining half.

```java
int binarySearch(int[] sortedArr, int target) {
    int lo = 0, hi = sortedArr.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2; // avoids integer overflow vs. (lo + hi) / 2
        if (sortedArr[mid] == target) return mid;
        if (sortedArr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1; // not found
}
```

Each comparison eliminates half the remaining search space, so a million-element array takes about 20 comparisons worst case — this is why `O(log n)` scales so much better than `O(n)` as data grows.

## 2. Binary Search on the Answer, Not Just an Array

The most powerful realization about this pattern: binary search doesn't require an actual sorted array at all — it works on any range of possible answers where you can check "is this candidate answer good enough" and that check is monotonic (true for everything above/below some threshold).

```java
// Find the minimum "speed" to eat all bananas within h hours
int minEatingSpeed(int[] piles, int h) {
    int lo = 1, hi = Arrays.stream(piles).max().getAsInt();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (canFinish(piles, mid, h)) {
            hi = mid; // this speed works — try to go slower
        } else {
            lo = mid + 1; // too slow — need to go faster
        }
    }
    return lo;
}

boolean canFinish(int[] piles, int speed, int h) {
    long hoursNeeded = 0;
    for (int pile : piles) hoursNeeded += Math.ceil((double) pile / speed);
    return hoursNeeded <= h;
}
```

There's no sorted array here at all — the "search space" is the range of possible speeds, and it's monotonic because any speed faster than a working speed also works. Recognizing this shape ("minimize/maximize X such that condition Y holds") is what unlocks binary search on problems that don't look like search problems at first glance.

## 3. Finding a Boundary, Not an Exact Match

A very common variant: instead of "does this exact value exist," the question is "what's the first/last position where some condition becomes true" — the classic "find first" and "find last" boundary search.

```java
// Find the leftmost index where target could be inserted to keep the array sorted
int lowerBound(int[] sortedArr, int target) {
    int lo = 0, hi = sortedArr.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (sortedArr[mid] < target) lo = mid + 1;
        else hi = mid; // keep searching left for an earlier valid position
    }
    return lo;
}
```

Java's own `Collections.binarySearch` and `Arrays.binarySearch` only tell you *whether* an exact match exists (returning a negative "insertion point" encoding if not) — implementing the boundary search by hand is necessary whenever the actual position (not just presence) matters.

## 4. Binary Search on a Rotated Sorted Array

A sorted array that's been rotated (e.g. `[4,5,6,7,0,1,2]`) is still binary-searchable — at every midpoint, one of the two halves is still guaranteed to be properly sorted, and that's enough to decide which side to search.

```java
int searchRotated(int[] arr, int target) {
    int lo = 0, hi = arr.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) return mid;
        if (arr[lo] <= arr[mid]) { // left half is properly sorted
            if (arr[lo] <= target && target < arr[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else { // right half is properly sorted
            if (arr[mid] < target && target <= arr[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    return -1;
}
```

## 5. Recognizing When Binary Search Applies

| Signal in the problem | Why binary search fits |
| --- | --- |
| "Sorted array, find X" | The direct, classic case |
| "Minimize/maximize X such that condition holds" | Binary search on the answer space, not the array |
| "Find first/last position where..." | Boundary search variant |
| Time limit strongly implies faster than `O(n)` on a large input | A very common hint that the intended solution is `O(log n)` |

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Use `lo + (hi - lo) / 2` instead of `(lo + hi) / 2` | Avoids integer overflow when `lo` and `hi` are both large. |
| Ask "is there a monotonic yes/no check" before assuming binary search doesn't apply | It works on any monotonic answer space, not just literal sorted arrays. |
| Be precise about loop invariants (`<=` vs. `<`, `mid` vs. `mid - 1`/`mid + 1`) | Off-by-one errors here are the single most common source of infinite loops or missed answers. |
| Use boundary search (lower/upper bound) when position matters, not just existence | `Arrays.binarySearch` only confirms presence — hand-write the boundary variant for "first/last occurrence" questions. |
| Verify a "sorted with a twist" (rotated, nearly-sorted) array still has a valid half at every midpoint | If one half is always properly ordered, binary search still applies even though the whole array isn't sorted. |
