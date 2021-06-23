#### [25. K 个一组翻转链表](https://leetcode-cn.com/problems/reverse-nodes-in-k-group/)

难度困难1144收藏分享切换为英文接收动态反馈

给你一个链表，每 *k* 个节点一组进行翻转，请你返回翻转后的链表。

*k* 是一个正整数，它的值小于或等于链表的长度。

如果节点总数不是 *k* 的整数倍，那么请将最后剩余的节点保持原有顺序。

**进阶：**

- 你可以设计一个只使用常数额外空间的算法来解决此问题吗？
- **你不能只是单纯的改变节点内部的值**，而是需要实际进行节点交换。

 

**示例 1：**

![img](https://assets.leetcode.com/uploads/2020/10/03/reverse_ex1.jpg)

```
输入：head = [1,2,3,4,5], k = 2
输出：[2,1,4,3,5]
```

**示例 2：**

![img](https://assets.leetcode.com/uploads/2020/10/03/reverse_ex2.jpg)

```
输入：head = [1,2,3,4,5], k = 3
输出：[3,2,1,4,5]
```

**示例 3：**

```
输入：head = [1,2,3,4,5], k = 1
输出：[1,2,3,4,5]
```

**示例 4：**

```
输入：head = [1], k = 1
输出：[1]
```



**提示：**

- 列表中节点的数量在范围 `sz` 内
- `1 <= sz <= 5000`
- `0 <= Node.val <= 1000`
- `1 <= k <= sz`



Solution 1

递归1

```java
public ListNode reverseKGroup(ListNode head, int k) {
        ListNode tail = head;
        for (int i = 0; i < k; i++) {
            //剩余数量小于k的话，则不需要反转。
            if (tail == null) {
                return head;
            }
            tail = tail.next;
        }
        // 反转前 k 个元素
        ListNode newHead = reverse(head, tail);
        //下一轮的开始的地方就是tail
        head.next = reverseKGroup(tail, k);
        return newHead;
    }

    private ListNode reverse(ListNode head, ListNode tail) {
        ListNode pre = null;
        ListNode next = null;
        ListNode curr = head;
        while (curr != tail) {
            next = curr.next;
            curr.next = pre;
            pre =curr;
            curr = next;
        }
        return pre;
    }
```

递归2

```java
class Solution {
    public ListNode reverseKGroup(ListNode head, int k) {
        ListNode tail = head;
        int count = 0;
        while (tail != null && count != k) {
            tail = tail.next;
            count++;
        }
        ListNode pre = null;
        if (count == k) {
            pre = reverseKGroup(tail, k);
            while (count != 0) {
                count--;
                ListNode tmp = head.next;
                head.next = pre;
                pre = head;
                head = tmp;
            }
            head = pre;
        }
        return head;
    }
}
```

Solution 2

栈

```java
class Solution {
    public ListNode reverseKGroup(ListNode head, int k) {
        Deque<ListNode> stack = new ArrayDeque<ListNode>();
        ListNode dummy = new ListNode(0);
        ListNode p = dummy;	// 用于连接节点
        while (true) {
            int count = 0;
            ListNode tail = head;
            while (tail != null && count < k) {	// K个一组加入stack中
                stack.add(tail);
                tail = tail.next;
                count++;
            }
            if (count != k) {	// 如果不足k个，p直接连上head节点，break
                p.next = head;
                break;
            }
            while (!stack.isEmpty()){	// 出栈，连接节点
                p.next = stack.pollLast();
                p = p.next;
            }
            head = tail;	// 更新头节点为下一组节点头
        }
        return dummy.next;
    }
}
```

