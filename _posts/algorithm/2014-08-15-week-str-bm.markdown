---
title: 一周一算法—— BM算法
layout: post
category: 技术
tag:
  - 技术
  - 算法
---
  
此次接上次的Zblock算法，讨论字符串精确匹配的BM算法. 
BM算法是实际应用中字符串匹配效率比较高的算法，其的应用效果是优于KMP算法的。具其他人测试应该是
2-5倍，不过KMP算法在理论上设计是较为优雅的，性能较BM算法在最坏情况下，是更优的。

###BM算法介绍

BM算法也如KMP一样，相较于暴力查找，其在出现不匹配时，使用了不同的策略来完成模式串的移动，以使得能够尽可能多的
跳过无用的匹配，有提升算法搜索的性能。

BM算法在进行不匹配时的模式串移动时，有两个跳转表：坏字符表、好后辍表。这两个跳转表都是需要通过预处理后得到的。
先看一组BM算法进行字符串匹配时跳转的例子。若模式串为"AT-THAT"，目标串为"WHICH-FINALLY-HALTS.--AT-THAT-POINT"。
则匹配过程可如下示：
<pre>
   1 2 3 4 5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35
   W H I C H  -  F  I  N  A  L  L  Y  -  H  A  L  T  S  .  -  -  A  T  -  T  H  A  T  -  P  O  I  N  T
 1 A T - T H  A  T                                                        
 2                  A  T  -  T  H  A  T                                          
 3                              A  T  -  T  H  A  T                                  
 4                                                A  T  -  T  H  A  T                      
 5                                                               A  T  -  T  H  A  T            
</pre>
实际是在上面的前四组匹配中都是利用的坏字符的跳转规则，最后一组则是利用了好后辍的跳转方法。下面的我们具体分析一下：
*   坏字符规则：坏字符是指在模式串与目的串进行匹配过程中，出现不匹配时，目标串中的字符。依据坏字符的跳转规则需要有
    两种情况。
    （1）坏字符不出现在模式串中时：当坏字符不出现在模式串中时，说明模式串中不可能存在与坏字符相匹配的字符出现，也就不可能出
        现存在包含此字符的一个模式串到目标串的匹配，那么在模式串与目标串进行匹配对比时，就可以完全跳过包含此字符的任意的对比
        过程。因此模式可以直接移动至此字符后，再进行匹配。即向后移动一个模式串的长度。如上例中的第1、2、3两种移动都属于此情况。
        也可见下图的匹配情况。
    [![bad_char_not](/media/files/tech/bad_char_in_pattern0.png)]()
    （2）坏字符出现在模式串中时：当坏字符出现在模式串中时，此时需要将坏字符与模式串中此字符出现的最靠右的字符相对（也有说与
        模式串的未匹配部分最靠右的字符相对，这样不会有回退），如下图所示的移动。
    [![bad_char_in](/media/files/tech/bad_char_not_in_pattern0.png)]()
    下面的图是为了更好的理解以上的两种情况：
    [![bad_char](/media/files/tech/bmbc-funtion0.png)]()

*   好后辍规则：好后辍是指模式串与目标串匹配过程中，不匹配出现时，模式串与目标串已经匹配的部分。好后辍规则分为以下三种情况：
    （1）模式串中存在好后辍构成的子串，这种情况只需要将模式串中最靠左的子串与好后辍对齐即可，如下图示：
    [![good](/media/files/tech/good_suffix_in_pattern0.png)]()
    （2）模式串不存在情况（1）中的子串，但是存在与好后辍匹配的前辍，此时只需要将好后辍的后辍与模式串中对应的最长前辍匹配即可，如下图示：
    [![good](/media/files/tech/good_suffix_not_pattern0.png)]()
    （3）以上两种情况都不存在时，则需要将模式串整个移动到目标串中不匹配位置之后即可。如下图示：
    [![good](/media/files/tech/good_suffix_not_prefix0.png)]()

实际是对于坏字符规则与好后辍规则的一些情况是有一定共通性的。如坏字符规则的情况1与好后辍规则的情况3，坏字符规则的情况2与好后辍规则的情况情况
1（将整个好后辍看着一个整体，这里也可以看出在一定程度上坏字符规则中情况1的移动方式存在一定争议的）。不过对于整个的BM算法来说，可能坏字符情况1
的移动原则可能不是需要总是最佳的，因为BM算法的移动时选择的是**坏字符规则与好后辍规则中移动时，移动最大距离的方案**。

###BM算法的模式串的预处理

BM算法模式串的预处理分两个步骤：一个针对坏字符规则的，另一个是针对好后辍规则的。

*   对于坏字符规则，跳转表的建立与模式串的具体长度关系不大，其更多的受字符集大小的影响。坏字符规则跳转表的建立，即建立一个模式串中各字符的其最右位置至
模式串结尾的一个hash映射。可用如下的伪码进行表示：

    B[1...n] <= n;
    For i, From 1 to n ;
        B[pattern[i]] = n - 1 -i;
    End

*   对于好后辍规则的预处理，则相对麻烦得多。上一次讨论Zblock算法时，说过其处理思想是可以利用于KMP与BM算法的模式串的预处理中的。而Zblock算法是用于求前辍的
而BM算法则是利用的模式串的后辍信息，这样就需要一些转化，才能开始计算。假设模式串为S="abcxxxabc"，可以先对其进行反转操作得到Sr="cbaxxxcba"。这样由上一次讨
论的Zblock算法可得到一个Sr的Zblock表如下示：
<pre>
            1   2   3   4   5   6   7   8   9
    Sr      c   b   a   x   x   x   c   b   a
    Zi(Sr)  0   0   0   0   0   0   3   0   0
</pre>
那么对于好后辍的情况1中，即在上表中找到大于等于好后辍的长度的位置，这些位置开始的模式串必定会存在一个为好后辍的子串。而对于情况2，也就是好后辍在模式串中不存在
与之相匹配的子串，这种情况就只能打最长前辍了。此时则需要能够在上表的后n个元素找到一个Zblock能够覆盖最后一个元素的位置，且此Zblock拥有符合条件中所有Zblock中的
最大长度，那么此Zblock即为最长前辍。
根据上面的思想，基本是可以将Zblock表转换为好后辍规则的跳转表的。

###BM算法的具体实现

下面的代码引用自其它地方，最后会给出引用位置：

```c
    #include <stdio.h>
    #include <string.h>
         
    #define MAX_CHAR 256
    #define SIZE 256
    #define MAX(x, y) (x) > (y) ? (x) : (y)
     
    void BoyerMoore(char *pattern, int m, char *text, int n);
     
    int main()
    {
        char text[256], pattern[256];
     
        while(1)
        {
            scanf("%s%s", text, pattern);
            if(text == 0 || pattern == 0) break;
     
            BoyerMoore(pattern, strlen(pattern), text, strlen(text));
            printf("\n");
        }
     
        return 0;
    }
     
    void print(int *array, int n, char *arrayName)
    {
        int i;
        printf("%s: ", arrayName);
        for(i = 0; i < n; i++)
        {
            printf("%d ", array[i]);
        }
        printf("\n");
    }
     
    void PreBmBc(char *pattern, int m, int bmBc[])
    {
        int i;
     
        for(i = 0; i < MAX_CHAR; i++)
        {
            bmBc[i] = m;
        }
     
        for(i = 0; i < m - 1; i++)
        {
            bmBc[pattern[i]] = m - 1 - i;
        }
     
    /*  printf("bmBc[]: ");
        for(i = 0; i < m; i++)
        {
            printf("%d ", bmBc[pattern[i]]);
        }
        printf("\n"); */
    }
     
    void suffix_old(char *pattern, int m, int suff[])
    {
        int i, j;
     
        suff[m - 1] = m;
     
        for(i = m - 2; i >= 0; i--)
        {
            j = i;
            while(j >= 0 && pattern[j] == pattern[m - 1 - i + j]) j--;
     
            suff[i] = i - j;
        }
    }
     
    void suffix(char *pattern, int m, int suff[]) {
       int f, g, i;
     
       suff[m - 1] = m;
       g = m - 1;
       for (i = m - 2; i >= 0; --i) {
          if (i > g && suff[i + m - 1 - f] < i - g)
             suff[i] = suff[i + m - 1 - f];
          else {
             if (i < g)
                g = i;
             f = i;
             while (g >= 0 && pattern[g] == pattern[g + m - 1 - f])
                --g;
             suff[i] = f - g;
          }
       }
     
    //   print(suff, m, "suff[]");
    }
     
    void PreBmGs(char *pattern, int m, int bmGs[])
    {
        int i, j;
        int suff[SIZE];  
     
        // 计算后缀数组
        suffix(pattern, m, suff);
     
        // 先全部赋值为m，包含Case3
        for(i = 0; i < m; i++)
        {
            bmGs[i] = m;
        }
     
        // Case2
        j = 0;
        for(i = m - 1; i >= 0; i--)
        {
            if(suff[i] == i + 1)
            {
                for(; j < m - 1 - i; j++)
                {
                    if(bmGs[j] == m)
                        bmGs[j] = m - 1 - i;
                }
            }
        }
     
        // Case1
        for(i = 0; i <= m - 2; i++)
        {
            bmGs[m - 1 - suff[i]] = m - 1 - i;
        }
     
    //  print(bmGs, m, "bmGs[]");
    }
     
    void BoyerMoore(char *pattern, int m, char *text, int n)
    {
        int i, j, bmBc[MAX_CHAR], bmGs[SIZE];
     
        // Preprocessing
        PreBmBc(pattern, m, bmBc);
        PreBmGs(pattern, m, bmGs);
     
        // Searching
        j = 0;
        while(j <= n - m)
        {
            for(i = m - 1; i >= 0 && pattern[i] == text[i + j]; i--);
            if(i < 0)
            {
                printf("Find it, the position is %d\n", j);
                j += bmGs[0];
                return;
            }
            else
            {
                j += MAX(bmBc[text[i + j]] - m + 1 + i, bmGs[i]);
            }
        }
     
        printf("No find.\n");
    }
```

###总结
最后感觉其中还是有很多说得模糊的地方，因为个人理解的原因。此外个人感觉BM算法还是有一些局限性，因为其一定程度受
字符集的大小的影响。

最后本文参考了以下的一些对BM算法的介绍：  
[grep之字符串搜索算法Boyer-Moore由浅入深（比KMP快3-5倍）](http://blog.jobbole.com/52830/)   
[再谈KMP/BM算法（II）](http://blog.csdn.net/joylnwang/article/details/6880240)   
[字符串匹配那些事（一）](http://www.searchtb.com/2011/07/%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8C%B9%E9%85%8D%E9%82%A3%E4%BA%9B%E4%BA%8B%EF%BC%88%E4%B8%80%EF%BC%89.html)   











