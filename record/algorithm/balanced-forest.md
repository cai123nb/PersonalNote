# Balanced Forest

![dfs](https://image.cjyong.com/dfs-sample.jpeg)

`Balanced Forest`, 一个基于`DFS`(Depth First Search)的高级应用, 以DFS计算总和为基础, 进行最小树划分为最终目的, 需要花费一段时间好好思考和品味这道算法题.

## Description

Greg has a tree of nodes containing integer data. He wants to insert a node with some non-zero integer value somewhere into the tree. His goal is to be able to cut two edges and have the values of each of the three new trees sum to the same amount. This is called a balanced forest. Being frugal, the data value he inserts should be minimal. Determine the minimal amount that a new node can have to allow creation of a balanced forest. If it's not possible to create a balanced forest, return -1.

For example, you are given node values `c=[15,12,8,14,13]` and `edges=[[1,2],[1,3],[1,4],[4,5]]` . It is the following tree:

![samole-1](https://image.cjyong.com/forestExample-1.png)

The blue node is root, the first number in a node is node number and the second is its value. Cuts can be made between nodes `1` and `3` and nodes `1` and `4` to have three trees with sums `27`, `27` and `8`. Adding a new node `w` of `c[w]=19` to the third tree completes the solution.

### Function Description

Complete the balancedForest function in the editor below. It must return an integer representing the minimum value `c[w]` of that can be added to allow creation of a balanced forest, or `-1` if it is not possible.

balancedForest has the following parameter(s):

- `c`: an array of integers, the data values for each node
- `edges`: an array of 2 element arrays, the node pairs per edge

### Input Format

The first line contains a single integer, `q`, the number of queries.

Each of the following `q` sets of lines is as follows:

- The first line contains an integer, `n`, the number of nodes in the tree.
- The second line contains `n` space-separated integers describing the respective values of `C[1],C[2],...,C[n]`, where each `c[i]` denotes the value at node `i`.
- Each of the following `n - 1` lines contains two space-separated integers, `x[j]` and `y[j]`, describing edge `j` connecting nodes `x[j]` and `y[j]`.

### Constraints

- `1 <= q <= 5`

- `1 <= n <= 5 * 10^4`

- `1 <= c[i] <= 10^9`

- Each query forms a valid undirected tree.

### Output Format

For each query, return the minimum value of the integer `c[w]`. If no such value exists, return `-1` instead.

## Sample

### Sample Input

```java
2
5
1 2 2 1 1
1 2
1 3
3 5
1 4
3
1 3 5
1 3
1 2
```

### Sample Output

```java
2
-1
```

### Explanation

We perform the following two queries:

[1]: The tree initially looks like this:

![sample-2](https://image.cjyong.com/forestSample-2.png)

Greg can add a new node `w=6` with `c[w]=2` and create a new edge connecting nodes `4` and `6`. Then he cuts the edge connecting nodes `1` and `4` and the edge connecting nodes `1` and `3`. We now have a three-tree balanced forest where each tree has a sum of `3`.

![sample-2](https://image.cjyong.com/forestSample-3.png)

[2]: In the second query, it's impossible to add a node in such a way that we can split the tree into a three-tree balanced forest so we return `-1`.

## Thinking space

1. 将节点通过节点的方式存储下来, 节点之间的关联关系使用`adjacent`存储到每个节点中
2. 通过`DFS`的方式, 计算每一个点的总和(包含其子图的节点)
3. 通过`DFS`遍历所有的路径, 查找其中的最小的填充节点(难点)
4. 难点在于统计所有的可能进行切分的方式:

  1. 存储根节点到当前节点的路径之和列表
  2. 非当前节点的其他所遇节点的路径之和列表
  3. 利用两者列表进行动态拆分

## Solution

```java
static class Node {
  long cost;
  boolean costVisited = false, solveVisited = false;
  ArrayList<Integer> adjacent = new ArrayList<>();

  public Node(int cost) {
    this.cost = cost;
  }
}
static long sum, mini;
// it not contain self root node
static HashSet<Long> visitedNodeCostSum = new HashSet<>();
static HashSet<Long> allMeetNodeCostSum = new HashSet<>();

static long balancedForest(int[] c, int[][] edges) {
  Node[] nodes = new Node[c.length];
  for (int i = 0; i < c.length; i++) {
    nodes[i] = new Node(c[i]);
  }
  for (int[] edge : edges) {
    nodes[edge[0] - 1].adjacent.add(edge[1] - 1);
    nodes[edge[1] - 1].adjacent.add(edge[0] - 1);
  }

  sum = mini = dfsCalculateSum(nodes, 0);

  dfsCalculateMiniValue(nodes, 0);

  return sum == mini ? -1 : mini;
}

private static long dfsCalculateSum(Node[] nodes, int position) {
  Node node = nodes[position];
  if (node.costVisited) {
    return 0;
  }
  node.costVisited = true;

  for(Integer adjacentNodePosition: node.adjacent) {
    node.cost += dfsCalculateSum(nodes, adjacentNodePosition);
  }
  return node.cost;
}

private static void dfsCalculateMiniValue(Node[] nodes, int position) {
  Node node = nodes[position];
  if (node.solveVisited) {
    return;
  }
  node.solveVisited = true;

  // if it can combine three graph, it must be two big and one small graph

  // case 1, it can split to two equal part, with another node (value is sum / 2)
  if (sum % 2 == 0 && (sum / 2) == node.cost) {
    mini = Math.min(mini, sum / 2);
  }

  // case 2, big one is current node
  long addToSmall = 3 * node.cost - sum;
  if (
    // it is bigger enough to be big one
    (addToSmall >= 0) &&
        // other node have equal big one
      (allMeetNodeCostSum.contains(node.cost) ||
        // other node have small one
      (allMeetNodeCostSum.contains(sum - 2 * node.cost) ||
        // self node contain big + big node (can split to two)
        (visitedNodeCostSum.contains(sum - node.cost))))) {
    mini = Math.min(mini, addToSmall);
  }

  // case 3, small one is current node
  long bigOneCost = (sum - node.cost) / 2;
  if (
    // it can split to two big node
    ((sum - node.cost) % 2 == 0 && bigOneCost >= node.cost) &&
      // other node have equal big one
      (allMeetNodeCostSum.contains(bigOneCost) ||
        // self node contain big + small node
        (visitedNodeCostSum.contains(sum - bigOneCost)))) {
    mini = Math.min(mini, bigOneCost - node.cost);
  }

  visitedNodeCostSum.add(node.cost);

  for(Integer adjacentNodePosition: node.adjacent) {
    dfsCalculateMiniValue(nodes, adjacentNodePosition);
  }

  visitedNodeCostSum.remove(node.cost);

  allMeetNodeCostSum.add(node.cost);
}
```

- [HackRank算法链接](https://www.hackerrank.com/challenges/balanced-forest)
- [DFS Wikipedia](https://en.wikipedia.org/wiki/Depth-first_search)
- [DFS 相关应用](https://blog.csdn.net/weixin_43272781/article/details/82959089)
