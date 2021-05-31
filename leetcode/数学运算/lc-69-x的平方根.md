#### [69. x 的平方根](https://leetcode-cn.com/problems/sqrtx/)

难度简单683收藏分享切换为英文接收动态反馈

实现 `int sqrt(int x)` 函数。

计算并返回 *x* 的平方根，其中 *x* 是非负整数。

由于返回类型是整数，结果只保留整数的部分，小数部分将被舍去。

**示例 1:**

```
输入: 4
输出: 2
```

**示例 2:**

```
输入: 8
输出: 2
说明: 8 的平方根是 2.82842..., 
     由于返回类型是整数，小数部分将被舍去。
```

通过次数315,843

提交次数804,729



#### Solution

##### Solution1 

牛顿迭代法

假设x0为结果，x0的更新点 X0 为 曲线y = x^2 - C 过点(x0, x0^2 - C)的切线与 x轴相交的点。

过程：设 切线为 y = ax + b ，y = 2 * x0 * x + b , x0 ^ 2 - C = 2 * x0 * x0 + b，b = -x0 ^ 2 - C, 直线y = 2x0 * x - x0 ^ 2 - C，则 x = (x0^2 + C) / 2x0 = 0.5 * (x0 + C / x0)；

计算可得 **X0  = 0.5 * (x0 + x / x0)**

```java
public int mySqrt(int x) {
    double x0 = x;
    while (Math.abs(x0 * x0 - x) > 1e-6) {
        x0 = 0.5 * (x0 + x / x0);
    }
    return (int)x0;
}
```



##### Solution 2

二分法

```java
class Solution {
    public int mySqrt(int x) {
        int l = 0, r = x, ans = -1;
        while (l <= r) {
            int mid = l + (r - l) / 2;
            if ((long) mid * mid <= x) {
                ans = mid;
                l = mid + 1;
            } else {
                r = mid - 1;
            }
        }
        return ans;
    }
}
```

##### Solution 3

使用 exp 和 ln 代替 sqrt:

![image-20210530133821638](C:\Users\wwwjp\AppData\Roaming\Typora\typora-user-images\image-20210530133821638.png)

注意： 由于计算机无法存储浮点数的精确值（浮点数的存储方法可以参考 IEEE 754，这里不再赘述），而指数函数和对数函数的参数和返回值均为浮点数，因此运算过程中会存在误差。例如当 x = 2147395600x=2147395600 时，e^(0.5ln(x))  的计算结果与正确值 4634046340 相差 10^{-11} ，这样在对结果取整数部分时，会得到 4633946339 这个错误的结果。

因此在得到结果的整数部分 \textit{ans}ans 后，我们应当找出 \textit{ans}ans 与 \textit{ans} + 1ans+1 中哪一个是真正的答案。

```java
public int mySqrt(int x) {
    if (x == 0) return 0;
    int ans = (int) Math.exp(0.5 * Math.log(x));	// 使用exp 和 ln 简介的求出 sqrt()
    return (long) (ans + 1) * (ans + 1) <= x ? ans + 1 : ans;
}
```

