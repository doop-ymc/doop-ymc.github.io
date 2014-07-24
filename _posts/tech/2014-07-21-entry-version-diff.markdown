---
title: 网页版本对比系列一 —— diff 基础算法
layout: post
category: 技术
tag:
    - 技术
    - 算法
---

由于各种需求，需要对两个网页的内容进行一些关注度元素的对比，如百度百科的历史版本对比功能。
对于这种两个网页的对比，不可以简单的看作两文本的对比。因为，网页中的一些标签实际上可能是等价的，
也可能是无关的（及对具体的对比结果不会产生影响）。但是，网页的对比是可以基于文本的对比的，其对比
结果是可以建立在纯文本对比的基础上的。  

对于纯文本的对比，基础上来看就是求两字符串的最长公共子序列问题（LCS)，而对于这类问题，常规的解法是
通过动态规划来求解（简单的暴力计算，这儿就不讨论了）。下面对动态规划求解LCS问题进行具体讨论：

###方法一、常规的动态规划方法

动态规划求解最长公共子序列问题，首先需要构建出最基本的子问题，方才能迭代出最优解。  
最长公共子序列有如下的描述：
    
    设序列X=<x1, x2, …, xm>和Y=<y1, y2, …, yn>的一个最长公共子序列Z=<z1, z2, …, zk>，则：
        若xm=yn，则zk=xm=yn且Zk-1是Xm-1和Yn-1的最长公共子序列；
        若xm≠yn且zk≠xm ，则Z是Xm-1和Y的最长公共子序列；
        若xm≠yn且zk≠yn ，则Z是X和Yn-1的最长公共子序列。
    其中Xm-1=<x1, x2, …, xm-1>，Yn-1=<y1, y2, …, yn-1>，Zk-1=<z1, z2, …, zk-1>。

其递归结构可用下式表示：
[![lcs](/media/files/tech/lcs.png)]()
根据上式，则对于串`ABCBDAB`与串`BDCABA` 的LCS求解过程可如下图示：
[![lcs](/media/files/tech/lcscompute.png)]()
下面是LCS求解过程：

    Procedure LCS_LENGTH(X,Y);  
        begin  
            m:=length[X];  
            n:=length[Y];  
            for i:=1 to m do c[i,0]:=0;  
            for j:=1 to n do c[0,j]:=0;  
            for i:=1 to m do  
            for j:=1 to n do  
            if x[i]=y[j] then  
                begin  
                    c[i,j]:=c[i-1,j-1]+1;  
                    b[i,j]:="↖";  
                end  
            else if c[i-1,j]≥c[i,j-1] then  
                begin  
                    c[i,j]:=c[i-1,j];  
                    b[i,j]:="↑";  
                end  
            else  
                begin  
                    c[i,j]:=c[i,j-1];  
                    b[i,j]:="←"  
                end;  
            return(c,b);  
        end;

下面为构造LCS的过程:
    
    Procedure LCS(b,X,i,j);  
        begin  
        if i=0 or j=0 then return;  
        if b[i,j]="↖" then  
            begin  
                LCS(b,X,i-1,j-1);  
                print(x[i]); {打印x[i]}  
            end  
        else if b[i,j]="↑" then LCS(b,X,i-1,j)   
        else LCS(b,X,i,j-1);  
        end; 

###方法二、转化为求最长递增子序列
个人认为，求最长公共子序列问题，是可以转化为求最长递增子序列的。转化的好处是，求最长递增子序列的过程是可以将时间复杂度与空间复杂度分别降低到
O(nlgn)和O(n+m)的。我们先不验证是否转化后求出的子序列是否为最长公共子序列的解之一，我们先来看如何完成最长公共子序列问题向最长递增子序列问题的转化。
同样，还是设两序列分别为X=<x1, x2, …, xm>和Y=<y1, y2, …, yn>:

* 先对序列Y进行预处理。使得对C[yi]={a,b,c...i} 集合为yi出现于Y中的行号。且有a>b>c>...>i
* 再依次检查X,若X中的元素出现在了Y中，则将上面的C对应值的赋给对给X[i],并记录为此值对应的在X中的行号，以方便构建出最长公共子序列

下面给出php版本的求最长公共子序列的过程：

```php
    /** 
     * 查找子序列的相似解，可能存在误差
     * @param array $left 
     * @param array $right 
     * @param string $l_begin 左侧的开始索引位置
     * @param string $r_begin 右侧的开始索引位置
     *
     * #return array
     */
    private function _getSimilarLCSArray($left, $right, $l_begin=0, $r_begin=0) {
        $l_length = count($left);
        $r_length = count($right);

        $t_res = $this->_getMatchBlocksArray($left, $right);
        $t_array = $t_res[0];
        $c_array = $t_res[1];
        //获取最长递增子序列
        $b_array[0] = -1;
        $b_len = 1;
        foreach ($t_array as $key => $value){
            if ($value > $b_array[$b_len - 1]) {
                $b_array[$b_len++] = $value;
            } else {
                $l = 1;
                $r = $b_len;
                while ($l < $r) {
                    $m = ($l + $r) / 2;
                    if ($b_array[$m] == $value) {
                        break;
                    }
                    if ($b_array[$m] > $value)
                        $r = $m - 1;
                    else
                        $l = $m + 1;
                }
                $b_array[$m] = $value;
            }
        }
        //生成差异码
        $lcs_len = count($t_array);
        $end = $b_array[$b_len - 1];
        $b_end = true;
        $b_array[$b_len] = $end + 1;
        $res = array();
        $res[] = array($l_length + $l_begin , $r_length + $r_begin , 0);
        for ($i = $lcs_len - 1; $i >= 0; $i--) {
            if ($t_array[$i] >= $end ) {
                if (!($b_end && $t_array[$i] == $end))
                    continue;
                else
                    $b_end = false;
            }
            if($t_array[$i] >= $b_array[$b_len - 2] ){
                array_unshift($res, array($c_array[$i] + $l_begin , $t_array[$i] + $r_begin , 1));
                $end = $t_array[$i];
                $b_len--;
                if ($b_len < 2)
                    break;
            }
        }
        return $res;
    }
```

其实对于以上两种方法都是利用动态规划的思想来求解，对于转换为最长递增子序列后，再进行求解LCS实际上在性能上的优化不是很大，
在空间上的优化却是要分情况的，对于两串的相似度较低的情况则空间的消耗可以是渐近线性的，而对于两串相似度非常高时，则空间的
消耗瞬间下降至n*n, 实际上作用不大,不过在实际项目的使用中，方法2通常情况下优于方法1。

不过，在最后的实际项目中，由于需要对性能及空间利用上，有更高的要求，因此，以上两种方法均未采用，还是利用了google-diff的思想，同样对论文
[Myer's diff algorithm]( https://neil.fraser.name/software/diff_match_patch/myers.pdf )
进行实现。下节，将详细对Myers' Diff Algorithm进行详细的介绍。


