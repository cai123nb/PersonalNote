# Common Child

最大相同子元素问题,算法详情请查看`HackerRank`:[原文 link](https://www.hackerrank.com/challenges/common-child/problem)

## Description

A string is said to be a child of a another string if it can be formed by deleting 0 or more characters from the other string. Given two strings of equal length, what's the longest string that can be constructed such that it is a child of both?

For example, `ABCD` and `ABDC` have two children with maximum length 3, `ABC` and `ABD`. They can be formed by eliminating either the `D` or `C` from both strings. Note that we will not consider `ABCD` as a common child because we can't rearrange characters and `ABCD`, `ABDC`.

### Function Description

Complete the commonChild function in the editor below. It should return the longest string which is a common child of the input strings.

commonChild has the following parameter(s):

- `s1, s2`: two equal length strings

### Input Format

There is one line with two space-separated strings, `s1` and `s2`.

### Constraints

- `1 <= s1, s2 <= 5000`.
- All characters are upper case in the range ascii[A-Z].

### Output Format

Print the length of the longest string , such that is a child of both `s1` and `s2`.

## Sample

### Sample Input

```java
HARRY
SALLY
```

### Sample Output

```java
2
```

### Explanation

The longest string that can be formed by deleting zero or more characters from `HAPPY` and `SALLY` is `AY`, whose length is 2.

## Thinking space

使用分治法可以很好解决这种情况, 重要的是要存储每一次分治的结果. 具体思路清参照代码注释.

## Solution

```java
// Complete the commonChild function below.
    static int commonChild(String s1, String s2) {
        //https://en.wikipedia.org/wiki/Longest_common_subsequence_problem
        //Assume s1 char array is [X1, X2, X3, ... , Xn]
        //Assume s2 char array is [Y1, Y2, Y3, ... , Ym]
        /*
                        if (n = 0 || m = 0) return "";
        LCS(Xn, Ym) =   else if (Xn = Xm) return LCS(Xn-1, Ym-1) + Xn;
                        else return Math.max(LCS(Xn-1, Ym), LCS(Xn, Ym-1));
        */
        // Assume s1=AGCAT, S2=GAC
        //    ""    A       G       C       A       T
        // "" ""    ""      ""      ""      ""      ""
        // G  ""    ""      G       G       G       G
        // A  ""    A      A|G      A|G     GA      GA
        // C  ""    A      A|G      AC|GC  AC|GC|GA AC|GC|GA

        char[] charsA = s1.toCharArray(), charsB = s2.toCharArray();
        int[][] datas = new int[charsA.length + 1][charsB.length + 1];

        for (int i = 0; i < charsA.length; i++) {
            for (int j = 0; j < charsB.length; j++) {
                if (charsA[i] == charsB[j]) {
                    datas[i + 1][j + 1] = 1 + datas[i][j];
                } else {
                    datas[i + 1][j + 1] = Math.max(datas[i][j + 1], datas[i + 1][j]);
                }
            }
        }

        return datas[charsA.length][charsB.length];
    }
```
