# 笔试算法题两则

![interview](https://image.cjyong.com/interview.jpeg)

最近参加面试的时候遇到两道比较有趣的算法题, 也是比较有代表性, 在笔试的时候做的并不理想, 下来思考回顾之后, 总结了一下两道算法题的解决方案和相关拓展.

## 数字组合最大

给你一个数字列表, 如`20, 31, 4`, 将这些数字进行组合, 找到其中最大的组合方式.

如组合可以是: `20314`, `31204`, `42031`, `43120`, `20431`, `31420`, 供六种. 其中最大的组合为: `43120`.

这道题比较简单, 思路也比较清晰: 贪婪算法, 通过将数字进行组合比较, 将位数较大的往前排, 位数较小的往后排.

```java
private static String getMaxNumber(String[] numbers) {
  for (int i = 0; i < numbers.length - 1; i++) {
    for (int j = i + 1; j < numbers.length; j++) {
      if (compareNumberInString(numbers[j] + numbers[i], numbers[i] + numbers[j])) {
        String tmpJ = numbers[j];
        numbers[j] = numbers[i];
        numbers[i] = tmpJ;
      }
    }
  }
  return Arrays.toString(numbers);
}

private static boolean compareNumberInString(String prev, String next) {
  for (int i = 0; i < prev.length(); i++) {
    if (prev.charAt(i) != next.charAt(i)) {
      return prev.charAt(i) >= next.charAt(i);
    }
  }
  return false;
}
```

## 二叉树依据中序和后序输出层次遍历

给定一个二叉树的中序和后序遍历结果, 输出该二叉树的层次遍历结果.

如一个二叉树的:

- 中序遍历为: `"D", "B", "H", "E", "A", "F", "C", "G", "I"`
- 后序遍历为: `"D", "H", "E", "B", "F", "I", "G", "C", "A"`

可以构建该二叉树为:

![binary_tree](https://image.cjyong.com/binary-tree-exampl1.jpg)

则该二叉树的层次遍历为: `ABCDEFGHI`.

这道题难点在于使用后序和中序重构该二叉树, 在构建好二叉树之后, 使用队列就可以很容易的输出层次遍历的结果了.

需要重构该二叉树, 需要熟悉`中序`和`后序`遍历的特点, 如果将整体当做一个二叉树, 我们可以划分一下情况:

- 中序遍历为:

  - LEFT TREE: `"D", "B", "H", "E"`,
  - ROOT: `"A"`,
  - RIGHT TREE: `"F", "C", "G", "I"`

- 后序遍历为:

  - LEFT TREE `"D", "H", "E", "B"`,
  - RIGHT TREE: `"F", "I", "G", "C",`
  - ROOT: `"A"`

可以确定的两点有:

- 在一个树结构中, 后序遍历的最后一个为根节点(因为后序的遍历是: 左, 右,根).
- 通过可以确定的根节点, 可以在中序中确认左右子树的长度(因为中序的顺序是左, 中, 右).

依据这两个特性在生成的两个左右子树中是同样存在这个规律的. 因此可以递归分解, 直到叶节点.

```java
public static void main(String[] args) {
  String[] preOrder = new String[]{"A", "B", "D", "E", "H", "C", "F", "G", "I"};
  String[] midOrder = new String[]{"D", "B", "H", "E", "A", "F", "C", "G", "I"};
  String[] postOrder = new String[]{"D", "H", "E", "B", "F", "I", "G", "C", "A"};

  // TreeNode root = initTreeWithPreOrderAndMidOrder(preOrder, midOrder, 0, 0, preOrder.length);
  TreeNode root = initTreeWithPostOrderAndMidOrder(postOrder, midOrder, 0, 0, preOrder.length);
  levelOrderTraverse(root);
}

public static TreeNode initTreeWithPreOrderAndMidOrder(String[] preOrder, String[] midOrder, int preOrderIndex, int midOrderIndex, int length) {

  if (length == 0) {
    return null;
  }

  if (length == 1) {
    return new TreeNode(preOrder[preOrderIndex]);
  }

  //pre order first element must be tree root of current tree
  String root = preOrder[preOrderIndex];

  // find middle root index
  int midRootIndex = 0;
  for (int i = midOrderIndex; i < (midOrderIndex + length); i++) {
    if (root.equals(midOrder[i])) {
      midRootIndex = i;
      break;
    }
  }
  int leftTreeLength = midRootIndex - midOrderIndex;
  int rightTreeLength = length - leftTreeLength - 1;

  TreeNode node = new TreeNode(root);
  node.left = initTreeWithPreOrderAndMidOrder(preOrder, midOrder, preOrderIndex + 1, midRootIndex - leftTreeLength, leftTreeLength);
  node.right = initTreeWithPreOrderAndMidOrder(preOrder, midOrder, preOrderIndex + leftTreeLength + 1, midRootIndex + 1, rightTreeLength);
  return node;
}

public static TreeNode initTreeWithPostOrderAndMidOrder(String[] postOrder, String[] midOrder, int postOrderIndex, int midOrderIndex, int length) {
  if (length == 0) {
    return null;
  }

  if (length == 1) {
    return new TreeNode(postOrder[postOrderIndex]);
  }

  //post order last element must be tree root of current tree
  String root = postOrder[postOrderIndex + length - 1];

  // find middle root index
  int midRootIndex = 0;
  for (int i = midOrderIndex; i < (midOrderIndex + length); i++) {
    if (root.equals(midOrder[i])) {
      midRootIndex = i;
      break;
    }
  }
  int leftTreeLength = midRootIndex - midOrderIndex;
  int rightTreeLength = length - leftTreeLength - 1;

  TreeNode node = new TreeNode(root);
  node.left = initTreeWithPostOrderAndMidOrder(postOrder, midOrder, postOrderIndex, midRootIndex - leftTreeLength, leftTreeLength);
  node.right = initTreeWithPostOrderAndMidOrder(postOrder, midOrder, postOrderIndex + leftTreeLength, midRootIndex + 1, rightTreeLength);
  return node;
}

private static void levelOrderTraverse(TreeNode<String> root) {
  Queue<TreeNode> queue = new LinkedList<>();
  queue.add(root);
  while(queue.size() > 0) {
    TreeNode topNode = queue.remove();
    System.out.print(topNode.value);
    if (topNode.left != null) {
      queue.add(topNode.left);
    }
    if (topNode.right != null) {
      queue.add(topNode.right);
    }
  }
}
```
