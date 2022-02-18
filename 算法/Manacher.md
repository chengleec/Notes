#### 题目描述

给定一个字符串 s，找到 s 中最长的回文子串。

#### 暴力法

求出每一个子串，之后判断是不是回文，找到最长的那个。求每一个子串时间复杂度O(n^2^)，判断子串是不是回文O(n)，两者是相乘关系，所以时间复杂度为O(N^3)。

```java
public String palindromic(String s){
    String res = "";
    int start = 0, end = 0;
    //起始地址
    for (int i = 0; i < s.length(); i++) {
        //结束地址
        for (int j = i+1; j < s.length(); j++){
            start = i;
            end = j;
            while(start < end && s.charAt(start) == s.charAt(end)){
                start ++;
                end --;
            }
            if (start >= end && j - i + 1 > res.length()){
                res = s.substring(i,j+1);
            }
        }
    }
    return res;
}
```

#### 中心扩展法

中心扩展就是把给定的字符串的每一个字母当做中心，向两边扩展。算法时间复杂度为O(n^2^)。
对长度为奇数的回文串和长度为偶数的回文串，要分别进行求解。

```java
public String palindromic2(String s){
    String res = "";
    for(int i = 0; i < s.length(); i++){
        String s1 = palindromic2Help(s, i, i);
        res = s1.length() > res.length() ? s1 : res;
        String s2 = palindromic2Help(s, i,i + 1);
        res = s2.length() > res.length() ? s2 : res;
    }
    return res;
}
public String palindromic2Help(String s, int start, int end){
    while(start > 0 && end < s.length() && s.charAt(start) == s.charAt(end)){
        start --;
        end ++;
    }
    return s.substring(start + 1, end);
}
```

#### Manacher

首先，Manacher算法提供了一种巧妙地办法，将长度为奇数的回文串和长度为偶数的回文串一起考虑，具体做法是，在原字符串的每个相邻两个字符中间插入一个分隔符，同时在首尾也要添加一个分隔符，分隔符的要求是不在原串中出现，一般情况下可以用 # 号。下面举一个例子：

<img src="/Users/licheng/Documents/Typora/Picture/image-20200501105639950.png" alt="image-20200501105639950" style="zoom:67%;" />

##### Len 数组简介与性质

Manacher 算法用一个辅助数组 Len[i] 表示以字符 T[i] 为中心的最长回文字串的最右字符到T[i]的长度，比如以 T[i] 为中心的最长回文字串是 T[l,r]，那么 Len[i] = r - i + 1。

对于上面的例子，可以得出 Len[i] 数组为:

<div align=center> 
	<img src="C:\Users\admin\Typora\Picture\image-20200501105605201.png" alt="image-20200501105605201" style="zoom: 67%;"/> 
</div> 

Len 数组有一个性质，那就是 Len[i]-1 就是该回文子串在原字符串 S 中的长度。有了这个性质，那么原问题就转化为求所有的 Len[i]。下面介绍如何在线性时间复杂度内求出所有的 Len。

##### Len 数组的计算

设 P 为之前计算中最长回文子串的右端点的最大值，并且设取得这个最大值的中心位置为 P~0~。

* 如果 i <= P，那么找到 i 相对于 P~0~ 的对称位置，设为 j。

  <img src="/Users/licheng/Documents/Typora/Picture/image-20200501110057646.png" alt="image-20200501110057646" style="zoom:67%;" />

  1. 如果 Len[j] < P - i，那么说明以 j 为中心的回文串一定在以 P~0~ 为中心的回文串的内部，且 j 和 i 关于位置 P~0~ 对称，所以以 i 为中心的回文串的长度至少和以 j 为中心的回文串一样，即 Len[i] >= Len[j]。因为 Len[j] < P - i，所以说 i + Len[j] < P，由对称性可知 Len[i] = Len[j]。

     <img src="/Users/licheng/Documents/Typora/Picture/image-20200501110119246.png" alt="image-20200501110119246" style="zoom:67%;" />

  2. 如果 Len[j] >= P - i，由对称性，说明以 i 为中心的回文串可能会延伸到P之外，而大于P的部分我们还没有进行匹配，所以要从 P + 1 位置开始一个一个进行匹配，直到发生失配，从而更新 P、P~0~ 以及 Len[i]。

* 如果 i > P，说明对于中心为 i 的回文串还一点都没有匹配，这个时候，就只能老老实实地一个一个匹配了，匹配完成后要更新 P、P~0~ 以及 Len[i]。

<img src="/Users/licheng/Documents/Typora/Picture/image-20200501110137702.png" alt="image-20200501110137702" style="zoom:67%;" />

##### 时间复杂度分析

因为算法只有遇到还没有匹配的位置时才进行匹配，已经匹配过的位置不再进行匹配，所以对于字符串 T 中的每一个位置，只进行一次匹配，所以 Manacher 算法的总体时间复杂度为 O(n)。

```java
public String palindromic3(String s){

    StringBuffer str = new StringBuffer(s);
    //填充字符串
    for (int i = 0; i < str.length(); i += 2){
        str.insert(i,'#');
    }
    str.append("#");

    //分别代表最右端、当到达最右端是的中心位置、最大长度、取得最大长度时的中心
    int right = 0, center = 0, maxlen = 0, index = 0; 
    int[] len = new int[str.length()];
    for (int i = 0; i < str.length(); i++) {
        if (right > i) len[i] = Math.min(len[2 * center - i], right - i);
        else len[i] = 1;
        while(i - len[i] >= 0 && i + len[i] < str.length() && str.charAt(i-len[i])== str.charAt(i+len[i])){
            len[i] ++;
        }
        if(i + len[i] > right){
            right = i + len[i];
            center = i;
        }
        index = len[i] > maxlen ? i : index;
        maxlen = Math.max(maxlen, len[i]);
    }
    return str.substring(index - maxlen + 1, index + maxlen).replace("#","");
}
```

