# Maximum Xor

巧妙的使用特里树(`Trie`)进行数据存储, 有效降低时间复杂度的典型例子. 算法详情请查看`HackerRank`: [原文 link](https://www.hackerrank.com/challenges/maximum-xor/problem).

## Description

You are given an array `arr` of `n` elements. A list of integers, `queries` is given as an input, find the maximum value of `queries[j]` XOR `each arr[r]` for all 0 <= i < n, where `XOR` represents xor of two elements.

Note that there are multiple test cases in one input file.

For example:

- `arr = [3, 7, 15, 10]`
- `queries[j]=3`
- `3 XOR 3 = 0, max = 0`
- `3 XOR 7= 4, max = 4`
- `3 XOR 12 12 0, max = 12`
- `3 XOR 9 = 12, max = 12`

## Input Format

The first line contains an integer `n` , the size of the array `arr`.

The second line contains `n` space-separated integers`arr[i]`, from `0 <= i < n`.

The third line contain `m`, the size of the array `queries`.

Each of the next `m` lines contains an integer `queries[j]` where `0 <= j <m`.

## Constraints

- `1<= n, m <= 10^5`
- `1<= arr[i],queries[j] <= 10^9`

### Output Format

The output should contain `m` lines with each line representing output for the corresponding input of the testcase.

## Sample

### Sample Input

```java
3
0 1 2
3
3
7
2
```

### Sample Output

```java
3
7
3
```

### Explanation

- `arr=[0,1,2]`
- `queries[0]=3`

  - `3 XOR 0 = 3, max = 3`
  - `3 XOR 1 = 2, max = 3`
  - `3 XOR 2 = 1, max = 3`

- `queries[1]=7`

  - `7 XOR 0 = 7, max = 7`
  - `7 XOR 1 = 6, max = 7`
  - `7 XOR 2 = 5, max = 7`

- `queries[2]=2`

  - `2 XOR 0 = 2, max = 2`
  - `2 XOR 1 = 3, max = 3`
  - `2 XOR 2 = 0, max = 3`

## Analysis

如果使用暴力破解法是非常简单的, 时间复杂度是`O(n*m)`. 详细的做法不再详细讲解, 请查看`code1`. 这样虽然可以通过部分用例, 对于数据量大的用例还是会超时. 如何降低时间复杂度?

首先`n`和`m`相比, 比较容易降低的肯定是`n`, `m`基本上很难降低复杂度. 要降低`n`的复杂度, 即要对需要比较的数组进行特定结构的存储, 方便后面每次找出最大的`xor`的值, 而不用每次都遍历一遍. 想到这里`Trie`结构就非常适合这种情况. `Trie`: 二叉树的一种变种, 有一个公共的父节点, 派生出不同的子节点, 详情请查看[链接](https://zh.wikipedia.org/wiki/Trie). 简单举个例子. 如有一组数据是姓名和年龄是一一对应:

```javascript
amy      56
ann      15
emma     30
rob      27
roger    52
```

如果使用`Trie`来存储结果为:

```java
root:         .
      /       |      \
     a        e       r
   /   \      |       |
  m     n     m       o
  |     |     |     /   \
  y     n     m    b     g
  |     |     |    |     |
\0 56 \0 15   a  \0 27   e
              |          |
            \0 30        r
                         |
                       \0 52
```

使用这种结构, 我们就可以很快找出最匹配的`xor`的值. 我们都知道最大的`xor`值正好是相反的, 如`5(101)`的最大`xor`值为`010`. 我们只需要将提前所有进行比较的数据存储成`Trie`树, 然后再去其中找到尽可能符合条件的那个值即可.

## Solution

### Code1 - Brute

```java
static int[] maxXor(int[] arr, int[] queries) {
  // solve here
  return Arrays.stream(queries).map(x -> maxXorByOne(arr, x)).toArray();
}

static int maxXorByOne(int[] arr, int data) {
  int max = Integer.MIN_VALUE;
  for (int element : arr) {
    int current = element ^ data;
    max = max > current ? max : current;
  }
  return max;
}
```

### Code2 - Trie

```java
static class TrieNode {
  TrieNode leftNode; // 0 store in left
  TrieNode rightNode; // 1 store in right
  Integer value; // store current value
  public TrieNode() {}
}

static int[] maxXor(int[] arr, int[] queries) {
  // solve here
  TrieNode root = buildTrie(arr);
  return Arrays.stream(queries).map(x -> queryMaxByTrie(root, x)).toArray();
}

private static int queryMaxByTrie(TrieNode root, int x) {
  TrieNode head = root;
  // get binary string, like 10 -> 1010
  String binaryString = Integer.toBinaryString(x);
  for (int i = 0; i < (32 - binaryString.length()); i++) {
    // before is all zero, like 1010, actually: 00000000...01010,
    // try go to right
    if (head.rightNode != null) {
      head = head.rightNode;
    } else {
      // right is not arrivable, have to left
      head = head.leftNode;
    }
  }
  for (int i = 0; i < binaryString.length(); i++) {
    if (binaryString.charAt(i) == '0') {
      head = head.rightNode != null ? head.rightNode : head.leftNode;
    } else {
      head = head.leftNode != null ? head.leftNode : head.rightNode;
    }
  }
  return head.value ^ x;
}

private static TrieNode buildTrie(int[] arr) {
  TrieNode root = new TrieNode();
  for (int data : arr) {
    insertTrieNode(root, data);
  }
  return root;
}

private static void insertTrieNode(TrieNode root, int data) {
  TrieNode head = root;
  // get binary string, like 10 -> 1010
  String binaryString = Integer.toBinaryString(data);
  for (int i = 0; i < (32 - binaryString.length()); i++) {
    // before is all zero, like 1010, actually: 00000000...01010,
    if (head.leftNode == null) {
      head.leftNode = new TrieNode();
    }
    head = head.leftNode;
  }
  for (int i = 0; i < binaryString.length(); i++) {
    if (binaryString.charAt(i) == '0') {
      if (head.leftNode == null) {
        head.leftNode = new TrieNode();
      }
      head = head.leftNode;
    } else {
      if (head.rightNode == null) {
        head.rightNode = new TrieNode();
      }
      head = head.rightNode;
    }
  }
  head.value = data;
}
```
