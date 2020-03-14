# 单链表经典算法题

单链表是非常常见的数据结构, 这里选取三道有关单链表的经典算法题. 虽然简单, 但是思路值得借鉴. 为了统一情况, 统计将链表结构固定如下:

```java
class SinglyLinkedListNode {
  int data;
  SinglyLinkedListNode next;
}
```

## Find Merge Point of Two Lists

找到两个链表的交汇点, 如下图所示两个链表相交于`x`点, 找出两个链表的相交点`x`.

```java
[List #1] a--->b--->c
                     \
                      x--->y--->z--->NULL
                     /
     [List #2] p--->q
```

### Think Space1

由于相交点相对两个链表位置各不相同, 无法保证同步前进可以同时达到相交点. 这时候有两种思路:

- 使用暴力穷尽法, 对链表一中的每一个节点, 都去遍历链表二, 知道找到相同节点为止. 时间复杂度高, `O(n^2)`.
- 将链表一结尾与链表二开头接连, 链表二结尾与链表一开头接连, 同时顺序遍历两个链表, 这样可以保证两个链表在到达第二次交汇点`x`的时候一定相遇(因为第二次走到交汇点时, 双方链表走的节点数是完全一样, 一定会相遇).

### Solution1

```java
static int findMergeNode(SinglyLinkedListNode head1, SinglyLinkedListNode head2) {
    SinglyLinkedListNode currentA = head1, currentB = head2;
    while(currentA != currentB) {
        if (currentA.next == null) {
            currentA = head2;
        } else {
            currentA = currentA.next;
        }

        if (currentB.next == null) {
            currentB = head1;
        } else {
            currentB = currentB.next;
        }
    }
    return currentA.data;
}
```

## Detect a Cycle

检测单链表中是否存在环状结构.

### Think Space2

解决方案非常简单就是采用追及方案: 同一个起点, 选取两位选手向前遍历, 一位选手每次步行一个节点, 另一个选手每次步行两个节点. 如果链表中存在环状, 两个选手肯定会相遇.

**证明:**:

- 假设选手A(步行1步)到达到达环状结构时与选手B(步行两步)的距离为`S`(因为选手B速度快, 肯定已经先到达环里, 并在里面一直转圈).
- 假设环的长度为`C(C > S)`.
- 假设在后面的时间`T`内, 两位选手会相遇.

即在`T秒后`, `选手B`行走的距离为: `2T`, `选手A`行走的距离为: `T`. 两者会交汇于一点: `2T - T - S = nC(n为大于等于1的整数)`. 简化为: `T = nC + S`, 即对于任何`C`和`S`都存在一个最小时间点`T = C + S`两者会相遇.

### Solution2

```java
boolean hasCycle(SinglyLinkedListNode head) {
    if (head == null) {
        return false;
    }

    SinglyLinkedListNode slow = head, fast = head.next;
    while (slow != fast) {
        if (slow == null || fast == null || fast.next == null) {
            return false;
        }
        slow = slow.next;
        fast = fast.next.next;
    }
    return true;
}
```

## Reverse a linked list

逆转一个链表, 如 `head -> 1 -> 2 -> 3 -> null`逆转之后成为: `head -> 3 -> 2 -> 1 -> null`.

### Think Space3

思路很多, 然而这里采用一个递归的方式, 详情请参照代码注释.

### Solution 3

```java
static SinglyLinkedListNode reverse(SinglyLinkedListNode head) {
    // 如果当前节点已经到达尾部, 返回当前节点(即最后的remain节点: 原来的终点)
    if (head == null || head.next == null) {
        return head;
    }

    // 对于每一个节点, 依次对其子节点调用reverse方法进行操作
    SinglyLinkedListNode remain = reverse(head.next);
    // 将子节点的next指向更改为自己(修改next的方向)
    head.next.next = head;
    // 将指向子节点的next指针置为null
    head.next = null;
    // 返回最终节点
    return remain;
}
```
