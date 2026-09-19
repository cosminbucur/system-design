Two pointers is the pattern for scanning a sorted (or otherwise structured) sequence with two indices moving toward each other, or at different speeds, instead of checking every pair with a nested loop. It's the single biggest lever for turning an `O(n²)` "check every pair" solution into `O(n)`.

## 1. The Core Idea: Two Indices Instead of a Nested Loop

Whenever the brute-force instinct is "compare every element with every other element," and the array is sorted, two pointers usually replaces that nested loop with one pass.

```java
// Does a sorted array contain a pair summing to target?
boolean hasPairWithSum(int[] sortedArr, int target) {
    int left = 0, right = sortedArr.length - 1;
    while (left < right) {
        int sum = sortedArr[left] + sortedArr[right];
        if (sum == target) return true;
        if (sum < target) left++;  // sum too small — need a bigger left value
        else right--;               // sum too big — need a smaller right value
    }
    return false;
}
```

The reason this works and doesn't miss any pair: because the array is sorted, moving `left` forward only increases the sum, and moving `right` backward only decreases it — so at every step there's exactly one correct direction to move, and no pair is ever skipped over.

## 2. Opposite-Direction Pointers

The most common variant: one pointer starts at the beginning, the other at the end, and they move toward each other based on some comparison.

```java
// Is a string a palindrome, ignoring non-alphanumeric characters?
boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (!Character.isLetterOrDigit(s.charAt(left))) { left++; continue; }
        if (!Character.isLetterOrDigit(s.charAt(right))) { right--; continue; }
        if (Character.toLowerCase(s.charAt(left)) != Character.toLowerCase(s.charAt(right))) return false;
        left++; right--;
    }
    return true;
}
```

## 3. Same-Direction Pointers (Fast/Slow)

A second variant: both pointers move forward, but at different rates or triggered by different conditions — common for in-place array modification or cycle detection.

```java
// Remove duplicates from a sorted array in-place, return new length
int removeDuplicates(int[] arr) {
    if (arr.length == 0) return 0;
    int slow = 0; // last position of a unique element
    for (int fast = 1; fast < arr.length; fast++) {
        if (arr[fast] != arr[slow]) {
            slow++;
            arr[slow] = arr[fast];
        }
    }
    return slow + 1;
}
```

The fast/slow variant also solves cycle detection in a linked list (covered separately) — a slow pointer moving one step and a fast pointer moving two steps will meet if and only if a cycle exists.

## 4. Three Pointers: Extending the Idea

Some problems (like 3Sum) fix one element and run two-pointers on the rest — the outer loop picks a fixed index, the inner two-pointer scan handles the remaining pair.

```java
// 3Sum: find all unique triplets that sum to zero
List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> result = new ArrayList<>();
    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i - 1]) continue; // skip duplicate fixed values
        int left = i + 1, right = nums.length - 1;
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum == 0) {
                result.add(List.of(nums[i], nums[left], nums[right]));
                left++; right--;
                while (left < right && nums[left] == nums[left - 1]) left++; // skip duplicates
            } else if (sum < 0) left++;
            else right--;
        }
    }
    return result;
}
```

## 5. Recognizing When Two Pointers Applies

| Signal in the problem | Why two pointers fits |
| --- | --- |
| Array/string is sorted, or can be sorted without losing needed info | Moving pointers in one deterministic direction never misses the answer |
| "Pair," "triplet," or "closest sum" in a sorted structure | Fixes one or more indices and scans the rest in one pass |
| Palindrome checks | Naturally compares from both ends inward |
| Cycle detection, "middle of a list" | Fast/slow same-direction variant |

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Sort first if the problem doesn't require preserving original order | Two pointers relies on sorted order to guarantee a correct move direction at every step. |
| Move only the pointer that can't possibly be part of a better answer | In the sum-comparison variant, moving the "wrong" pointer risks skipping the correct pair. |
| Skip duplicate values explicitly when the problem asks for unique results | Otherwise the same triplet/pair gets added multiple times (see the 3Sum duplicate-skip lines). |
| Recognize fast/slow as a distinct sub-pattern from opposite-direction | Fast/slow solves cycle detection and "find the middle," not sum-matching — don't force one variant to solve the other's problem. |
| Default to two pointers before a nested loop on sorted data | It's almost always the `O(n)` replacement for an `O(n²)` brute-force pair search. |
