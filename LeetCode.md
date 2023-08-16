# LeetCode Problems in Omega

## 4. Median Of Two Sorted Arrays

Given two sorted arrays `nums1` and `nums2` of size `m` and `n` respectively, return the median of the two sorted arrays.

The overall run time complexity should be `O(log (m+n))`.

**Example 1:**

```
Input: nums1 = [1,3], nums2 = [2]
Output: 2.00000
Explanation: merged array = [1,2,3] and median is 2.
```

**Example 2:**

```
Input: nums1 = [1,2], nums2 = [3,4]
Output: 2.50000
Explanation: merged array = [1,2,3,4] and median is (2 + 3) / 2 = 2.5.
```

**Constraints:**

- `nums1.length == m`
- `nums2.length == n`
- `<= m <= 1000`
- `<= n <= 1000`
- `<= m + n <= 2000`
- `106 <= nums1[i], nums2[i] <= 106`

**Solution**

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

**Example 1**

```
Input: s = "aa", p = "a"
Output: false
Explanation: "a" does not match the entire string "aa".
```

**Example 2**

```
Input: s = "aa", p = "a*"
Output: true
Explanation: '*' means zero or more of the preceding element, 'a'. Therefore, by repeating 'a' once, it becomes "aa".
```

**Example 3**

```
Input: s = "ab", p = ".*"
Output: true
Explanation: ".*" means "zero or more (*) of any character (.)".
```

**Constraints**

- `1 <= s.length <= 20`
- `1 <= p.length <= 20`
- `s` contains only lowercase English letters.
- `p` contains only lowercase English letters, `'.'`, and `'*'`.
- It is guaranteed for each appearance of the character `'*'`, there will be a previous valid character to match.

**Solution**

```clj
(:- (match-char Char Pattern NewPattern) 
    (or (first Pattern ".")
        (first Pattern Char))
    (if (nth Pattern 1 "*")
      (contains [0, 2] DropAmount)
      (= DropAmount 1))
    (drop DropAmount Pattern NewPattern))

(:- (match-str String Pattern NewPattern)
    (cons Char Rest String)
    (match-char Char Pattern Matched)
    (match-str Rest Matched NewPattern))

(:- (regex-matches String Pattern Matches)
    (passes (match-str String Pattern "")
            Matches))
```

## 23. Merge k Sorted Lists

You are given an array of `k` linked-lists `lists`, each linked-list is sorted in ascending order.

*Merge all the linked-lists into one sorted linked-list and return it.*

**Example 1:**

```
Input: lists = [[1,4,5],[1,3,4],[2,6]]
Output: [1,1,2,3,4,4,5,6]
Explanation: The linked-lists are:
[
  1->4->5,
  1->3->4,
  2->6
]
merging them into one sorted list:
1->1->2->3->4->4->5->6
```

**Example 2:**

```
Input: lists = []
Output: []
```

**Example 3:**

```
Input: lists = [[]]
Output: []
```

**Constraints:**

- `k == lists.length`
- `0 <= k <= 104`
- `0 <= lists[i].length <= 500`
- `-104 <= lists[i][j] <= 104`
- `lists[i]` is sorted in ascending order. The sum of `lists[i].length` will not exceed `10^4`.

**Solution:**

```clj
(:- (k-sorted-lists Lists Result)
    (nth Lists I List)
    (assert! (sort List List))
    (fold Lists concat Merged)  
    (sort Merged Result))
```

## 25. Reverse Nodes in k-Group

Given the head of a linked list, reverse the nodes of the list `k` at a time, and return the modified list.

`k` is a positive integer and is less than or equal to the length of the linked list. If the number of nodes is not a multiple of `k` then left-out nodes, in the end, should remain as it is.

You may not alter the values in the list's nodes, only nodes themselves may be changed.


**Example 1:**

<image src="https://assets.leetcode.com/uploads/2020/10/03/reverse_ex1.jpg" />

```
Input: head = [1,2,3,4,5], k = 2
Output: [2,1,4,3,5]
```

**Example 2:**

<image src="https://assets.leetcode.com/uploads/2020/10/03/reverse_ex2.jpg" />

```
Input: head = [1,2,3,4,5], k = 3
Output: [3,2,1,4,5]
```

**Constraints:**

- The number of nodes in the list is `n`.
- `1 <= k <= n <= 5000`
- `0 <= Node.val <= 1000`

**Follow-up:** Can you solve the problem in O(1) extra memory space?

```clj
;; (nth-partition [1 2 3 4 5] 2 0 [1 2])
;; (nth-partition [1 2 3 4 5] 2 1 [3 4])
;; (nth-partition [1 2 3 4 5] 2 2 [5])
;; (reverse [1 2] [2 1])
(:- (reverse-nodes-in-k-group List K Reversed)
    (length List Length)
    (length Reversed Length)
    (nth-partition List K I Partition)
    (if (length Partition K)
      (reverse Partition ReverseNode)
      (= Partition ReverseNode))
    (nth-partition ReverseNode I Reversed))
```

## 30. Substring with Concatenation of All Words

You are given a string `s` and an array of strings `words`. All the strings of `words` are of the same length.

A concatenated substring in `s` is a substring that contains all the strings of any permutation of `words` concatenated.

For example, if `words = ["ab","cd","ef"]`, then `"abcdef"`, `"abefcd"`, `"cdabef"`, `"cdefab"`, `"efabcd"`, and `"efcdab"` are all concatenated strings. `"acdbef"` is not a concatenated substring because it is not the concatenation of any permutation of words.

Return the starting indices of all the concatenated substrings in `s`. You can return the answer in any order.

**Example 1:**

```
Input: s = "barfoothefoobarman", words = ["foo","bar"]
Output: [0,9]
Explanation: Since words.length == 2 and words[i].length == 3, the concatenated substring has to be of length 6.
The substring starting at 0 is "barfoo". It is the concatenation of ["bar","foo"] which is a permutation of words.
The substring starting at 9 is "foobar". It is the concatenation of ["foo","bar"] which is a permutation of words.
The output order does not matter. Returning [9,0] is fine too.
```

**Example 2:**

```
Input: s = "wordgoodgoodgoodbestword", words = ["word","good","best","word"]
Output: []
Explanation: Since words.length == 4 and words[i].length == 4, the concatenated substring has to be of length 16.
There is no substring of length 16 is s that is equal to the concatenation of any permutation of words.
We return an empty array.
```

**Example 3:**

```
Input: s = "barfoofoobarthefoobarman", words = ["bar","foo","the"]
Output: [6,9,12]
Explanation: Since words.length == 3 and words[i].length == 3, the concatenated substring has to be of length 9.
The substring starting at 6 is "foobarthe". It is the concatenation of ["foo","bar","the"] which is a permutation of words.
The substring starting at 9 is "barthefoo". It is the concatenation of ["bar","the","foo"] which is a permutation of words.
The substring starting at 12 is "thefoobar". It is the concatenation of ["the","foo","bar"] which is a permutation of words.
```

**Constraints:**
- `1 <= s.length <= 104`
- `1 <= words.length <= 5000`
- `1 <= words[i].length <= 30`
- `s` and `words[i]` consist of lowercase English letters.

**Solution:**

```clj
(:- (concat-all Words ConcatAll)
    (if (empty? Words)
      (= ConcatAll "")
      (and (nth Words I Word)
           (remove Words I Rest)
           (concat-all Rest ConcatRest)
           (concat Word ConcatRest ConcatAll))))

(:- (substring-indices S Words Indices)
    (concat-all Words ConcatAll)
    (slice S I _ ConcatAll)
    (list Indices I))
```


## 32. Longest Valid Parentheses

Given a string containing just the characters `'('` and `')'`, return the length of the longest valid (well-formed) parentheses
substring.

**Example 1:**

```
Input: s = "(()"
Output: 2
Explanation: The longest valid parentheses substring is "()".
```

**Example 2:**

```
Input: s = ")()())"
Output: 4
Explanation: The longest valid parentheses substring is "()()".
```

**Example 3:**

```
Input: s = ""
Output: 0
```

**Constraints:**
- `0 <= s.length <= 3 * 104`
- `s[i]` is `'('`, or `')'`.

**Solution:**

```clj
(:- (paren-balance String N)
    (contains String Char)
    (match Char
      "(" (= Balance 1)
      "(" (= Balance -1))
    (sum Balance N))

(:- (longest-matching-parens String N)
    (slice String _ _ Slice)
    (paren-balance Slice 0)
    (length Slice Length)
    (max Length N))
```

## 37. Sudoku Solver

Write a program to solve a Sudoku puzzle by filling the empty cells.

A sudoku solution must satisfy all of the following rules:

- Each of the digits 1-9 must occur exactly once in each row.
- Each of the digits 1-9 must occur exactly once in each column.
- Each of the digits 1-9 must occur exactly once in each of the 9 3x3 sub-boxes of the grid.

The `'.'` character indicates empty cells.

 
**Example 1:**

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/f/ff/Sudoku-by-L2G-20050714.svg/250px-Sudoku-by-L2G-20050714.svg.png" />

```
Input: board = [["5","3",".",".","7",".",".",".","."],["6",".",".","1","9","5",".",".","."],[".","9","8",".",".",".",".","6","."],["8",".",".",".","6",".",".",".","3"],["4",".",".","8",".","3",".",".","1"],["7",".",".",".","2",".",".",".","6"],[".","6",".",".",".",".","2","8","."],[".",".",".","4","1","9",".",".","5"],[".",".",".",".","8",".",".","7","9"]]
Output: [["5","3","4","6","7","8","9","1","2"],["6","7","2","1","9","5","3","4","8"],["1","9","8","3","4","2","5","6","7"],["8","5","9","7","6","1","4","2","3"],["4","2","6","8","5","3","7","9","1"],["7","1","3","9","2","4","8","5","6"],["9","6","1","5","3","7","2","8","4"],["2","8","7","4","1","9","6","3","5"],["3","4","5","2","8","6","1","7","9"]]
Explanation: The input board is shown above and the only valid solution is shown below:
```

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/3/31/Sudoku-by-L2G-20050714_solution.svg/250px-Sudoku-by-L2G-20050714_solution.svg.png" />

**Constraints:**

- `board.length == 9`
- `board[i].length == 9`
- `board[i][j]` is a digit or `'.'`.
- It is guaranteed that the input board has only one solution.

```clj
(:- (bind-char Char Binding)
    (if (= Char ".")
      (pass)
      (= Char Binding)))

(:- (soduko-array Array)
    (contains Array Field)
    (Integer/from-string Field IntField)
    (>= IntField 0)
    (<= IntField 9)
    (list-unique Array))

(:- (soduko Board Solution)
    (Matrix/row Board Ri Row)
    (Matrix/row Solution ri SolRow)
    (map Row bind-char SolRow)
    (soduko-array SolRow)
    (Matrix/column Board Ci Column)
    (Matrix/column Solution ri SolCol)
    (map Column bind-char SolCol) 
    (soduko-array SolCol)
    (Matrix/partition Board 3 3 Bi Box)
    (flatten Box FlatBox)
    (Matrix/partition Solution 3 3 Bi SolBox)
    (flatten SolBox FlatSolBox)
    (soduko-array FlatSolBox))
```

## 41. First Missing Positive

Given an unsorted integer array nums, return the smallest missing positive integer.

You must implement an algorithm that runs in `O(n)` time and uses `O(1)` auxiliary space.

**Example 1:**

```
Input: nums = [1,2,0]
Output: 3
Explanation: The numbers in the range [1,2] are all in the array.
```

**Example 2:**

```
Input: nums = [3,4,-1,1]
Output: 2
Explanation: 1 is in the array but 2 is missing.
```

**Example 3:**

```
Input: nums = [7,8,9,11,12]
Output: 1
Explanation: The smallest positive integer 1 is missing.
```

**Constraints:**

- `1 <= nums.length <= 105`
- `-231 <= nums[i] <= 231 - 1`

```clj
(:- (smallest-missing-positive Nums Smallest)
    (contains Nums Num)
    (not (contains Nums Positive))
    (> Positive 0)
    (int Positive)
    (int Num)
    (min Positive Smallest))
```

## 42. Trapping Rain Water

Given `n` non-negative integers representing an elevation map where the width of each bar is `1`, compute how much water it can trap after raining.

**Example 1:**

<img src="https://assets.leetcode.com/uploads/2018/10/22/rainwatertrap.png" />

```
Input: height = [0,1,0,2,1,0,1,3,2,1,2,1]
Output: 6
Explanation: The above elevation map (black section) is represented by array [0,1,0,2,1,0,1,3,2,1,2,1]. In this case, 6 units of rain water (blue section) are being trapped.
```

**Example 2:**

```
Input: height = [4,2,0,3,2,5]
Output: 9
```

**Constraints:**

- `n == height.length`
- `1 <= n <= 2 * 104`
- `0 <= height[i] <= 105`

```clj
(:- (all-water-held Heights WaterHeld)
    (nth Heights I _)
    (take LeftHeights I WithoutLeft)
    (cons Height RightHeights WithoutLeft)
    (list-max LeftHeights MaxLeft)
    (list-max RightHeights MaxRight)
    (min MaxLeft MaxRight WaterHeight)
    (- WaterHeight Height WaterHeldAtHeight)
    (sum WaterHeldAtHeight WaterHeld))
```
