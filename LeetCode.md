# LeetCode Problems in Omega

## 4. Median Of Two Sorted Arrays

Given two sorted arrays `nums1` and `nums2` of size `m` and `n` respectively, return the median of the two sorted arrays.

The overall run time complexity should be `O(log (m+n))`.

### Example 1:

```
Input: nums1 = [1,3], nums2 = [2]
Output: 2.00000
Explanation: merged array = [1,2,3] and median is 2.
```

### Example 2:

```
Input: nums1 = [1,2], nums2 = [3,4]
Output: 2.50000
Explanation: merged array = [1,2,3,4] and median is (2 + 3) / 2 = 2.5.
```

### Constraints:

- `nums1.length == m`
- `nums2.length == n`
- `<= m <= 1000`
- `<= n <= 1000`
- `<= m + n <= 2000`
- `106 <= nums1[i], nums2[i] <= 106`

### Solution

```clj
(:- (median-two-sorted-arrays Nums1 Nums2 Median)
    (assert!
     (sorted Nums1 Nums1)
     (sorted Nums2 Nums2))
    (concat Nums1 Nums2 Both)
    (sort Both Sorted)
    (median Sorted Median))
```

## 10. Regular Expression Matching

Given an input string `s` and a pattern `p`, implement regular expression matching with support for `'.'` and `'*'` where:

- `'.'` Matches any single character.​​​​
- `'*'` Matches zero or more of the preceding element.

The matching should cover the entire input string (not partial).

### Example 1

```
Input: s = "aa", p = "a"
Output: false
Explanation: "a" does not match the entire string "aa".
```

### Example 2

```
Input: s = "aa", p = "a*"
Output: true
Explanation: '*' means zero or more of the preceding element, 'a'. Therefore, by repeating 'a' once, it becomes "aa".
```

### Example 3

```
Input: s = "ab", p = ".*"
Output: true
Explanation: ".*" means "zero or more (*) of any character (.)".
```

### Constraints

- `1 <= s.length <= 20`
- `1 <= p.length <= 20`
- `s` contains only lowercase English letters.
- `p` contains only lowercase English letters, `'.'`, and `'*'`.
- It is guaranteed for each appearance of the character `'*'`, there will be a previous valid character to match.

### Solution

```clj
(:- (regex-match-char Char Pattern Matches) 
    (or (first Pattern ".")
        (first Pattern Char)))

(:- (repeating Pattern)
    (nth Pattern 1 "*"))

(:- (regex-consume-char String Pattern NewPattern NewString)  
    (first String Char)
    (if (regex-match-char Char Pattern)
      (and (rest String NewString)
           (if (repeating Pattern)
             (= Pattern NewPattern)
             (rest Pattern NewPattern))) 
      (and (= String NewString)
           (drop Pattern 2 NewPattern))))

(:- (regex-finished Pattern String)
    (empty String)
    (or (repeating Pattern)
        (empty Pattern)))

(:- (regex-match String Pattern)
    (or (regex-finished String Pattern)
        (and (regex-consume-char String Pattern NewPattern NewString)
             (regex-match NewString NewPattern))))

(:- (regex-match-bool String Pattern Matches)
    (passes (regex-match String Pattern)
            Matches))
```