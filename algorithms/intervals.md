Interval problems deal with ranges (`[start, end]`) — meetings, bookings, ranges on a number line — and almost every one of them starts with the same first move: sort the intervals by start time. Once sorted, most of these problems become a single pass comparing each interval to the one before it.

## 1. Why Sorting First Almost Always Helps

Unsorted intervals could overlap in any order, forcing you to compare every pair (`O(n²)`). Sorted by start time, any interval that could overlap with the current one must be adjacent to it in the sorted order — so a single left-to-right pass is enough.

```java
Arrays.sort(intervals, (a, b) -> a[0] - b[0]); // sort by start time — the near-universal first line
```

## 2. Merge Overlapping Intervals

The canonical interval problem: given a list of intervals, merge every pair that overlaps into one.

```java
int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
    List<int[]> result = new ArrayList<>();
    for (int[] interval : intervals) {
        if (result.isEmpty() || result.get(result.size() - 1)[1] < interval[0]) {
            result.add(interval); // no overlap with the last merged interval — start a new one
        } else {
            result.get(result.size() - 1)[1] = Math.max(result.get(result.size() - 1)[1], interval[1]); // extend it
        }
    }
    return result.toArray(new int[0][]);
}
```

The overlap check (`lastEnd < currentStart` means no overlap) is the one piece of logic every interval problem variant reuses in some form — internalizing that single comparison unlocks most of this category.

## 3. Insert a New Interval Into a Sorted List

A variant of merging: given an already-merged, sorted list of intervals, insert a new one and merge whatever it now overlaps with.

```java
int[][] insert(int[][] intervals, int[] newInterval) {
    List<int[]> result = new ArrayList<>();
    int i = 0;
    while (i < intervals.length && intervals[i][1] < newInterval[0]) {
        result.add(intervals[i++]); // entirely before newInterval — no overlap possible
    }
    while (i < intervals.length && intervals[i][0] <= newInterval[1]) {
        newInterval[0] = Math.min(newInterval[0], intervals[i][0]); // merge into newInterval
        newInterval[1] = Math.max(newInterval[1], intervals[i][1]);
        i++;
    }
    result.add(newInterval);
    while (i < intervals.length) {
        result.add(intervals[i++]); // entirely after newInterval — no overlap possible
    }
    return result.toArray(new int[0][]);
}
```

## 4. Detecting Any Overlap at All

The simplest interval question — can all these intervals coexist without any overlap (e.g., can one person attend every meeting)?

```java
boolean canAttendAllMeetings(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] < intervals[i - 1][1]) return false; // this one starts before the previous one ends
    }
    return true;
}
```

## 5. Minimum Resources Needed (Meeting Rooms II)

A harder variant: instead of "can one resource handle everything," ask "what's the minimum number of resources (rooms) needed" — solved by tracking start and end events separately and sweeping through time.

```java
int minMeetingRooms(int[][] intervals) {
    int[] starts = new int[intervals.length];
    int[] ends = new int[intervals.length];
    for (int i = 0; i < intervals.length; i++) {
        starts[i] = intervals[i][0];
        ends[i] = intervals[i][1];
    }
    Arrays.sort(starts);
    Arrays.sort(ends);

    int rooms = 0, maxRooms = 0, endPointer = 0;
    for (int start : starts) {
        while (endPointer < ends.length && ends[endPointer] <= start) {
            rooms--; // a meeting ended before this one starts — free up its room
            endPointer++;
        }
        rooms++; // this meeting needs a room
        maxRooms = Math.max(maxRooms, rooms);
    }
    return maxRooms;
}
```

This is really a sweep-line technique: sorted starts and ends are two separate timelines, and walking through them in time order tells you exactly how many meetings are simultaneously active at any point — the peak of that count is the answer.

## 6. Recognizing When Interval Techniques Apply

| Signal in the problem | Likely technique |
| --- | --- |
| "Merge overlapping ranges" | Sort by start, single pass comparing to the last merged interval |
| "Can one resource handle all of these" | Sort by start, check each against the immediately preceding interval |
| "Minimum number of resources needed simultaneously" | Sweep-line: separate sorted start/end arrays, track a running count |
| "Insert a new range into an existing sorted set" | Three-part scan: before, overlapping (merge), after |

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Sort by start time as the default first step | Almost every interval problem becomes a simple linear scan once sorted; almost none are tractable unsorted. |
| Internalize the overlap check (`lastEnd < currentStart` means no overlap) | This one comparison, inverted or adapted, solves the majority of interval sub-problems. |
| Use separate sorted start/end arrays for "how many active at once" problems | The sweep-line technique needs the two event types (start, end) processed in time order, not paired per-interval. |
| Watch for `<` vs `<=` at interval boundaries | Whether touching endpoints count as overlapping (`[1,3]` and `[3,5]`) is a common source of off-by-one bugs — check the problem's exact definition. |
| Recognize interval scheduling as a variant of the greedy pattern | Many interval problems (like maximum non-overlapping intervals) are solved by a greedy choice after sorting, not exhaustive search. |
