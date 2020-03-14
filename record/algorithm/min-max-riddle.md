# Min Max Riddle

最小中最大问题, 如何在固定时间内完成不同情况计算. 算法详情请查看`HackerRank`:[原文 link](https://www.hackerrank.com/challenges/min-max-riddle/problem)

## Description

Given an integer array of size `n` , find the maximum of the minimum(s) of every window size in the array. The window size varies from `1` to `n`.

For example, given `arr = [6, 3, 5, 1, 12]` , consider window sizes of `1` through `5`. Windows of size `1` are `(6),(3),(5),(1),(12)` . The maximum value of the minimum values of these windows is `12`. Windows of size `2` are `(6,3),(3,5),(5,1),(1,12)` and their minima are `(3,3,1,1)`. The maximum of these values is `1`. Continue this process through window size `5` to finally consider the entire array. All of the answers are `12, 3, 3, 1, 1`.

### Function Description

Complete the riddle function in the editor below. It must return an array of integers representing the maximum minimum value for each window size from `1` to `n` .

riddle has the following parameter(s):

- `arr`: an array of integers

### Input Format

The first line contains a single integer `n`, , the size of `arr`. The second line contains `n` space-separated integers, each an `arr[i]`.

### Constraints

- `1 <= n < 10^6`
- `0 <= arr[i] <= 10^9`

### Output Format

Single line containing `n` space-separated integers denoting the output for each window size from `1` to `n`.

## Thinking space

正常的处理, 请参见代码`riddle1()`. 结论是必然超时. 必须要采用某些方法将时间复杂度降下来:

A few hints for people who are banging their heads on the wall trying to pass all test cases:

1) O(N) solution is possible using stacks; avoid DP for this problem

2) Think about how to identify the largest window a number is the minimum for (e.g. for the sequence 11 2 3 14 5 2 11 12 we would make a map of number -> window_size as max_window = {11: 2, 2: 8, 3: 3, 14: 1, 5: 2, 12: 1}) - this can be done using stacks in O(n)

3) Invert the max_window hashmap breaking ties by taking the maximum value to store a mapping of windowsize -> maximum_value (continuing with example above inverted_windows = {1: 14, 8:2, 3:3, 2:11}

4) starting from w=len(arr) iterate down to a window size of 1, looking up the corresponding values in inverted_windows and fill missing values with the previous largest window value (continuing with the example result = [2, 2, 2, 2, 2, 3, 11, 14] )

5) Return the result in reverse order (return [14, 11, 3, 2, 2, 2, 2, 2])

--- 摘自[SagunB](https://www.hackerrank.com/SagunB)

## Solution

### Normally

```java
static long[] riddle1(long[] arr) {
    // complete this function
    long[] result = new long[arr.length];
    for (int i = 1; i <= arr.length; i++) {
        result[i - 1] = getMax(i, arr);
    }
    return result;
}

static long getMax(int windowSize, long[] data) {
    long MAX = Long.MIN_VALUE;
    for (int i = 0; i < (data.length - windowSize + 1); i++) {
        long min = getMin(data, i, windowSize);
        if (min > MAX) {
            MAX = min;
        }
    }
    return MAX;
}

static long getMin(long[] data, int startIndex, int windowSize) {
    long MIN = Long.MAX_VALUE;
    for (int i = startIndex; i < (startIndex + windowSize); i++) {
        if (data[i] < MIN) {
            MIN = data[i];
        }
    }
    return MIN;
}
```

### optimization

```java
static long[] riddle2(long[] arr) {
    // calculate number smallest to window size.
    List<ValuePair> pairs = getNumberSmallestWindowSize(arr);
    // find biggest one corresponding window size
    Map<Integer, Long> biggest = getBiggestNumberCoorespondingWindowSize(pairs);
    // fill empty position with small one in array
    return fillArrayWithSmallOne(biggest, arr.length);
}

private static long[] fillArrayWithSmallOne(Map<Integer, Long> biggest, int size) {
    long[] result = new long[size];
    long lastValue = 0;
    for (int i = size - 1; i >= 0; i--) {
        if (biggest.containsKey(i + 1)) {
            lastValue = Math.max(biggest.get((i + 1)), lastValue);
        } else {
            result[i] = lastValue;
        }
    }
    return result;
}

static Map<Integer, Long> getBiggestNumberCoorespondingWindowSize(List<ValuePair> pairs) {
    Map<Integer, Long> biggest = new TreeMap<>();
    for (ValuePair valuePair : pairs) {
        if (biggest.containsKey(valuePair.size)) {
            biggest.put(valuePair.size, Math.max(valuePair.value, biggest.get(valuePair.size)));
        } else {
            biggest.put(valuePair.size, valuePair.value);
        }
    }
    return biggest;
}

static List<ValuePair> getNumberSmallestWindowSize(long[] data) {
    List<ValuePair> result = new ArrayList<>();
    Stack<ValuePair> valuePairs = new Stack<>();
    for (long oneData : data) {
        int popBigDataCount = 0;
        // If bottom is big, just pop it.
        while (!valuePairs.isEmpty() && valuePairs.peek().value >= oneData) {
        // get a small one, pop big
        ValuePair bigValuePair = valuePairs.pop();
        popBigDataCount += bigValuePair.size;
        bigValuePair.size = popBigDataCount;
        if (bigValuePair.value != oneData) {
            result.add(bigValuePair);
        }
    }
        // oh, i am big then bottom, now.
        valuePairs.add(new ValuePair(oneData, 1 + popBigDataCount));
    }

    // pop up remaining element
    int remainBigCount = 0;
    while (!valuePairs.isEmpty()) {
        ValuePair bigValuePair = valuePairs.pop();
        bigValuePair.size = bigValuePair.size + remainBigCount;
        result.add(bigValuePair);
        remainBigCount = bigValuePair.size;
    }
    return result;
}

static class ValuePair {
    long value;
    int size;

    public ValuePair(long value, int size) {
        this.value = value;
        this.size = size;
    }
}
```
