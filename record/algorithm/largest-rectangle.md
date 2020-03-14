# Largest Rectangle

最大长方体问题, 栈的有趣用法. 算法详情请查看`HackerRank`:[原文 link](https://www.hackerrank.com/challenges/largest-rectangle/problem)

## Description

Skyline Real Estate Developers is planning to demolish a number of old, unoccupied buildings and construct a shopping mall in their place. Your task is to find the largest solid area in which the mall can be constructed.

There are a number of buildings in a certain two-dimensional landscape. Each building has a height, given by `h[i]`. If you join `k` adjacent buildings, they will form a solid rectangle of area `k x min(h[i], h[i+1], ... ,h[i+k-1]`.

For example, the heights array `h=[3,2,3]`. A rectangle of height `h=2` and length `k=3` can be constructed within the boundaries. The area formed is `hxk=2x3=6`.

### Function Description

Complete the function largestRectangle int the editor below. It should return an integer representing the largest rectangle that can be formed within the bounds of consecutive buildings.

largestRectangle has the following parameter(s):

- h: an array of integers representing building heights

### Input Format

The first line contains n, the number of buildings. The second line contains n space-separated integers, each representing the height of a building.

### Constraints

- `1 <= n < 10^5`
- `1 <= h[i] < 10^6`

### Output Format

Print a long integer representing the maximum area of rectangle formed.

## Sample

### Sample Input

```java
5
1 2 3 4 5
```

### Sample Output

```java
9
```

### Explanation

The largest rectangle will be `3 x 3` in the last three rectangle.

## Thinking space

使用栈依次压入长方体, 如果长方体比栈内的要矮, 动态弹出内部长方体(如果包含当前压如长方体就无法构建一个完整的矩形), 这个时候动态计算弹出矩形所能构造最大面积即可. 详情阅读代码.

## Solution

```java
static class Rectangle {
    int left;
    int right;
    int height;
    public Rectangle(int left, int height) {
        this.left = left;
        this.height = height;
    }

    public long getArea() {
        return height * (right - left);
    }
}

// Complete the largestRectangle function below.
static long largestRectangle(int[] h) {
    Deque<Rectangle> stack = new ArrayDeque<>();
    stack.push(new Rectangle(0, h[0]));
    long maxArea = 0;
    for (int i = 1; i < h.length; i++) {
        if (stack.size() > 0 && (stack.peek()).height > h[i]) {
            //这就意味着, 新进入的建筑会终结(比之前要矮)前面一些建筑所构成的长方形.
            Rectangle tr = null;
            Rectangle used = null;
            while (stack.size() > 0 && (tr = stack.peek()).height > h[i]) {
                tr = stack.pop();
                used = tr;
                tr.right = i;
                maxArea = Math.max(maxArea, tr.getArea());
            }
            stack.push(new Rectangle(used.left, h[i]));
        } else {
            //这就意味着, 新进入的建筑会拓展之前的建筑一个长度的矩形.
            //注意这里的起始点从当前开始(因为当前建筑目前是最高的)
            stack.push(new Rectangle(i, h[i]));
        }
    }
    //剩下在堆内的建筑都是可以达到所有长度的
    while (stack.size() > 0) {
        Rectangle rt = stack.pop();
        rt.right = h.length;
        maxArea = Math.max(maxArea, rt.getArea());
    }
    return maxArea;
}
```
