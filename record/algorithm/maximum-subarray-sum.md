# Maximum Subarray Sum

数学+分治, 就问你怕不怕! 算法详情请查看`HackerRank`:[原文 link](https://www.hackerrank.com/challenges/maximum-subarray-sum/problem)

## Description

给你一个大小为 N 的数组和另外一个整数 M。你的目标是找到每个子数组的和对 M 取余数的最大值。

子数组是指原数组的任意连续元素的子集。

注意你需要找到 max(子数组的和）%M ，其中一共有`N*(N + 1)/2`个可能的子数组。

### Input Format

第一行包含 T，后面的测试数据组数。 每组测试数据包含恰好 2 行。 每组测试数据的第一行包含 2 个空格分隔的整数和，数组的大小和模 M。

第二行包含 N 个空格分隔的整数，表示数组中的每个元素。

### Constraints

- 2 ≤ N ≤ 105

- 1 ≤ M ≤ 1014

- 1 ≤ 数组元素的值 ≤ 1018

- 2 ≤ 所有测试数据中 N 的总和 ≤ 500000

### Output Format

对每组测试数据，在新的一行输出上面要求的最大值。

## Sample

### Sample Input

```java
1
5 7
3 3 9 9 5
```

### Sample Output

```java
6
```

### Explanation

对 7 取余数能达到的最大的和是 6，我们可以用数组中的第一个元素和第二个元素得到 6。

## Analysis

[LinkUrl](https://www.quora.com/What-is-the-logic-used-in-the-HackerRank-Maximise-Sum-problem)

I hope I could explain it in a more simple way.! Before getting into the correct solution, we all know there is a `O(n^2)` algorithm to solve it. We call that Brutal Force solution(BF)

To solve it in a better way, the problem requires some knowledge of modular arithmetic. Here are some basic formula for this problem:

```java
(𝑎+𝑏)%𝑀=(𝑎%𝑀+𝑏%𝑀)%𝑀 --- 1
(𝑎−𝑏)%𝑀=(𝑎%𝑀−𝑏%𝑀)%𝑀 --- 2
```

This link: Modular arithmetic, explains 1, 2 in a straightforward way.

Now, let's solve the problem with the simple math formula!

Usually, a great many problems related to "subarray computation" could be solved with prefix array, which saves time for repeating computation.

Define:

```java
𝑝𝑟𝑒𝑓𝑖𝑥[𝑛]=(𝑎[0]+𝑎[1]+...+𝑎[𝑛])%𝑀
```

To construct a prefix table, we could use the following code. (It is a generalization of the 1, 2 formula I mentioned before)

```java
int curr = 0;
for(int i = 0; i < n; i ++) {
  curr = (arr[i] % M + curr) % M;
  prefix[i] = curr;
}
```

The we have to find a subarray. Any subarray can be expressed in the following way:

```java
𝑠𝑢𝑚𝑀𝑜𝑑𝑢𝑙𝑎𝑟[𝑖,𝑗]=(𝑝𝑟𝑒𝑓𝑖𝑥[𝑗]−𝑝𝑟𝑒𝑓𝑖𝑥[𝑖−1]+𝑀)%𝑀 --3
```

which is very simple and efficient. The correctness of 3 require some math based on 1 and 2\. And 3 is the key to the efficient solution!

Let's write the solution of the BF algorithm first:

```java
int ret = 0;
for(int i = 0; i < n; i ++) {
  for(int j = i-1; j >= 0; j --) {
    ret = max(ret, (prefix[i] - prefix[j] + M) % M)
  }
  ret = max(ret, prefix[i]); // Don't forget sum from beginning.
}
```

Okie! This code is simple, however, it did some non-sense job! Why? if `prefix[j]` is smaller than `prefix[i]`, in the previous code, there is no need to calculate, because for `prefix[j] < prefix[i]`, we have:

```java
(𝑝𝑟𝑒𝑓𝑖𝑥[𝑖]−𝑝𝑟𝑒𝑓𝑖𝑥[𝑗]+𝑀)%𝑀=𝑝𝑟𝑒𝑓𝑖𝑥[𝑖]−𝑝𝑟𝑒𝑓𝑖𝑥[𝑗]≤𝑝𝑟𝑒𝑓𝑖𝑥[𝑖]
```

So we just need to find those `j`, which `prefix[j] > prefix[i]`. Ok, let's just linear scan all `j` before i and find the ones that is smaller than `prefix[i]`. But that makes no difference in terms of time complexity.

The only way I could think of is to use a data structure to keep the array sorted. Moreover, the data structure has to support the insert operation. Red black tree can be a great `DS` in this situation. If you are using C++, you can use `set`. Java user could use `TreeSet`.

So every time you get a `prefix[i]`, all previous elements: `prefix[0...i-1]` are in the red black tree, sorted. You need to find the upper_bound of the `prefix[i]`, which is the first element that is larger than the `prefix[i]`. Why we just need the first larger element? The reason is very simple!

The code is very easy to write, if you learn the above ideas. !

## Solution

```java
static long maximumSum(long[] a, long m) {
    int length = a.length;
    long[] pre = new long[length];
    pre[0] = a[0] % m;
    TreeSet<Long> sorted = new TreeSet<>();
    long max = pre[0];
    sorted.add(pre[0]);
    for (int i = 1; i < length; i++) {
        pre[i] = (a[i] % m + pre[i - 1]) % m;
        Long needed = sorted.higher(pre[i]);
        if (needed != null) {
            long current = (pre[i] - needed + m) % m;
            max = Math.max(current, max);
        }
        sorted.add(pre[i]);
        max = Math.max(max, pre[i]);
    }
    return max;
}
```
