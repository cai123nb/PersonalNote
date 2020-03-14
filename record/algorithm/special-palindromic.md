# Special Palindrome Count

算法详情请查看`HackerRank`:[原文 link](https://www.hackerrank.com/challenges/special-palindrome-again/problem).

## Description

A string is said to be a `special palindromic` string if either of two conditions is met:

- All of the characters are the same, e.g. `aaa`.
- All characters except the middle one are the same, e.g. `aadaa`.

A special palindromic substring is any substring of a string which meets one of those criteria. Given a string, determine how many special palindromic substrings can be formed from it.

For example, given the string `s = mnonopoo`, we have the following special palindromic substrings: `{m, n, o, n, o, p, o, o, non, ono, opo, oo}`.

简单描述: 一个字符串中包含多少个回文类型的子字符串. 如`mnonopoo`包含的子回文字符串为:`{m, n, o, n, o, p, o, o, non, ono, opo, oo}`, 计数为12.

### Function Description

Complete the substrCount function in the editor below. It should return an integer representing the number of special palindromic substrings that can be formed from the given string.

substrCount has the following parameter(s):

- `n`: an integer, the length of string s.
- `s`: a string

### Input Format

The first line contains an integer, `n` , the length of `s`. The second line contains the string `s`.

### Constraints

- `1 <= n <= 10^6`
- Each character of the string is a lowercase alphabet, `ascii[a-z]`.

### Output Format

Print a single line containing the count of total special palindromic substrings.

## Sample

### Sample Input

```java
5
asasd
```

### Sample Output

```java
7
```

### Explanation

The special palindromic substrings of `s=asasd` are `{a, s, a, s, d, asa, sas}`.

## Thinking space

可以合理的将问题划分成两种情况:

- 全部元素都相同: 如`a,aaa`, 这时候计数为: `(curr.count * (curr.count + 1)) / 2`. 如`aaa`有: `3 * 4 / 2 = 6`个:`{a1,a2,a3,a1a2,a2a3,a1a2a3}`
- 中间嵌套其他元素, 如`aabaa`. 这时候的计数就为: 两边相等的个数. 如`aabaaa`就包含2个: `{aba, aabaa}`.

## Solution

```java
//Record char frequency struct
static class Point {
    public char key;
    public long count;

    public Point(char x, long y) {
        key = x;
        count = y;
    }
}

// Complete the substrCount function below.
static long substrCount(int n, String s) {
    s = s + " ";
    ArrayList<Point> l = new ArrayList<Point>();
    long count = 1;
    char ch = s.charAt(0);
    //Record all char frequency
    for(int i = 1; i <= n ; i++) {
        if(ch == s.charAt(i))
            count++;
        else {
            l.add(new Point(ch, count));
            count = 1;
            ch = s.charAt(i);
        }
    }
    count = 0;
    //If size >= 3, it may combine aabaa type
    if(l.size() >= 3) {
        Iterator<Point> itr = l.iterator();
        Point prev, curr, next;
        curr = (Point)itr.next();
        next = (Point)itr.next();
        count = (curr.count * (curr.count + 1)) / 2;
        for(int i = 1; i < l.size() - 1; i++) {
            prev = curr;
            curr = next;
            next = itr.next();
            count += (curr.count * (curr.count + 1)) / 2;
            //if chars can combile like: aaaxaaa, record counts
            if(prev.key == next.key && curr.count == 1)
                count += prev.count > next.count ? next.count : prev.count;
        }
        count += (next.count * (next.count + 1)) / 2;
    } else {
        //Just record equals situations
        for(Point curr:l){
            count += (curr.count * (curr.count + 1)) / 2;
        }
    }
    return count;
}
```
