# 二叉树遍历小结

![binary tree](https://image.cjyong.com/binary-tree-to-DLL.png)

二叉树的数据结构是我们在开发中非常常见的一种数据结构, 其中遍历二叉树也是一种常见的操作. 本文对于二叉树的三种遍历(前序,中序和后序)进行小结, 提供递归实现, 栈实现和Morris算法等三种不同实现方式.

本文使用的二叉树数据结构如下:

```java
public class TreeNode {
  public int val;
  public TreeNode left;
  public TreeNode right;

  public TreeNode(int val) {
    val = val;
  }
}
```

## 递归实现

迭代实现是最为简单的实现, 实现的逻辑清晰易懂, 使用方法递归调用时产生的栈帧存储节点关联关系, 来达到不同的遍历效果. 代码比较简单, 就不多介绍.

### 递归-前序

前序为: `根节点 -> 左节点 -> 右节点`, `result`中存储遍历的结果.

**Code**:

```java
public void preOrderByRecursive(TreeNode node, List<Integer> result) {
  if (node != null) {
    result.add(node.val);
    preOrderByRecursive(node.left, result);
    preOrderByRecursive(node.right, result);
  }
}
```

### 递归-中序

中序为: `左节点-根节点-右节点`.

**Code**:

```java
public void inOrderByRecursive(TreeNode node, List<Integer> result) {
  if (node != null) {
    inOrderByRecursive(node.left, result);
    result.add(node.val);
    inOrderByRecursive(node.right, result);
  }
}
```

### 递归-后序

后序为: `左节点-右节点-根节点`.

**Code**:

```java
public void afterOrderByRecursive(TreeNode node, List<Integer> result) {
  if (node != null) {
    afterOrderByRecursive(node.left, result);
    afterOrderByRecursive(node.right, result);
    result.add(node.val);
  }
}
```

## 栈实现

递归实现在数据量比较大的时候, 容易超出允许的最大栈深度. 改进的方法为使用一个栈来存储父子节点关系, 而不是使用递归来实现.

### 栈实现-前序

栈实现前序的核心思路为:

- 先使用栈依次存储所有的左子节点, 同时存储所有的左节点. 这一步保证了从开始的处理顺序为: `根节点 -> 左节点`(即为`根 -> 左`处理的依次循环).
- 依次处理当前到达的最左节点之后, 这时候使用同样的逻辑处理其右节点. 这时候保证当前二叉树最后一步处理的节点为`右节点`, 保证了处理顺序为: `根节点 -> 左节点 -> 右节点`.

**Code**:

```java
public void preOrderByStack(TreeNode root, List<Integer> result) {
    Stack<TreeNode> stack = new Stack<>();
    TreeNode current = root;
    while (current != null || !stack.isEmpty()) {
      while (current != null) {
        stack.push(current);
        result.add(current.val);
        current = current.left;
      }
      current = stack.pop().right;
    }
}
```

**具体的处理流程图如下**:

![pre-order](https://image.cjyong.com/preorder-stack.jpg)

### 栈实现-中序

栈实现中序的核心思路为:

- 先使用栈依次存储所有的左子节点, **注意这时候不进行存储节点**.
- 这时候从最左节点开始处理, 将当前左节点存储, 往上回溯存储根节点, 保证了处理的顺序为`左节点 -> 根节点`.
- 这时候处理当前最左节点的右节点, 以同样的逻辑进行处理. 这就保证了`右节点`的处理顺序为当前二叉树的处理的最后节点, 即为: `左节点 -> 根节点 -> 右节点`

**Code**:

```java
public void inOrderByStack(TreeNode root, List<Integer> result) {
  Stack<TreeNode> stack = new Stack<>();
  TreeNode current = root;
  while (current != null || !stack.isEmpty()) {
    while (current != null) {
      stack.push(current);
      current = current.left;
    }
    current = stack.pop();
    result.add(current.val);
    current = current.right;
  }
}
```

**具体的处理流程图如下**:

![in-order](https://image.cjyong.com/inorder-stack.jpg)

### 栈实现-后序

栈实现后序时需要用到一个取巧的方法, 因为先处理`左节点, 右节点`, 再处理`根节点`, 会带来一些顺序上存储的困难. 这时候可以转换一下思路:

- 目标是: `左节点 -> 右节点 -> 根节点`顺序, 这时候, 如果我们可以先处理成`根节点 -> 右节点 -> 左节点`的顺序, 然后进行反转不就是正确答案了吗.
- 而处理`根节点 -> 右节点 -> 左节点`则非常简单, 因为这就是前序的`根 -> 左 -> 右`的一种变种, 只需要简单的转换即可:

  - 先使用栈依次存储所有的右子节点, 同时存储所有的右节点. 这一步保证了从开始的处理顺序为: `根节点 -> 右节点`(即为`根节点 -> 右节点`处理的依次循环).
  - 依次处理当前到达的最右节点, 这时候使用同样的逻辑处理其左节点. 这时候保证当前二叉树最后一步处理的节点为`左节点`, 保证了处理顺序为: `根节点 -> 右节点 -> 左节点`.

- 这时候对处理结果进行反转, 即为`左节点 -> 右节点 -> 根节点`的处理顺序, 即为我们需要的结果.

**Code**:

```java
public void afterOrderByStack(TreeNode root, List<Integer> result) {
  Stack<TreeNode> stack = new Stack<>();
  TreeNode current = root;
  while (current != null || !stack.isEmpty()) {
    if (current != null) {
      stack.push(current);
      result.add(current.val);
      current = current.right;
    } else {
      current = stack.pop();
      current = current.left;
    }
  }
  Collections.reverse(result);
}
```

**具体的处理流程图如下**:

![after-order](https://image.cjyong.com/postorder-stack.jpg)

## Morris遍历实现

上面两种算法对存储空间的要求都是`O(N)`, 其中迭代是利用方法栈调用的栈帧进行存储调用顺序, 而栈实现则是使用栈来存储节点的顺序. 而`Morris`算法则是利用二叉树内部的空余指针, 来存储节点的调用顺序, 从而达到空间存储的极致压缩`O(1)`.

核心思路就是在要遍历左子树的时候, 先找到左子树的最右点, 让他的`右子节点`(一定为null)指向当前节点. 这样当我们遍历完左子树的时候, 利用这个特殊的节点, 我们就可以返回调用的节点. 如此循环向下处理, 从而达到记录节点顺序的目的.

### Morris遍历-前序

使用`Morris`实现前序, 逻辑比较清楚明了, 核心思路:

- 从根节点开始依次遍历其所有左子节点, 进行以下处理:
- 先找到左子树的最右节点(存储在pre), 将最右节点的右节点指向当前节点.(方便后续处理完该左子树之后回退到父节点), 此时记录当前节点的值(这里保证`根节点`最先被记录, 保证其顺序为`根节点 -> 左节点`).
- 当当前节点的左子树为null时, 已经达到最左节点, 这时候跳转当前节点的右节点进行处理, 但是这时候就会回到当前节点的父节点(因为上一步已经将当前节点的右节点绑定到父节点上).
- 当我们回到父节点上, 我们再次查找当前左子树的最右节点, 就会回到父节点本身(因为已经形成一个环了), 这时候将当前左子树的最右节点连接清空, 恢复原样.
- 按照以上处理步骤依次处理所有的节点即可.

**Code**:

```java
public void preOrderByMorris(TreeNode root, List<Integer> result) {
  TreeNode current = root, pre = null;
  while (current != null) {
    pre = current.left;
    if (pre == null) {
      result.add(current.val);
      current = current.right;
      continue;
    }
    while (pre.right != null && pre.right != current) {
      pre = pre.right;
    }
    if (pre.right == null) {
      pre.right = current;
      result.add(current.val);
      current = current.left;
      continue;
    } else {
      pre.right = null;
      current = current.right;
    }
  }
}
```

**具体的处理流程图如下**:

![pre-order-morris](https://image.cjyong.com/preorder-moris.jpg)

### Morris遍历-中序

中序的逻辑和前序逻辑基本一致, 唯一区别的地方在于记录的点不同:

- 在到达最左节点的时候开始记录, 保证此时优先的顺序为`左节点`.
- 在记录完`左节点`之后, 在回退到达父节点的时候记录`根节点`, 保证`根节点`第二记录.
- 最后按照同样的逻辑处理右子节点即可. 即可保证: `左节点 -> 根节点 -> 右节点`.

**Code**:

```java
public void inOrderByMorris(TreeNode root, List<Integer> result) {
  TreeNode current = root, pre;
  while (current != null) {
    if (current.left == null) {
      result.add(current.val);
      current = current.right;
      continue;
    }
    pre = current.left;
    while (pre.right != null && pre.right != current) {
      pre = pre.right;
    }
    if (pre.right == null) {
      pre.right = current;
      current = current.left;
    } else {
      pre.right = null;
      result.add(current.val);
      current = current.right;
    }
  }
}
```

**具体的处理流程图如下**:

![in-order-morris](https://image.cjyong.com/inorder-morris.jpg)

### Morris遍历-后序

如果对`Morris`遍历的前序和中序很熟悉的话, 后序同样非常简单. 如果我们直接使用空余指针来存储左右子树的路径的话会非常复杂, 但是我们可以通过反转巧妙的利用前序来完成. 借用栈的思路:

- 我们的目标是`左节点 -> 右节点 -> 根节点`, 如果我们反转的话, 即: `根节点 -> 右节点 -> 左节点`, 这就是`Morris遍历前序`的一个变种, 只需要对`Morris前序遍历`进行稍稍修改即可:

  - 从根节点开始依次遍历其所有右子节点, 进行以下处理:
  - 先找到左子树的最左节点(存储在pre), 将最左节点的左节点指向当前节点.(方便后续处理完该右子树之后回退到父节点), 此时记录当前节点的值(这里保证`根节点`最先被记录, 保证其顺序为`根节点 -> 右节点`).
  - 当当前节点的右子树为null时, 已经达到最右节点, 这时候跳转当前节点的左节点进行处理, 但是这时候就会回到当前节点的父节点(因为上一步已经将当前节点的左节点绑定到父节点上).
  - 当我们回到父节点上, 我们再次查找当前左子树的最左节点, 就会回到父节点本身(因为已经形成一个环了), 这时候将当前左子树的最右节点连接清空, 恢复原样.
  - 按照以上处理步骤依次处理所有的节点即可.

- 这时候我们获取的结果即为: `根节点 -> 右节点 -> 左节点`, 我们只需要反转输出即可.

**Code**:

```java
public static void afterOrderByMorris(TreeNode root, List<Integer> result) {
  TreeNode current = root, pre;
  while (current != null) {
    pre = current.right;
    if (pre == null) {
      result.add(current.val);
      current = current.left;
      continue;
    }
    while (pre.left != null && pre.left != current) {
      pre = pre.left;
    }
    if (pre.left == null) {
      pre.left = current;
      result.add(current.val);
      current = current.right;
      continue;
    } else {
      pre.left = null;
      current = current.left;
    }
  }
  Collections.reverse(result);
}
```

**具体的处理流程图如下**:

![post-order-morris](https://image.cjyong.com/postorder-morris.jpg)

## 总结

二叉树是个常用的数据结构, 也是一个非常基础的数据结构, 很多后序的丰富的功能都离不开二叉树的拓展, 如缓存, 数据库, 搜索等, 因此掌握二叉树的基本属性是非常实用的. 本文对二叉树的遍历做了一次小结, 方便后序查阅和温习, 与君共勉.
