几道适合用动态规划求解的经典题目。

## 爬楼梯

假设总共n个台阶的楼梯。每次可以走1步或者2步。爬上楼梯总共有多少种不同的走法？

这道题，假设把最终的答案记为f(n)，表示n阶的台阶有多少种走法。那么

    f(n) = f(n-1) + f(n-2)

即，第一次走的是1步，剩下n-1级台阶有多少种走法，加上第一次走的是2步，剩下n-2级台阶有多少种走法。

其中f(1)=1，f(2)=2，表示1阶台阶时只是走一步，2阶台阶时，可以有2个1步或者1个2步两种走法。

## 最大连续子数组和

在一个数组中，找出和最大的连续子数组。例如，对于数组 [−2,1,−3,4,−1,2,1,−5,4]， 连续子数组 [4,−1,2,1] 有最大的和6。

将数组中前k项的最大连续子数组和记为f(k)。假设我们已经处理了k-1个，正在处理第k个。那么f(k)跟f(k-1)有什么关系？它必然是下列三种情况中最大的那个：

1. 前k-1项的最大子数组和
2. 当前数组第k项
3. 当前第k项，加上前面连续的若干项

第二种跟第三种可以合并，从第k项开始往前面的的若干项。如果第k项前面"若干"项总和是负的，那么就是数组第k项。求f(k)可以写成：

    f(k) = max( f(k-1), sum(i,k) )

sum(i,k)表示从第i项连续加到第k项。具体实现时维护几个变量，记录f(k-1)的最大值，记录从第i项到第k项的连续数组和，如果它小于零后重新计算。一遍扫描即可求出f(n)。

## 最长公共子序列

如果字符串一的所有字符按其在字符串中的顺序出现在另外一个字符串二中，则字符串一称之为字符串二的子串。注意，并不要求子串（字符串一）的字符出现在字符串二中必须**连续**。

请编写一个函数，输入两个字符串，求它们的最长公共子串。例如：输入两个字符串BDCABA和ABCBDAB，字符串BCBA和BDAB都是是它们的最长公共子序列，则输出它们的长度4，并打印任意一个子序列。

解这道题，仍然是找到n与n-1之间的关系。假设第一个串是x[1]...x[n]，第二个串是y[1]...y[m]，那么，如果x[n]等于y[m]，最长公共子串是x[1]...x[n-1]与y[1]...y[m-1]的最长公共子串，加上x[n]。如果x[n]不等于y[m]的情况，则分别求x[1]...x[n-1]与y[1]...y[m]的最长公共子串，以及x[1]...x[n]与y[1]...y[m-1]的最长公共子串，两者中较长的那个。

用f(n, m)表示最长公共子序列的长度，则：

    f(n, m) = f(n-1, m-1) + 1, 当x[n] == y[m]
    f(n, m) = max( f(n, m-1), f(n-1, m)), 当x[n] != y[m]

初始条件，m或者n等于1时，f(n, m)为0。

如果直接套公式递归，会有大量的重复计算。可以用一个矩阵来保存中间计算结构，按迭代来计算。

## 编辑距离

两个单词word1和word2，求从word1变化到word2最少需要多少步。每次操作算一步。允许下列三种操作：

1. 插入一个字符
2. 删除一个字符
3. 替换一个字符

这个题目，我们可以用f(i,j)表示从word1的前i个，变到word2的前j个需要的最小步数--编辑距离。

从word1前i-1项变到word2的前j项，需要最少f(i-1, j)步。在此的基础上，只需要word[1..i-1]再插入一个字符，可以得到f(i, j)。

通过f(i-1, j-1)可以将word1[1..i-1]和word2[1..j-1]变到一模一样。在此基础上，我们只需要对接下来一个字符做替换操作，可以得到f(i, j)。

对于f(i, j-1)，换另一个角度看，将word2删除一个字符可以得到f(i, j)。

由于编译距离是最少的变化，于是f(i, j)应该由以上三种情况中最小的一种得到。

    f(i, j) = min( f(i-1, j)+1, f(i-1, j-1)+?, f(i, j-1)+1)

其中?是0或者1，如果word1[i]等于word2[j]了，则不需要做替换操作，所以?是0。否则要word1[i]替换到word2[j]，?是1。

看一下初始条件，f(0,0)肯定是0的。f(i,0)需要原始字符串做i次插入操作，所以f(i,0)等于i。f(0,j)需要对原始字符串做j次删除操作，所以f(0,j)等于j。

## 卖股票的最佳时间

假设给你一个数组，其中第i个元素是股票在第i天的价格。可以最多进行k次交易，设计一个算法求最大利润。注意，两次交易之间不能交叉，即买进前要先卖出。

这道题在leetcode上面是hard级别，不算经典。不过正是因为偏难，所以放在最后作为本文的结束吧。

跟最大连续子数组和的那个题目类似，这里分三种情况讨论：

* 到第i天，进行了不到k次(最多k-1次)交易，已经达到最大收益。
* 进行了k次交易，但是在第i天之前(i-1之前的某天)，已经达到最大收益。
* 刚好在第i天进行第k次交易，达到最大收益。

用f(k, i)表示当前到达第i天，进行最多k次交易，可以取得的最大利润。于是根据上面的三种情况有：

    f(k, i) = max( f(k-1, i), f(k, i-1), f(k-1, j-1)+price[i]-price[j])

可以将最后一项，f(k-1, j-1)-price[j]单独维护，因为只要维护一个最大值，可以边遍历边计算。

小结一下，这类题目都有的特点：最优子结构，重叠子问题。

由n-1的最优解，可以根据一个状态转移方程，还推算n的最优解。求解这类问题最关键的是，找到状态转移的推导关系。在求解中子问题的解可以打表保存起来，递归变成迭代求解。
