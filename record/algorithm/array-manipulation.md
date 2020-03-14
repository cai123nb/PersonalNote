# Array Manipulation

数组驼峰问题, 算法详情请查看`HackerRank`:[原文 link](https://www.hackerrank.com/challenges/crush/problem).

## Description

给你一个长度为 N 的列表，列表的初始值全是 0。对此列表，你要进行 M 次查询，输出列表中最终 N 个值的最大值。对每次查询，给你的是 3 个整数----a,b 和 k，你要对列表中从位置 a 到位置 b 范围内的（包含 a 和 b)的全部元素加上 k。

### Input Format

第一行包含两个整数 N 和 M。 接下来 M 行，每行包含 3 个整数 a, b 和 k。

### Constraints

- `3 <= N <= 10^7`

- `1 <= M <= 2*10^5`

- `1 <= a <= b <= N`

- `0 <= k <= 10^9`

### Output Format

单独的一行包含 最终列表里的最大值.

## Sample

### Sample Input

```java
5 3
1 2 100
2 5 100
3 4 10
```

### Sample Output

```java
200
```

### Explanation

第一次更新后，列表变为 `100 100 0 0 0`. 第二次更新后，列表变为 `100 200 100 100 100`。 第三次更新后，列表变为 `100 200 200 200 100`。 因此要求的答案是`200`

## Thinking space

求解不难, 却容易超时. 关键点是通过对每个特殊节点设置上升下降浮标, 动态计算可以有效降低时间复杂度.

## Solution

```java
String[] nm = scanner.nextLine().split(" ");

int n = Integer.parseInt(nm[0]);

int m = Integer.parseInt(nm[1]);

//Use datas as number buoy, recoard number change.
long[] datas = new long[n];

int[] group = new int[3];

for (int i = 0; i < m; i++) {
    String[] queriesRowItems = scanner.nextLine().split(" ");
    scanner.skip("(\r\n|[\n\r\u2028\u2029\u0085])?");

    for (int j = 0; j < 3; j++) {
        group[j] = Integer.parseInt(queriesRowItems[j]);
    }
    //From group[0] - grou[1], it will increase nums.
    datas[group[0] - 1] += group[2];
    //Here need to be notice, the range is include， after the range, it will decrease
    if (group[1] < n) {
        datas[group[1]] -= group[2];
    }
}

long currentValue = 0;
long max= Long.MIN_VALUE;
for (int i = 0; i < n; i++) {
    currentValue += datas[i];
    if (currentValue > max) {
        max = currentValue; // target value
    }
}
```
