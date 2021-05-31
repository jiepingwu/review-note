#### [5. 最长回文子串](https://leetcode-cn.com/problems/longest-palindromic-substring/)

难度中等3684收藏分享切换为英文接收动态反馈

给你一个字符串 `s`，找到 `s` 中最长的回文子串。

 

**示例 1：**

```
输入：s = "babad"
输出："bab"
解释："aba" 同样是符合题意的答案。
```

**示例 2：**

```
输入：s = "cbbd"
输出："bb"
```

**示例 3：**

```
输入：s = "a"
输出："a"
```

**示例 4：**

```
输入：s = "ac"
输出："a"
```

 

**提示：**

- `1 <= s.length <= 1000`
- `s` 仅由数字和英文字母（大写和/或小写）组成

**这题是lc-647-回文子串的后续，647求所有回文子串的数量，显然，知道了全部的回文子串，肯定就能知道最长的回文子串。**

使用 DP 或者 中心拓展遍历，为了得到最床回文串，我们可以记录left and right 指针，而不需要记录整个子串，直接给代码。

```java
public class lc_5_longestPalindrome {
    // dp
    public String longestPalindrome(String s) {
        int n = s.length();
        boolean[][] dp = new boolean[n][n];
        int palindromeLeft = 0, palindromeRight = 0; // left and right pointer
        for (int i = n - 1; i >= 0; i--) {
            for (int j = i; j < n; j++) {
                dp[i][j] = s.charAt(i) == s.charAt(j) && (j - i < 2 || dp[i+1][j-1]);
                if (dp[i][j] && j - i > (palindromeRight - palindromeLeft)) {
                    palindromeLeft = i;
                    palindromeRight = j;
                }
            }
        }
        return s.substring(palindromeLeft, palindromeRight + 1);
    }

    // center loop 中心拓展遍历
    // 注意这里的写法：中心遍历其实还是挨个遍历，只是中心可能有1个或者2个字符。
    // 而 left = i，right = i 或者 i + 1。如果 s[left] == s[right]，继续往两边拓展。
    // 同样的，只需要记录两个指针使子串最长。
    public String longestPalindrome2(String s) {
        int len = s.length();
        int l = 0, r = -1;
        for (int i = 0; i < len; i++) {
            for (int j = 0; j <= 1; j++) {
                for (int left = i, right = i + j;
                     left >= 0 && right < len && s.charAt(left) == s.charAt(right);
                     left--, right++) {
                    if (right - left > r - l) {
                        l = left;
                        r = right;
                    }
                }
            }
        }
        return s.substring(l, r + 1);
    }
}
```

