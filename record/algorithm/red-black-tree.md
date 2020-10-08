# 红黑树详解

![red-black-1](https://image.cjyong.com/red-black-1.png)

一直想要写一篇关于红黑树的文章, 正好趁着十一佳节完成这个想法. 本文主要整理红黑树的插入和删除中各类情况的处理, 以及具体伪代码的实现, 未完待续(后序补充Java HashMap中的具体应用).

## 红黑树入门

红黑树是什么? 红黑树是一种特殊的二叉查找树, 是一种**自平衡的**二叉查找树. 具备如下几条特性:

- 每个节点都具备一个颜色(红色或者黑色).
- root节点一定为黑色, 所有的叶子节点(空指针节点)视作黑色.
- 相邻的父子节点不能连续为红色.(即红色节点的父子节点一定是黑色的, 但是黑色的节点父子节点可以为黑色).
- 从根节点到任意叶子节点(空指针)的路径中包含相同数量的黑色节点.

红黑树就是具备以上四条属性的特殊的二叉查找树. 为什么认为红黑树是一个自平衡的二叉查找树呢? 因为在一个节点数量为 `n` 的二叉树中, 红黑树的高度最大为: `2*log2(n+1)`. 而普通的二叉查找树, 是不满足这个特性的, 极限情况下, 高度为 `n`.

证明过程:

- [doctrina-prove](http://doctrina.org/maximum-height-of-red-black-tree.html)
- [geeks-prove](https://www.geeksforgeeks.org/red-black-tree-set-1-introduction-2/)
- [codes-dope-prove](https://www.codesdope.com/course/data-structures-red-black-trees/)

文章使用的数据结构为:

```java
public class RedBlackTreeNode {
  public RedBlackTreeNode left, right, parent;
  public int data;
  public boolean red;

  public RedBlackTreeNode(int data) {
    this.data = data;
    this.red = true;
  }

  public RedBlackTreeNode(int data, boolean red) {
    this.data = data;
    this.red = red;
  }

  public RedBlackTreeNode siblingNode() {
    if (parent == null) {
      return null;
    }
    return isOnParentLeft() ? parent.right : parent.left;
  }

  private boolean isOnParentLeft() {
    return this == parent.left;
  }

  public RedBlackTreeNode root() {
    RedBlackTreeNode current = this, currentParent = current.parent;
    while (currentParent != null) {
      current = currentParent;
      currentParent = current.parent;
    }
    return current;
  }

  private static RedBlackTreeNode rotateLeft(RedBlackTreeNode root, RedBlackTreeNode target) {
    // 将当前节点的右子节点进行保存(注意, 该节点需要左旋成为父节点)
    RedBlackTreeNode targetRight = target.right;
    // 将右子节点的左节点连接当前节点的右节点(之前的右子节点已经单独保存为 targetRight)
    target.right = targetRight.left;
    // 更新该节点的父节点(父亲节点已经变更了)
    if (target.right != null) {
      target.right.parent = target;
    }
    // 将右子节点父节点更新为父亲节点的父节点(提升为父节点)
    targetRight.parent = target.parent;
    // 如果当前左旋提升之后为根节点了, 需要更新root指针
    if (target.parent == null) {
      root = targetRight;
      // 更新父节点的父节点中关联的指针.
    } else if (target == target.parent.left) {
      target.parent.left = targetRight;
    } else {
      target.parent.right = targetRight;
    }
    // 将提升后的右子节点的左节点关联到之前的父节点
    targetRight.left = target;
    // 更新父亲节点的parent指针
    target.parent = targetRight;
    return root;
  }

  public static RedBlackTreeNode rotateRight(RedBlackTreeNode root, RedBlackTreeNode target) {
    // 将当前节点的左子节点进行保存(注意, 该节点需要右旋成为父节点)
    RedBlackTreeNode targetLeft = target.left;
    // 将左子节点的右节点连接当前节点的左子节点(之前的左子节点已经单独保存为 targetLeft)
    target.left = targetLeft.right;
    // 更新该节点的父节点(父亲节点已经变更了)
    if (target.left != null) {
      target.left.parent = target;
    }
    // 将左子节点父节点更新为父亲节点的父节点(提升为父节点)
    targetLeft.parent = target.parent;
    // 如果当前左旋提升之后为根节点了, 需要更新root指针
    if (target.parent == null) {
      root = targetLeft;
      // 更新父节点的父节点中关联的指针.
    } else if (target == target.parent.left) {
      target.parent.left = targetLeft;
    } else {
      target.parent.right = targetLeft;
    }
    // 将提升后的右子节点的右节点关联到之前的父节点
    targetLeft.right = target;
    // 更新父亲节点的parent指针
    target.parent = targetLeft;
    return root;
  }
}
```

## 红黑树插入

我们先从红黑树的插入开始入手, 我们插入逻辑的基础为:

- 插入的节点默认为红色节点
- 按照二叉查找树的情况进行插入, 即: 大节点放到右边, 小节点放到左边, 直到最后的叶子节点.

**Code**:

```java
private static RedBlackTreeNode insert(RedBlackTreeNode root, RedBlackTreeNode target) {
  if (root == null) {
    return target;
  }
  // 如果小于当前节点, 挂载到左节点中
  if (target.data < root.data) {
    root.left = insert(root.left, target);
    root.left.parent = root;
  } else if (target.data > root.data) {
    // 如果大于当前节点, 挂载到右节点中
    root.right = insert(root.right, target);
    root.right.parent = root;
  }
  // 如果等于当前节点, 则直接返回
  return root;
}
```

逻辑和二叉查找树插入逻辑保持一致, 依次查找到叶节点, 将当前节点挂载到叶节点中, 如果存在节点, 则跳过.

但是对于红黑树, 这样直接挂载可能会存在冲突: **挂载在一个红色节点下面, 形成了父亲节点和自己节点都是红色节点, 违背了性质3.** 这时候我们就需要采用一些措施来维护红黑树的黑节点高度平衡:

- Recoloring: 更新父亲和叔叔节点的颜色, 让其变成黑色, 这样就不会出现父子双红的情况了.
- Rotation: 通过左右旋, 将插入的节点进行提升, 同时更新父亲节点的颜色, 维护红黑树的节点平衡. [wiki详情](https://en.wikipedia.org/wiki/Tree_rotation)

接下来, 我们根据冲突情况的不同情况, 进行分析和处理.

已知的冲突为, 父亲节点和插入的子节点都为红色, 造成了父子双红的冲突情况.

这时候我们依据叔叔节点(父亲节点的兄弟节点)的颜色进行划分处理.

### 叔叔节点为红色

![red-insert-1](https://image.cjyong.com/red-black-insert-1.png)

在这种情况下: 我们插入的是 `X` 节点为红色, 父亲节点为 `P` 红色节点(造成了双红冲突), 其中叔叔节点为 `U` 节点是红色. 这时候我们需要使用到 `Recoloring` 操作来更新父亲和叔叔节点成黑色, 来解决双红的情况:

- 修改父亲节点和叔叔节点颜色为黑色(这样 `X` 节点和父亲节点 `P` 就不会形成双红)
- 修改爷爷节点为红色(之前, 爷爷节点一定为黑色, 因为父亲节点为红色, 爷爷节点肯定为黑色), 这样正好将叔叔和父亲新增的黑子节点抵消掉了, 同时保证了爷爷节点和父亲节点的差别(不会造成双红).
- 检查处理爷爷节点的情况, 因为爷爷节点变成了红色, 有可能因为曾爷爷节点也是红色, 而造成双红, 所以要依次向上处理.

**Code**:

```java
...
targetGrandParent.red = true;
targetParent.red = false;
targetUncle.red = false;
// 检查处理爷爷节点的情况
target = targetGrandParent;
...
```

### 叔叔节点为黑色

如果叔叔节点为黑色, 那么可以通过左右旋操作将父子两个红色节点, 分配给父亲子树和叔叔子树各一个, 这样就可以平衡两边子树的高度. 具体情况分为以下四种情况:

- 父亲节点在爷爷节点的左边, 插入节点也是父亲的左子节点, 简称 `Left Left Case`.
- 父亲节点在爷爷节点的左边, 插入节点是父亲的右子节点, 简称 `Left Right Case`.
- 父亲节点在爷爷节点的右边, 插入节点是父亲的右子节点, 简称 `Right Right Case`.
- 父亲节点在爷爷节点的右边, 插入节点也是父亲的左子节点, 简称 `Right Left Case`.

#### Left Left Case

![red-insert-2](https://image.cjyong.com/red-black-insert-2.png)

具体步骤为:

- 右旋, 将父亲节点提升为爷爷节点.
- 然后更新父亲节点 `P` 和 爷爷节点`G` 的颜色.

**Code**:

```java
...
root = rotateRight(root, targetGrandParent);
swapColors(targetParent, targetGrandParent);
target = targetParent;
...
```

#### Left Right Case

![red-insert-3](https://image.cjyong.com/red-black-insert-3.png)

具体步骤为:

- 左旋, 形成 Left Left Case, 然后使用 Left Left的解决方案, 注意这时候 `X` 和 `P` 的位置发生了变换.
- 右旋, 将当前节点提升为爷爷节点.
- 然后更新当前节点 `X` 和 爷爷节点`G` 的颜色.

**Code**:

```java
...
if (target == targetParent.right) {
  root = rotateLeft(root, targetParent);
  // 左旋之后, 父子的位置发生了更换, 记得更新.
  target = targetParent;
  targetParent = target.parent;
}

// left left case
root = rotateRight(root, targetGrandParent);
swapColors(targetParent, targetGrandParent);
target = targetParent;
...
```

#### Right Right Case

![red-insert-4](https://image.cjyong.com/red-black-insert-4.png)

同理 `Left Left Case`:

- 右旋, 将父亲节点提升为爷爷节点.
- 然后更新父亲节点 `P` 和 爷爷节点`G` 的颜色.

**Code**:

```java
root = rotateLeft(root, targetGrandParent);
swapColors(targetParent, targetGrandParent);
target = targetParent;
```

#### Right Left Case

![red-insert-5](https://image.cjyong.com/red-black-insert-5.png)

同理 `Left RIght Case`:

- 右旋, 形成 `Right Right Case`, 这时候 `X` 和 `P` 位置发生了替换.
- 左旋, 将当前节点提升为爷爷节点.
- 然后更新节点 `X` 和 爷爷节点`G` 的颜色.

**Code**:

```java
if (target == targetParent.left) {
  root = rotateRight(root, targetParent);
  // 右旋之后, 父子的位置发生了更换, 记得更新.
  target = targetParent;
  targetParent = target.parent;
}
// right right case.
root = rotateLeft(root, targetGrandParent);
swapColors(targetParent, targetGrandParent);
target = targetParent;
```

### 红黑树插入小结

到此二叉树的插入就结束了, 可见和普通的二叉查找树的插入唯一的区别就在于需要解决插入时 `双红` 的特殊情景. 这里将完整的插入代码更新到这里:

**Code**:

```java
public static RedBlackTreeNode insert(RedBlackTreeNode root, int data) {
  RedBlackTreeNode target = new RedBlackTreeNode(data);
  root = insert(root, target);
  return fixViolation(root, target);
}

private static RedBlackTreeNode insert(RedBlackTreeNode root, RedBlackTreeNode target) {
  if (root == null) {
    target.red = false;
    return target;
  }
  if (target.data < root.data) {
    root.left = insert(root.left, target);
    root.left.parent = root;
  } else if (target.data > root.data) {
    root.right = insert(root.right, target);
    root.right.parent = root;
  }
  return root;
}

private static RedBlackTreeNode fixViolation(RedBlackTreeNode root, RedBlackTreeNode target) {
  RedBlackTreeNode targetParent, targetGrandParent;
  while (target != root && target.red && target.parent.red) {
    targetParent = target.parent;
    targetGrandParent = targetParent.parent;
    //Case A: target parent is left child of grand parent
    if (targetParent == targetGrandParent.left) {
      RedBlackTreeNode targetUncle = targetGrandParent.right;
      // Case1\. current is red, uncle is also red, need recoloring
      if (targetUncle != null && targetUncle.red) {
        targetGrandParent.red = true;
        targetParent.red = false;
        targetUncle.red = false;
        target = targetGrandParent;
        // Case2\. current is red, uncle is black, need rotation
      } else {
        // current is right, parent is left, make right left case
        // rotateLeft make it left left.
        if (target == targetParent.right) {
          root = rotateLeft(root, targetParent);
          // after rotate left, right node become new parent
          target = targetParent;
          targetParent = target.parent;
        }

        // now it's left left case, need rotate right
        root = rotateRight(root, targetGrandParent);
        // swap parent and grand parent color
        swapColors(targetParent, targetGrandParent);
        target = targetParent;
      }
      // Case B: target parent is right child of grand parent,
    } else {
      RedBlackTreeNode targetUncle = targetGrandParent.left;
      // target parent and uncle both red, need recoloring
      if (targetUncle != null && targetUncle.red) {
        targetGrandParent.red = true;
        targetParent.red = false;
        targetUncle.red = false;
        target = targetGrandParent;
      } else {
        // it make right left case, need rotate right make it right right
        if (target == targetParent.left) {
          root = rotateRight(root, targetParent);
          target = targetParent;
          targetParent = target.parent;
        }
        // it become right right case.
        root = rotateLeft(root, targetGrandParent);
        // swap parent and grand parent color
        swapColors(targetParent, targetGrandParent);
        target = targetParent;
      }
    }
  }
  root.red = false;
  return root;
}

private static void swapColors(RedBlackTreeNode first, RedBlackTreeNode second) {
  boolean firstColor = first.red;
  first.red = second.red;
  second.red = firstColor;
}
```

## 红黑树删除

红黑树的插入主要是要解决插入之后, 形成的 `双红` 结局. 而红黑树的删除, 如果删除的节点为黑色节点, 则会导致红黑树高度发生变更, 这时候就需要进行平衡处理. 我们根据删除节点的状态分为以下三种情况:

- 删除节点为叶节点, 即没有左右子节点.
- 删除节点只有一个子节点.
- 删除节点存在两个子节点.

其中第三种情况可以转化为前两种情况, 如:

```java
// 如果我们想要删除根节点30, 存在两个子节点20,40
      30
     /  \
   20    40
  / \    / \
10  NIL 35  44

// 这时候, 我们只需要将 35节点值 替换到 30节点
     35
    /   \
  20     40
  / \    / \
10  NIL 30  44

// 这时候我们再删除 30 节点即可, 此时的30节点为叶子节点, 没有子节点.
      35
     /  \
   20    40
  / \    / \
10  NIL NIL 44

// 即我们只需要找到替换的节点即可
// 注意这里取35(大于当前节点的最小节点)进行更换, 可以保证二叉查找树的结构不会被破坏
```

所以当我们遇到第三种情况的时候, 我们只需要找到删除节点的替换节点(大于当前节点的最小节点或者小于当前节点的最大节点)即可, 交换替换节点的值, 删除替换节点即可, 然后替换节点一定是满足情况一或者情况二的. 所以最终情况, 一定是转换成情况一和情况二:

- 删除节点为叶节点, 即没有左右子节点.
- 删除节点只有一个子节点.
- 删除节点存在两个子节点 -> 转换为删除替换节点(1,2).

### 删除节点为没有子节点

当前节点没有子节点:

- 如果当前节点为红色, 直接删除节点即可.
- 如果当前节点时黑色, 删除之后, 黑色节点树发生变更, 需要进行调整, 这种情况简称双黑情况 `DoubleBlack`.

**Code**:

```java
// 如果是根节点, 设置根节点为null
if (target == root) {
  root = null;
} else {
  // 如果删除节点是黑色的, 就需要进行平衡处理
  if (!target.red) {
    root = fixDoubleBlack(root, target);
  }
  // 将当前删除节点从红黑树中移除
  if (target.isOnParentLeft()) {
    parent.left = null;
  } else {
    parent.right = null;
  }
  target.parent = null;
}
```

### 删除节点只有一个子节点

当前节点只有一个子节点, 需要注意的是, 这个子节点一定是红色的子节点, 结构为: `黑色父节点 + 红色子节点`. 因为另外一个子节点为 `NULL`, 则存活的子节点就一定是 `红色子节点`, 不然左右子树高度就不一致了. 因为子节点为红色, 其父节点就一定为黑色. 所以结构一定是: `黑色父节点 + 红色子节点`.

这时候直接将子节点替换当前节点, 并将子节点变黑即可:

**Code**:

```java
// replace 为唯一存在的子节点
// 将 replace 节点替换当前节点
if (target.isOnParentLeft()) {
  parent.left = replace;
} else {
  parent.right = replace;
}
replace.parent = parent;
target.parent = null;
replace.red = false;
`
```

### 完整的删除伪逻辑

**Code**:

```java
public static RedBlackTreeNode deleteByVal(RedBlackTreeNode root, int value) {
  if (root == null) {
    return null;
  }
  // 查找对应需要删除的节点
  RedBlackTreeNode target = search(root, value);
  // 如果找不到节点, 直接返回
  if (target == null) {
    System.out.println("No node found to delete with value: " + value);
    return root;
  }
  return deleteNode(root, target);
}

private static RedBlackTreeNode search(RedBlackTreeNode root, int value) {
  RedBlackTreeNode temp = root;
  while (temp != null) {
    if (temp.data == value) {
      return temp;
    }
    if (value < temp.data) {
      temp = temp.left;
    } else {
      temp = temp.right;
    }
  }
  return temp;
}

private static RedBlackTreeNode deleteNode(RedBlackTreeNode root, RedBlackTreeNode target) {
  // 查找替换节点, 无子节点时返回null, 单子节点时返回对应子节点, 双子节点时返回大于当前节点的最小节点
  RedBlackTreeNode replace = findReplaceNode(target);
  RedBlackTreeNode parent = target.parent;
  // 当前节点无子节点
  if (replace == null) {
    // 如果是根节点, 设置根节点为null
    if (target == root) {
      root = null;
    } else {
      // 如果删除节点是黑色的, 就需要进行平衡处理
      if (!target.red) {
        root = fixDoubleBlack(root, target);
      }
      // 将当前删除节点从红黑树中移除
      if (target.isOnParentLeft()) {
        parent.left = null;
      } else {
        parent.right = null;
      }
      target.parent = null;
    }
    return root;
  }

  // 当前节点只存在一个子节点, 结构为黑节点 + 红子节点
  if (target.left == null || target.right == null) {
    // 如果是根节点, 直接替换根节点
    if (target == root) {
      root.data = replace.data;
      root.left = root.right = null;
      replace.parent = null;
    } else {
      // 将子节点替换当前节点
      if (target.isOnParentLeft()) {
        parent.left = replace;
      } else {
        parent.right = replace;
      }
      replace.parent = parent;
      target.parent = null;
      replace.red = false;
    }
    return root;
  }
  // 如果存在两个子节点, 则替换当前节点和替换节点的值, 删除替换节点即可.
  swapValues(replace, target);
  return deleteNode(root, replace);
}

private static RedBlackTreeNode findReplaceNode(RedBlackTreeNode target) {
  if (target.left != null && target.right != null) {
    return successor(target.right);
  }
  if (target.left == null && target.right == null) {
    return null;
  }
  return target.left == null ? target.right : target.left;
}

private static RedBlackTreeNode successor(RedBlackTreeNode target) {
  RedBlackTreeNode temp = target;
  while (temp.left != null) {
    temp = temp.left;
  }
  return temp;
}
```

### 处理双黑情况

当我们删除一个叶子节点时, 如果当前节点是黑色就会出现 `双黑情况`, 这时候红黑树左右子树是不平衡的, 需要进行调整.

已知条件为删除节点为黑色叶节点, 这时候我们依据其兄弟节点的性质进行划分:

- 兄弟节点是黑色的, 包含一个子节点为红色.
- 兄弟节点是黑色的, 但是没有一个子节点为红色(常见情况为子节点都为NULL).
- 兄弟节点是红色的, 拥有两个黑色子节点

#### 黑兄弟节点含红色子节点

根据兄弟节点在父亲节点的左右侧和红色子节点在兄弟节点的左右侧, 可以划分为以下四种情况:

`Right Right Case`: 兄弟节点在父亲节点的右侧, 红色子节点在兄弟节点的右侧.

![red-black-delete-1](https://image.cjyong.com/red-black-delete-1.png)

具体操作步骤为:

- 更新右子节点颜色为黑色, 兄弟节点为父亲节点颜色.
- 左旋: 将兄弟节点提升为父节点, 右子点提升为兄弟节点, 父亲节点替代待删除节点
- 将父亲节点颜色变黑, 补充删除的黑色节点数量

**Code**:

```java
...
sibling.right.red = false;
sibling.red = parent.red;
root = rotateLeft(root, parent);
parent.red = false;
...
```

`Right Left Case`: 兄弟节点在父亲节点的右侧, 红色子节点在兄弟节点的左侧.

![red-black-delete-2](https://image.cjyong.com/red-black-delete-2.png)

具体操作步骤为:

- 将红色子节点更新为父亲节点颜色(后面替代父亲节点位置)
- 右旋, 将红色左子节点提升为兄弟节点
- 左旋, 将红色节点提升为父亲节点
- 将父亲节点颜色变黑, 补充删除的黑色节点数量

**Code**:

```java
...
sibling.left.red = parent.red;
root = rotateRight(root, sibling);
root = rotateLeft(root, parent);
parent.red = false;
...
```

**Left Left Case**: 兄弟节点在父亲节点的左侧, 红色子节点在兄弟节点的左侧.

具体步骤类似 `Right Right Case` :

- 更新左子节点颜色为黑色, 兄弟节点为父亲节点颜色.
- 右旋: 将兄弟节点提升为父节点, 左子点提升为兄弟节点, 父亲节点替代待删除节点
- 将父亲节点颜色变黑, 补充删除的黑色节点数量

**Code**:

```java
...
sibling.left.red = false;
sibling.red = parent.red;
root = rotateRight(root, parent);
parent.red = false;
...
```

**Left Right Case**: 兄弟节点在父亲节点的左侧, 红色子节点在兄弟节点的右侧.

具体步骤类似 `Right Left Case` :

- 将红色子节点更新为父亲节点颜色(后面替代父亲节点位置)
- 左旋, 将红色右子节点提升为兄弟节点
- 右旋, 将红色节点提升为父亲节点
- 将父亲节点颜色变黑, 补充删除的黑色节点数量

**Code**:

```java
...
sibling.right.red = parent.red;
root = rotateLeft(root, sibling);
root = rotateRight(root, parent);
parent.red = false;
...
```

#### 黑兄弟节点无红色子节点

![red-black-delete-3](https://image.cjyong.com/red-black-delete-3.png)

这时候需要划分两种情况:

- 如果父亲节点是红色的, 直接将父亲节点变黑(补充一个黑色节点高度), 兄弟节点变红即可(因为兄弟节点没有红色子节点, 可以变红减少一个黑节点高度).
- 如果父亲节点是黑色的, 这时候将兄弟节点变红, 处理父节点双黑情况即可.

**Code**:

```java
...
sibling.red = true;
if (!parent.red) {
  root = fixDoubleBlack(root, parent);
} else {
  parent.red = false;
}
...
```

#### 红色兄弟节点

![red-black-delete-4](https://image.cjyong.com/red-black-delete-4.png)

首先红色兄弟节点, 可以表明父亲节点为黑色节点, 兄弟节点的子节点一定是黑色子节点.

其次只需要对父亲节点进行左旋, 就会将兄弟的一个黑色子节点给删除节点的作为一个兄弟节点, 如图: 左旋之后, 就会将节点`25`给删除节点作为兄弟节点. 这时候就可以服用兄弟节点为黑色的情况进行处理即可:

具体处理步骤:

- 将兄弟节点变黑(后面提升为父亲节点), 父亲节点变红
- 左旋(在左边时进行右旋), 提升兄弟节点为父亲节点, 父亲节点替代删除节点位置
- 按照黑色兄弟节点的情况进行处理

```java
parent.red = true;
sibling.red = false;
if (sibling.isOnParentLeft()) {
  root = rotateRight(root, parent);
} else {
  root = rotateLeft(root, parent);
}
return fixDoubleBlack(root, target);
```

### 处理双黑情况小结

```java
private static RedBlackTreeNode fixDoubleBlack(RedBlackTreeNode root, RedBlackTreeNode target) {
  if (target == root) {
    return root;
  }
  RedBlackTreeNode sibling = target.siblingNode(), parent = target.parent;
  if (sibling == null) {
    // 无兄弟节点, 上传
    return fixDoubleBlack(root, parent);
  } else {
    // 红色兄弟节点
    if (sibling.red) {
      parent.red = true;
      sibling.red = false;
      if (sibling.isOnParentLeft()) {
        root = rotateRight(root, parent);
      } else {
        root = rotateLeft(root, parent);
      }
      // after rotate, target node sibling node become old sibling child, it must be black
      // it become target black and sibling black situation
      return fixDoubleBlack(root, target);
    } else {
      // 黑色兄弟节点情况
      if (sibling.hasRedChild()) {
        // 兄弟节点有红色节点情况
        if (sibling.left != null && sibling.left.red) {
          // sibling left child is red
          if (sibling.isOnParentLeft()) {
            // sibling in parent left
            // left left case, make sibling left be black, sibling become parent color
            // rotate right, make sibling become new parent
            sibling.left.red = false;
            sibling.red = parent.red;
            root = rotateRight(root, parent);
          } else {
            // sibling in parent right
            // right left case, make sibling left become parent color, after two rotation, it will become new parent
            sibling.left.red = parent.red;
            // rotate right, make sibling left become new sibling
            root = rotateRight(root, sibling);
            // rotate left, make new sibling become new parent
            root = rotateLeft(root, parent);
          }
        } else {
          // sibling right child is red
          if (sibling.isOnParentLeft()) {
            // sibling in parent left
            // left right case
            // sibling right become new parent
            sibling.right.red = parent.red;
            // rotate left, make sibling right become new sibling
            root = rotateLeft(root, sibling);
            // rotate right, make sibling become new parent
            root = rotateRight(root, parent);
          } else {
            // right right case, rotate left, make sibling become new parent
            // right child become black right child
            sibling.right.red = false;
            sibling.red = parent.red;
            root = rotateLeft(root, parent);
          }
        }
        // make parent be black node, make left height balance.
        parent.red = false;
      } else {
        // sibling has 2 black children, make sibling node be red(it can be, to make right height equal to left height)
        sibling.red = true;
        if (!parent.red) {
          // if parent is black two, it will change parent height, should fix it.
          root = fixDoubleBlack(root, parent);
        } else {
          // if parent is red, make it black, it can balance the tree
          parent.red = false;
        }
      }
    }
    return root;
  }
}
```

### 删除小结

由上面可以看出, 当我们对节点进行删除时, 最终处理都会转变为黑色叶子节点的删除. 而黑色叶节点的删除, 依据其兄弟节点的差异而进行不同的处理. 这也就是删除节点的难点所在. 其本质也都是在于当我们删除一个黑色节点的时候, 该如何进行补偿保证左右子树高度的平衡.

## 参考文档

- [binary-search-tree-wiki](https://brilliant.org/wiki/binary-search-trees/)
- [red-black-tree-wiki](https://brilliant.org/wiki/red-black-tree/)
- [geeksforgeeks-red-black-tree](https://www.geeksforgeeks.org/red-black-tree-set-1-introduction-2/)
