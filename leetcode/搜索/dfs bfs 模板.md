DFS

##### 递归

```java
public class Solution {
    private static class Node {
        /**
         * 节点值
         */
        public int value;
        /**
         * 左节点
         */
        public Node left;
        /**
         * 右节点
         */
        public Node right;

        public Node(int value, Node left, Node right) {
            this.value = value;
            this.left = left;
            this.right = right;
        }
    }

    public static void dfs(Node treeNode) {
        if (treeNode == null) {
            return;
        }
        // 遍历节点
        process(treeNode)
        // 遍历左节点
        dfs(treeNode.left);
        // 遍历右节点
        dfs(treeNode.right);
    }
}
```

##### Stack

```java
/**
 * 使用栈来实现 dfs
 * @param root
 */
public static void dfsWithStack(Node root) {
    if (root == null) {
        return;
    }
    Stack<Node> stack = new Stack<>();
    // 先把根节点压栈
    stack.push(root);
    while (!stack.isEmpty()) {
        Node treeNode = stack.pop();
        // 遍历节点
        process(treeNode)

        // 先压右节点
        if (treeNode.right != null) {
            stack.push(treeNode.right);
        }

        // 再压左节点
        if (treeNode.left != null) {
            stack.push(treeNode.left);
        }
    }
}
```



BFS

##### Queue

```java
/**
 * 使用队列实现 bfs
 * @param root
 */
private static void bfs(Node root) {
    if (root == null) {
        return;
    }
    Queue<Node> stack = new LinkedList<>();
    stack.add(root);

    while (!stack.isEmpty()) {
        Node node = stack.poll();
        System.out.println("value = " + node.value);
        Node left = node.left;
        if (left != null) {
            stack.add(left);
        }
        Node right = node.right;
        if (right != null) {
            stack.add(right);
        }
    }
}
```

