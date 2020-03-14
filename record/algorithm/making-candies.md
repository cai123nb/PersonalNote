# Making Candies

制造蛋糕问题, 关于贪婪的权衡问题. 算法详情请查看`HackerRank`:[原文 link](https://www.hackerrank.com/challenges/making-candies/problem)

## Description

Karl loves playing games on social networking sites. His current favorite is CandyMaker, where the goal is to make candies.

Karl just started a level in which he must accumulate `n` candies starting with `m` machines and `w` workers. In a single pass, he can make `m x w` candies. After each pass, he can decide whether to spend some of his candies to buy more machines or hire more workers. Buying a machine or hiring a worker costs `p` units, and there is no limit to the number of machines he can own or workers he can employ.

Karl wants to minimize the number of passes to obtain the required number of candies at the end of a day. Determine that number of passes.

For example, Karl starts with `m = 1` machine and `w = 2` workers. The cost to purchase or hire, `p = 1` and he needs to accumulate `60` candies. He executes the following strategy:

Make `m x w = 1 x 2 = 2` candies. Purchase two machines. Make `3 x 2 = 6` candies. Purchase `3` machines and hire `3` workers. Make `6 x 5 = 30` candies. Retain all `30` candies. Make `6 x 5 = 30` candies. With yesterday's production, Karl has `60` candies. It took `4` passes to make enough candies.

### Function Description

Complete the minimumPasses function in the editor below. The function must return a long integer representing the minimum number of passes required.

minimumPasses has the following parameter(s):

- m: long integer, the starting number of machines
- w: long integer, the starting number of workers
- p: long integer, the cost of a new hire or a new machine
- n: long integer, the number of candies to produce

### Input Format

A single line consisting of four space-separated integers describing the values of , , , and , the starting number of machines and workers, the cost of a new machine or a new hire, and the the number of candies Karl must accumulate to complete the level.

### Constraints

- `1 <= m,w,p,n <= 10^12`

### Output Format

Return a long integer denoting the minimum number of passes required to accumulate at least `n` candies.

## Sample

### Sample Input

```java
3 1 2 12
asasd
```

### Sample Output

```java
3
```

### Explanation

Karl makes three passes:

In the first pass, he makes `m x w = 1 x 3 = 3` candies. He then spends `p = 2` of them hiring another worker, so `w = 2` and he has one candy left over. In the second pass, he makes `3 x 2 = 6` candies. He spends `2 x p = 4` of them on another machine and another worker, so `w = 3` and and `m = 4` he has `3` candies left over. In the third pass, Karl makes `4 x 3 = 12` candies. Because this satisfies his goal of making at least `12` candies, we print the number of passes (i.e.,`3` ) as our answer.

## Thinking space

每次生产的蛋糕有两个选择:

- 购买机器或者工人: 这时候需要尽量补充差值, 让工人数尽量等于机器数(这样乘积最大, 生产效率最高).
- 当做存货, 攒够最终生产数量: 这时候就没办法购买工人或者机器数.

每一轮生产结束之后, 对两种选择做出判断. 选择最优解. 一直到最后满足要求.

## Solution

```java
static long minimumPasses(long m, long w, long p, long n) {
    long candies = 0;
    //invest统计实时的轮次信息
    long invest = 0;
    //spend统计, 本轮购买只生产需要的轮次(记录最小值).
    long spend = Long.MAX_VALUE;

    while (candies < n) {
        // preventing overflow in m*w
        //在先前没有足够的钱购买机器或者工人时, 只能单纯的生产, 计算单纯生产的轮次
        long passes = (long) (((p - candies) / (double) m) / w);

        if (passes <= 0) {
            //如果candies > p, 说明剩余的钱可以购买机器和工人

            // machines we can buy in total
            //尽量补差到不够的数量那边, 因为这样乘积最大
            long mw = candies / p + m + w;
            long half = mw >>> 1;
            if (m > w) {
                m = Math.max(m, half);
                w = mw - m;
            } else {
                w = Math.max(w, half);
                m = mw - w;
            }
            candies %= p;
            //将passes赋值1
            passes++;
        }

        // handling overflowing
        // if overflowing is encountered -> candies count are definitely more than long
        // thus it is more than n since n is long
        // so we've reached the goal and we can break the loop
        //这里要注意溢出, 一旦溢出也就说明实际上的生产数量满足了目标的数量.
        long mw;
        long pmw;
        try {
            mw = Math.multiplyExact(m, w);
            pmw = Math.multiplyExact(passes, mw);
        } catch (ArithmeticException ex) {
            // we need to add current pass
            invest += 1;
            // increment will be 1 because of overflow
            spend = Math.min(spend, invest + 1);
            break;
        }

        //将本轮生产生产的糖果统计
        candies += pmw;
        //添加本轮计算的轮次
        invest += passes;
        //这里计算, 如果后续不再购买机器和人工时(只是单纯的生产), 需要的轮次
        long increment = (long) Math.ceil((n - candies) / (double) mw);
        //将本轮过后, 不在购买只生产需要的轮次记录到spend中去, spend取最小值(和过往相比)
        spend = Math.min(spend, invest + increment);
    }

    //取一直购买的轮次数量和中间停顿购买轮次数量的较小值.
    return Math.min(spend, invest);
}
```
