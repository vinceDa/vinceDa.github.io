---
categories: [algorithm]
tags: [algorithm]
---

在[字符串匹配（一）：BF、RK算法](https://vinceda.github.io/posts/2021-08-24-字符串匹配（一）：BF、RK算法)中我们认识了两种入门级别的字符串匹配算法：BF、RK算法；BF暴力匹配效率差，RK算法使用哈希算法，设计一个好用的哈希算法难度并不低，而且有退化的风险，一个工程级的算法是不允许出现退化严重的情况的；所以有没有一种字符串匹配算法，它的效率高并且不会存在效率退化的风险呢？当然是有的，Boyer-Moore算法就是其中之一。

在介绍BM算法之前，先从下图的示例开始说起

![](https://raw.githubusercontent.com/vinceDa/image-host/main/img/1630255871944-406265a7-5942-41d0-b799-a5082bb781e4.jpeg)

我们根据图中的主串和模式串可以一次性找到模式串需要挪动的位置，因为我们不需要像计算机一样机械般的一步一步的挪动去匹配；那么，我们怎么能让计算机也具备人脑的智慧呢？这就是BM算法需要解决的核心问题，那么BM算法是怎么解决这个问题的呢？

## 坏字符规则
将复杂的方法概括起来的形容词总是有点抽象，所以我们通过画图来解释一下什么叫**"坏字符规则"**。

![](https://raw.githubusercontent.com/vinceDa/image-host/main/img/1630255884378-4d905880-56e3-4da4-b302-ac791dc011aa.jpeg)

如图所示，我们将模式串和主串**倒着**进行匹配，碰到第一个和主串字符A不匹配的字符B时，我们将B在模式串中的下标记为bci(bad character index)，将A在模式串中的下标记为ei(exist index)，那么最大可移动位数则为bci-ei。为什么匹配的时候是倒着匹配的呢？我们通过下图解释一下。

![](https://raw.githubusercontent.com/vinceDa/image-host/main/img/1630255889649-f09a51ef-a58c-43c5-a47b-d1f28da52249.jpeg)

从图中很明显可以看出，倒序匹配是在找到最大可移动位数时为了避免移动过多的情况出现。

利用坏字符规则，BM 算法在最好情况下的时间复杂度非常低，是 $O(n/m)$。比如，主串是 aaabaaabaaabaaab，模式串是 aaaa。每次比对，模式串都可以直接后移四位，所以，匹配具有类似特点的模式串和主串的时候，BM 算法非常高效。

### 代码实现

1. 记录模式串的每个字符在模式串中所处的位置
```java
	/**
     * 记录模式串的每个字符在模式串中所处的位置
     * @param b 模式串
     * @param m 模式串的长度
     * @param bc 散列表
     */
    private void generateBC(char[] b, int m, int[] bc) {
        // 初始化bc数组
        for (int i = 0; i < SIZE; i++) {
            bc[i] = -1;
        }
        // 以每个字符的ascii码为key, 在模式串中的位置为值存储
        for (int i = 0; i < m; i++) {
            int ascii = b[i];
            // 如果存在相同字符, 总是用靠后的字符覆盖之前的, 保证移动位数不会过多, 避免错过可匹配的情况
            bc[ascii] = i;
        }
    }
```

2. 字符匹配的过程推进
```java
	/**
     * @param a 主串
     * @param n 主串长度
     * @param b 模式串
     * @param m 模式串长度
     * @return 第一次匹配上的位置
     */
    public int bm(char[] a, int n, char[] b, int m) {
        // 存储模式串中每个字符的位置
        int[] bc = new int[SIZE];
        generateBC(b, m, bc);
        // 循环比对, 直到i + m = n
        int i = 0;
        while (i <= n - m) {
            // 模式串倒序匹配
            int j;
            for (j = m - 1; j >= 0; j--) {
                // 坏字符在模式串中的位置为j
                if (a[i + j] != b[j]) {
                    break;
                }

            }
            // 匹配成功, 返回主串中和模式串相匹配的第一个字符的下标
            if (j < 0) {
                return i;
            }
            i = i + j - bc[a[i + j]];
        }
        return -1;
    }
```

但是，光靠坏字符规则还不足以实现BM算法。因为根据坏字符规则算出来的移动位数是有可能为**负数**的。

![](https://raw.githubusercontent.com/vinceDa/image-host/main/img/1630255908604-77c5419d-33db-4403-8063-237caad6bf9d.jpeg)

所以我们还需要用到好后缀原则。

## 好后缀规则
介绍完坏字符规则后，相信好后缀规则也没有那么"抽象"了，我们还是用一张图来开始好后缀规则的讲解。

![](https://raw.githubusercontent.com/vinceDa/image-host/main/img/1630508562549-d747d177-633f-4d33-b41d-5c1157f13bac.jpeg)

如上图所示，如果存在好后缀并且存在好后缀的匹配子串时，可以将模式串移动【坏字符下标-匹配子串起始下标】位来进行下一次匹配；如果不存在则直接移动【模式串长度】位进行下一次匹配。那么，这样就是完整的好后缀规则吗？我们再来看一张图。

![](https://raw.githubusercontent.com/vinceDa/image-host/main/img/1630508580004-bb28efaf-5cb7-4169-be8b-34ceaf33fa0c.jpeg)

从图中我们可以看出，从当前我们理解到的好后缀规则的用法来看，当好后缀同时也是模式串的前缀子串时，是会存在过度移动的情况的。我们需要解决这种场景才能将好后缀规则称为一个完整的解决方案。所以，我们实现好后缀的主要思路如下：

1. 模式串中存在和好后缀匹配的子串（以靠后的子串为准）时，应该怎么移动？

![](https://raw.githubusercontent.com/vinceDa/image-host/main/img/1630508589564-79d98dc0-2942-4558-b5b2-2e7e1b8ae0bf.jpeg)

2. 模式串中不存在和好后缀匹配的子串，但是好后缀中存在模式串的前缀子串时，应该怎么移动？
3. 如果上述两个条件都不满足时，应该怎么移动？

![](https://raw.githubusercontent.com/vinceDa/image-host/main/img/1630508658548-e7628402-772c-401f-add5-284e7df98553.jpeg)



### suffix和prefix
根据刚才提出的思路，我们通过suffix和prefix两个数组来实现这个思路。举个例子，suffix[k]=i。suffix数组记录的是匹配上的子串的起始位置，其中k表示好后缀的长度，i表示模式串中子串和好后缀匹配时的下标的起始位置。preffix[k]=true。preffix数组记录的是好后缀是否为模式串的前缀子串。如果我描述的不是很明白的话，我们直接通过下面这个图来理解吧。

![](https://raw.githubusercontent.com/vinceDa/image-host/main/img/1630508676442-5e30cf3e-41bd-435e-a202-600ec89d92a5.jpeg)

理清楚所有流程后，我们开始代码实现吧。

### 代码实现

1. 填充suffix和prefix
```java
 /**
     * 记录suffix和prefix，供后续比对使用
     * @param b 模式串
     * @param m 模式串的长度
     * @param prefix 记录每个好后缀是否同时是模式串的前缀
     * @param suffix 记录每个好后缀在模式串中所匹配到的字符串的起始下标，如果存在多个则记录靠后的下标避免过度移动，如果不存在则记为-1
     */
    private void generateGS(char[] b, int m, boolean[] prefix, int[] suffix) {
        // 初始化bc数组
        for (int i = 0; i < m; i++) {
            prefix[i] = false;
            suffix[i] = -1;
        }

        for (int i = 0; i < m - 1; i++) {
            int j = i;
            // 好后缀的长度
            int k = 0;
            // 在匹配上第一个相同字符后， j--表示向前做进一步匹配, 尝试获取最大长度的好后缀
            // 例如acbecb中, 末尾的b匹配成功后向前一位判断c是否匹配, 如此循环下去
            while (j >= 0 && b[j] == b[m - 1 - k]) {
                j--;
                k++;
                // j表示坏字符的位置, j+1表示好后缀的起始位置,
                suffix[k] = j + 1;
            }
            // 说明j从好后缀的末端一直到模式串的起始位置都是匹配成功的,
            // 也就是说好后缀同时也是模式串的前缀子串
            if (j == -1) {
                prefix[k] = true;
            }
        }
    }
```

2. 根据不同情况计算移动位数
```java
 /**
     * 根据不同情况计算移动位数
     * 1、存在匹配的子串
     * 2、不存在匹配子串但是好后缀为模式串前缀子串
     * @param j 坏字符对应的模式串中的字符下标
     * @param m 模式串长度
     */
    private int moveByGS(int j, int m, int[] suffix, boolean[] prefix) {
        // 好后缀长度(好后缀起始位置的前一位一定是坏字符)
        int k = m - 1 - j;
        // 1、如果存在多个, 这个好后缀肯定是靠后的位置的下标(查看generateGS代码得出)
        if (suffix[k] != -1) {
            return j - suffix[k] + 1;
        }
        // 2、不满足条件1则判断所有的好后缀中是否存在为模式串的前缀子串的情况
        // j+1是好后缀，j+2是好后缀的后缀子串的起始(避免出现过渡移动导致错失匹配的情况)
        for (int r = j+2; r <= m - 1 ; r++) {
            if (prefix[m - r]) {
                return r;
            }
        }
        // 3、不满足上述条件说明所有的好后缀中不存在模式串的前缀子串, 所以直接移动m位
        return m;
    }
```

现在两种规则的代码实现都有了，那么怎么选择坏字符和好后缀规则？

答案很简单，那个规则移动的位数多就用哪个，所以我们把两个规则的代码融合一下。

```java
   /**
     *
     * @param a 主串
     * @param n 主串长度
     * @param b 模式串
     * @param m 模式串长度
     * @return 第一次匹配上的位置
     */
    public int bm(char[] a, int n, char[] b, int m) {
        // 初始化模式串
        int[] bc = new int[SIZE];
        generateBC(b, m, bc);
        int[] suffix = new int[m];
        boolean[] prefix = new boolean[m];
        generateGS(b, m, suffix, prefix);
        // 模式串与主串匹配的下标
        int i = 0;
        while (i <= n - m) {
            int j;
            for (j =  m - 1; j >= 0; j--) {
                if (a[i + j] != b[j]) {
                    break;
                }
            }
            // 匹配成功(返回主串与模式串第一个匹配的字符的位置)
            if (j < 0) {
                return i;
            }
            // 移动si - xi的位置(这里等同于将模式串往后滑动j-bc[(int)a[i+j]]位)
            int x = j - bc[a[i + j]];
            int y = 0;
            // 如果有好后缀
            if (j < m - 1) {
                y = moveByGS(j, m, suffix, prefix);
            }
            System.out.println("x: " + x + "   y:" + y);
            i = i + Math.max(x, y);
        }
        return -1;
    }
```

这样，完整的BM算法的代码实现已经完成了。


## BM算法的性能分析及优化

整个算法额外用到了3个数组。bc数组大小和字符集大小有关。suffix和prefix数组和模式串长度有关；

如果对内存要求十分严格，可以只使用好后缀规则，但是性能会下降一点；

> 实际上，BM 算法的时间复杂度分析起来是非常复杂，这篇论文“[A new proof of the linearity of the Boyer-Moore string searching algorithm](https://dl.acm.org/doi/10.1109/SFCS.1977.3)”证明了在最坏情况下，BM 算法的比较次数上限是 5n。这篇论文“[Tight bounds on the complexity of the Boyer-Moore string matching algorithm](https://dl.acm.org/citation.cfm?id=127830)”证明了在最坏情况下，BM 算法的比较次数上限是 3n。你可以自己阅读看看。
> --- 《数据结构与算法》 王争


## 总结
今天我们学习了十分复杂的BM算法，它的主要核心在于怎么移动尽可能多的位数提高匹配效率。为了实现这个目标就定义了两个规则：坏字符和好后缀。这两个规则互相搭配使得BM的性能很优秀，其中suffix和prefix的设计十分巧妙，值得我们借鉴。如果没有看懂的可以多看几遍或者查看其他资料帮助理解。在这里，我强烈推荐大家去看看王争的极客时间专栏《数据结构与算法》，通俗易懂，讲得很透彻。

# 参考资料

[王争 —— 数据结构与算法之美](https://time.geekbang.org/column/article/71525)

[算法4](https://item.jd.com/11098789.html)
