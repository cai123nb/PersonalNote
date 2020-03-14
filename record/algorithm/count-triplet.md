# Count Triplet

暴力超时,巧妙建模例子,算法详情请查看`HackerRank`:[原文 link](https://www.hackerrank.com/challenges/count-triplets-1/problem)

## Description

You are given an array and you need to find number of tripets of indices `(i,j,k)` such that the elements at those indices are in geometric progression for a given common ratio `r` and `i < j < k`.

For example, `arr=[1,4,16,64]`. If `r=4`, we have `[1,4,16]` and `[4,16,64]` at indices `(0,1,2)`and`(1,2,3)` .

### Function Description

Complete the countTriplets function in the editor below. It should return the number of triplets forming a geometric progression for a given `r` as an integer.

countTriplets has the following parameter(s):

- `arr`: an array of integers.

- `r`: an integer, the common ratio.

### Input Format

The first line contains two space-separated integers `n` and `r`, the size of `arr` and the common ratio. The next line contains `n` space-seperated integers `arr[i]` .

### Constraints

- `1 <= n <= 10^5`

- `1 <= r <= 10^9`

- `1 <= arr[i] <= 10^9`

### Output Format

Return the count of triplets that form a geometric progression.

## Sample

### Sample Input

```java
4 2
1 2 2 4
```

### Sample Output

```java
2
```

### Explanation

There are 2 triplets in satisfying our criteria, whose indices are `(0,1,3)` and `(0,2,3)`.

## Thinking space

暴力求解非常容易超时, 关键在于将问题分解抽象为A,B,C需求,然后进行建模.

## Solution

```java
// Complete the countTriplets function below.
static long countTriplets(List<Long> arr, long r) {
    //Conside the triple is A, B, C (B = A * r, C = A * r * r = B * r)
    //B demand counts, if an long input will increase B(A * r)demand
    Map<Long, Long> bDemand = new HashMap<>();
    //C deamnd counts, if an long input will statisfy C demand,
    //will combine a triple
    Map<Long, Long> cDemand = new HashMap<>();
    long result = 0L;

    for(Long a : arr) {
        //If long a can fit C demand, it will combine some triples.
        result += cDemand.getOrDefault(a, 0L);
        if (bDemand.containsKey(a)){
            //If long a can fit B demand, will increase C demand.
            cDemand.put(a*r, cDemand.getOrDefault(a*r, 0L) + t2.get(a));
        }
        //every a can be A, will increase B demand.
        bDemand.put(a*r, bDemand.getOrDefault(a*r, 0L) + 1);
    }
    return result;
}
```
