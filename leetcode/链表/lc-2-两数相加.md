#### [2. 两数相加](https://leetcode-cn.com/problems/add-two-numbers/)

难度中等6252收藏分享切换为英文接收动态反馈

给你两个 **非空** 的链表，表示两个非负的整数。它们每位数字都是按照 **逆序** 的方式存储的，并且每个节点只能存储 **一位** 数字。

请你将两个数相加，并以相同形式返回一个表示和的链表。

你可以假设除了数字 0 之外，这两个数都不会以 0 开头。

 

**示例 1：**

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2021/01/02/addtwonumber1.jpg)

```
输入：l1 = [2,4,3], l2 = [5,6,4]
输出：[7,0,8]
解释：342 + 465 = 807.
```

**示例 2：**

```
输入：l1 = [0], l2 = [0]
输出：[0]
```

**示例 3：**

```
输入：l1 = [9,9,9,9,9,9,9], l2 = [9,9,9,9]
输出：[8,9,9,9,0,0,0,1]
```

 

**提示：**

- 每个链表中的节点数在范围 `[1, 100]` 内
- `0 <= Node.val <= 9`
- 题目数据保证列表表示的数字不含前导零



#### Solution

##### Solution 1 

​	遍历链表，把 null 节点的值作为 0，最后如果还有进位，在add node(1)。

```java
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    ListNode head = new ListNode(-1);
    ListNode curr = head;
    
    int carry = 0;
    while (l1 != null || l2 != null) {
        int x = (l1 == null) ? 0 : l1.val;	// 节点值，null节点值为0
        int y = (l2 == null) ? 0 : l2.val;
        
        int sum = x + y + carry;
        int curNum = sum % 10; // 计算结果的个位
        carry = sum / 10;	// 计算结果的进位
        
        curr.next = new ListNode(curNum); // 连接链表
        curr = curr.next;
        
        if (l1 != null) l1 = l1.next; // 遍历下个节点
        if (l2 != null) l2 = l2.next;
    }
    
    if (carry == 1) curr.next = new ListNode(carry); // 还有进位
    return head.next;
}
```

##### Solution 2

​	递归。

 	1. 终止条件：
      	1. l1 == null and l2 == null and carry == 0时，遍历完成，return null
      	2. l1 或 l2 为 null && carry == 0 时，直接node.next指向另一个不为null的节点，return l2(l1)
	2. 递归返回值：返回相加的结果 sum % 10 的节点
	3. 注意：null节点的值为0

```java
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    return helper(l1, l2, 0);
}

private ListNode helper(ListNode l1, ListNode l2, int carry) {
    // terminate
    if (l1 == null && l2 == null && carry == 0) return null;
    if (l1 == null && l2 != null && carry == 0) return l2;
    if (l1 != null && l2 == null && carry == 0) reutrn l1;
    
    // process
    int sum = (l1 == null ? 0 : l1.val) + (l2 == null ? 0 : l2.val) + carry;
    carry = sum / 10;
    int curNum = sum % 10;
    ListNode node = new ListNode(curNum);
    node.next = helper(l1 == null ? null : l1.next, l2 == null ? null : l2.next, carry);
    
    return node;
}
```

