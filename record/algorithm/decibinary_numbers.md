# Decibinary Numbers

![dynamic programming](https://image.cjyong.com/dynamic-programming.jpg)

动态规划的高级应用, 高难度的一道算法题. 不仅难在建模, 更是难在使用. 花费一个礼拜, 终于解答该题. 特此纪念, 留在后续温习查看. 算法源连接请查看`底部参考资料`.

## Description

Let's talk about binary numbers. We have an `n-digit` binary number `b`, and we denote the digit at index `i` (zero-indexed from right to left) to be `b(i)`. We can find the decimal value of `b` using the following formula:

```java
(b)2 = b(n-1) * 2^(n - 1) + ... + b(2) * 2^2 + b(1) * 2^1 + b(0) * 2^0;
```

For example, if binary number `b=10010`, we compute its decimal value like so:

```java
(10010)2 = 1 * 2^4 + 0 * 2^3 + 0 * 2^2 + 1 * 2^1 + 0 * 2^0 = (18)10
```

Meanwhile, in our well-known decimal number system where each digit ranges from 0 to 9, the value of some decimal number, `d` , can be expanded in the same way:

```java
d = d(n-1) * 10^(n - 1) + ... + d1 * 10^1 + d0 * 10^0
```

Now that we've discussed both systems, let's combine decimal and binary numbers in a new system we call decibinary! In this number system, each digit ranges from 0 to 9 (like the decimal number system), but the place value of each digit corresponds to the one in the binary number system. For example, the decibinary number `2016` represents the decimal number `24`. because:

```java
(2016)decibinary = 2 * 2^3 + 0 * 2^2 + 1 * 2^1 + 6 * 2^0 = (24)10
```

Pretty cool system, right? Unfortunately, there's a problem: two different decibinary numbers can evaluate to the same decimal value! For example, the decibinary number `2008` also evaluates to the decimal value `24` :

```java
(2008)decibinary = 2 * 2^3 + 0 * 2^2 + 0 * 2^1 + 8 * 2^0 = (24)10
```

This is a major problem because our new number system has no real applications beyond this challenge!

Consider an infinite list of non-negative decibinary numbers that is sorted according to the following rules:

- The decibinary numbers are sorted in increasing order of the decimal value that they evaluate to.
- Any two decibinary numbers that evaluate to the same decimal value are ordered by increasing decimal value, meaning the equivalent decibinary values are strictly interpreted and compared as decimal values and the smaller decimal value is ordered first. For example,`(2)decibinary` and `(10)decibinary` both evaluate to `(2)10` . We would order `(2)decibinary` before `(10)decibinary` because `(2)10 < (10)10`.

Here is a list of first few decibinary numbers properly ordered:

![data](https://image.cjyong.com/decibinary_data.png)

You will be given `q` queries in the form of an integer, `x` . For each `x` , find and print the the `xth` decibinary number in the list on a new line.

### Function Description

Complete the decibinaryNumbers function in the editor below. For each query, it should return the decibinary number at that one-based index.

decibinaryNumbers has the following parameter(s):

x: the index of the decibinary number to return

- `x`: the index of the decibinary number to return.

### Input Format

The first line contains an integer, `q` , the number of queries. Each of the next `q` lines contains an integer, `x` , describing a query.

### Constraints

- `1 <= q <= 10^5`

- `1 <= x <= 10^16`

### Output Format

For each query, print a single integer denoting the the `xth` decibinary number in the list. Note that this must be the actual decibinary number and not its decimal value. Use 1-based indexing.

## Sample

### Sample Input

```java
5
1
2
3
4
10
```

### Sample Output

```java
0
1
2
10
100
```

### Explanation

Just look for the `xth` decibinary value from the table.

## Thinking space

思考的突破点, 来自于`Hristo lliev`的一段评论:

![hristo lliev](https://image.cjyong.com/hristo-lliev-comment.png)

这段评论, 也很好的确定了解决思路:

- 第一点: 使用一个动态规划, 构建一个N(d,m)的map, 用于存储数字d在m位bit下可能的`Decibinary`表现方式.
- 第二点: 通过`map`我们可以定位到, 所给的值是对应那个十进制数的第几个`Decibinary`表现值, 然后通过这个信息和排序规则, 还原对应的`decibinary`的值.

其中这两点都是有一定难度的, 第一点的难点在于合理的建模, 需要将位数考虑进去, 不然容易出错(位数冲突). 第二点的难点是, 如何利用获取的定位信息正确的还原对应的`Decibinary`值.

当然这应该只是其中一种解决方案, 肯定还有别的思路, 但是这应该是最简单, 最容易理解的解决思路了.

`talk is cheap, show me the code`:

## Solution

```java
// calculate max digit
static final int MAX_DIGIT = 20;
// calculate max decimal value
static final int MAX_DECIMAL_VALUE = 300_000;
// decimal digit count: 0,1,2,3,4,5,6,7,8,9
static final int DECIMAL_DIGIT_COUNT = 10;
// it store decimal value use part digit, have how many decibinary express
// for example: [4][2] = 3, because, 4 in two digit decibinary have: 12, 20, 4 (not 100)
static long[][] DECIMAL_VALUE_DIGIT_DECIBINARY_COUNT = new long[MAX_DECIMAL_VALUE][MAX_DIGIT];
// cumulative count for decimal value
// for example: 0: have 1 decibinary, 1: have 1 decibinary, 2: have 2 decibinary
// it will be: [1,2,4]
static long[] CUMULATIVE_DECIMAL_COUNTS = new long[MAX_DECIMAL_VALUE];

// call before all query
static void preCalculate() {
  // set DECIMAL_VALUE_DIGIT_DECIBINARY_COUNT init value
  for (int decimalValue = 0; decimalValue < MAX_DECIMAL_VALUE; decimalValue++) {
    // use one digit, it max value is 9, if large than 9, it can't use one digit to represent
    DECIMAL_VALUE_DIGIT_DECIBINARY_COUNT[decimalValue][0] = decimalValue < DECIMAL_DIGIT_COUNT ? 1 : 0;
  }

  // use dynamic calculate to fill DECIMAL_VALUE_DIGIT_DECIBINARY_COUNT
  for (int digit = 1; digit < MAX_DIGIT; digit++) {
    // current digit value
    long currentDigitDecimalValue = 1 << digit;
    // fill table with target digit
    for (int decimalValue = 0; decimalValue < MAX_DECIMAL_VALUE; decimalValue++) {
      // it can use the digit count(max 9)
      int maxUses = Math.min(9, (int)Math.floor(decimalValue/currentDigitDecimalValue));

      // it should add all combine possibility
      for (int i = 0; i <= maxUses; i++) {
        // it minus used value(on current digit), calculate count
        int remainValue = (int)(decimalValue - i * currentDigitDecimalValue);
        DECIMAL_VALUE_DIGIT_DECIBINARY_COUNT[decimalValue][digit] += DECIMAL_VALUE_DIGIT_DECIBINARY_COUNT[remainValue][digit - 1];
      }
    }
  }

  // Count the accumulated number
  CUMULATIVE_DECIMAL_COUNTS[0] = 1;
  for (int i = 1; i < MAX_DECIMAL_VALUE; i++) {
    int maxDigit = (int)Math.floor(Math.log(i)/Math.log(2));
    CUMULATIVE_DECIMAL_COUNTS[i] = DECIMAL_VALUE_DIGIT_DECIBINARY_COUNT[i][maxDigit] +
      CUMULATIVE_DECIMAL_COUNTS[i - 1];
  }
}


static long decibinaryNumbers(long targetPosition) {
  long result = 0;
  // get decimal value, like position 9 correspond decimal : 4
  int decimalValue = Arrays.binarySearch(CUMULATIVE_DECIMAL_COUNTS, targetPosition);
  decimalValue = decimalValue < 0 ? -decimalValue - 1: decimalValue;
  // get offset of current decimal, like 9 correspond 4's offset is 3, the third decibinary number of 4.
  long offset = decimalValue > 0 ? (targetPosition - CUMULATIVE_DECIMAL_COUNTS[decimalValue - 1]): 0;

  // from max digit to zero, more digit must contain less digit value count
  for (int currentDigit = MAX_DIGIT - 1; currentDigit >= 1; currentDigit--) {
    // current digit representative value
    long digitValue = 1 << currentDigit;
    // from 0 to 9 (keep the order)
    for (int digitCount = 0; digitCount < DECIMAL_DIGIT_COUNT; digitCount++) {
      // sub use value get remain value
      int remainValue = (int)(decimalValue - digitCount * digitValue);
      // if offset is small than current value count, it's what we need
      if (offset <= DECIMAL_VALUE_DIGIT_DECIBINARY_COUNT[remainValue][currentDigit - 1]) {
        result += digitCount;
        decimalValue -= digitValue * digitCount;
        break;
      }
      // if offset is big than current value count, it will remove those count
      offset -= DECIMAL_VALUE_DIGIT_DECIBINARY_COUNT[remainValue][currentDigit - 1];
    }
    // it store in result with decimal format
    result *= 10;
  }
  result += decimalValue;

  return result;
}
```

## 参考资料

- [HackRank算法链接](https://www.hackerrank.com/challenges/decibinary-numbers)
- [Hristo lliev评论链接](https://math.stackexchange.com/questions/3540243/whats-the-number-of-decibinary-numbers-that-evaluate-to-given-decimal-number/3545775#3545775)
- [Generating_function](https://en.wikipedia.org/wiki/Generating_function)
