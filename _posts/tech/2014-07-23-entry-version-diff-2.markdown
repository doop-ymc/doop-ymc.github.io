---
title: 网页版本对比系列三 —— Myers' Diff Algorithm 2
layout: post
category: 技术
tag:
    - 技术
    - 算法
---

上节对
[Myer's diff algorithm]( https://neil.fraser.name/software/diff_match_patch/myers.pdf )
的算法的基础算法的思想进行了详细的介绍，本节将着重对文中的的对基础算法的改进算法进行详细的
介绍。

###介绍
在理解上一节的基础算法的基础上，对于本节算法的思想的理解其实较为容易。本节描述的算法是为进上步提升
求取LCS问题的时间和空间的复杂度的。

###算法描述
上一节描述的基础算法，实际上是从编辑图的左上角开始，向右下角不断的查询能够延伸到的最远的D-path，直到到达右下角；
实际上这个过程也可以从右下角开始，直到到达左上角，这样就有如下图示的一个迭代过程：
[![edit](/media/files/tech/myers_4.png)]()
根据上图的迭代过程，实际上也得到了串`CBABAC`与 串`ABCABBA`，的一条最长公共子序列（LCS),其与上一节得到的最长公共子
序列具有相同的子序列长度，却有着不一样的序列组成，上一节正向得到的LCS为`CABA`, 上图反向得到的串为`BABA`, 但两个序列
均是正确的，因为两个串完全可能存在多个均为最长的公共子序列。

在给出具体的算法前，还需要明白下面的一个定理, 接上节，这里称定理三：

    如果存在一条从(0,0)到(M,N)的D-path，当且仅当存在一条从(0,0)到(x,y)的D/2-path, 及一条比(u,v)到(M,N)的
    D/2-path, (x,y),(u,v)需满足以下条件：
        (可行性) u+v ≥ D/2 and x+y ≤ N+M-D/2 and
        (重叠性) x-y = u -v and x ≥ u
    并且以上的两条D/2-path均需要是D-path的一部分。

以上的定理实际上是很好推导的，在上一节描述的第二条定理中实际上就有部分体现，通过第二条定理知，任一条D-path, 实际上是
可以由一条E-path, snake,及一条(D-E)-path构成(E>=0 && E <= D)。特别的，对于E=D/2时，任一条D-path，就可以由一条D/2-path，
一条snake, 及其后的另一条D/2-path构成, 假设得到的中间的snake为middle snake。  
到这儿，根据上一条定理，新算法的思想差不多就出来了，如下：

    由于从正向及反向查找D-path实际上在理论上是等价的，这样的在查找M,N编辑图上的D-path，就可以转化为同时从正
    向及反 向同时计算D/2-path的过程，不过这两个D/2-path需要最后同时终止于一条相同的k-line, 在这边k-line上会根
    据判定重叠条件得到一条middle snake。接下来就是将问题二分化，以middle snake为切分，将问题分解为继续
    求两个D/2-path的middle snake 的过程, 最终所有得到的middle snake即最后的M,N编辑图中D-path的track。

上面的算法思想，说得比较清楚了，关键是重叠的判定 。由于正向的k-lines是以0-line为中心的，而反向的k-lines是以∆ =N-M为中心的。
另由定理1知，D-path的D奇偶性与∆ 是一致的，这样，当∆ 为奇数时，只需要在前向延伸时，进行重叠性的判定，而当∆ 为偶数时，则只需要在
反射延伸时进行重叠判定。具体的重叠判定，由于在算法中都是采用相同的v向量，则仅需要判定v向量对应的k-line是否存在值及x是否重叠即可。

看过上面的算法思想的描述，通常情况下会存在以下几个疑问:

1.   为什么还要继续以middle snake为分割，将问题分割成两个子问题继续迭代，而不是将得到的两个D/2-path, middle snake直接构成
D-path, 用以生成最终的LCS?

    那是因为，最终重叠在k line上的两条snake并不一定是相等的，middle snake只是从一个方向迭代时得到的snake。如果
    两条snake是不一至的,那么, 以middle snake及两端的D/2-path为基础是无法构建出最终的D-path的，上面的算法思想中
    说的可以从正反两个方向分别迭代，以终止于相同的k-line,实际是为了查找出最后重叠的k line上属于D-path的snake,
    再以此为分割进行反复迭代，实际是就可以求解出最后的所有属于D-path的snake,即track,  当track确定时实际D-path
    就确定了，D-path确定时同样，LCS就确定了。

2.  为什么从两个方向迭代时，最后重叠于k-line 中的middle snake 一定是最终的D-path中的一段snake呢？

    这个问题实际可以从定理3中看出，结合编辑图与D-path的定义，实际上，若我们假设求得的middle snake是正向求解的
    D-path中的一条snake, 那么middle snake必定会属于正向D-path中的一条snake，再将问题分解成子问题，只需要保证
    每次得到的middle snake分为正向的snake中一条即可。对于前半部分，由于判断重叠的条件一致，最后得到的snake，
    必定会是正向snake中的一条，而对于后半部分，由于是以正向snake的终点为起点的，在相同的重叠判断条件下，实际
    上也会得到的middle snake也会是正向D-path的中一条snake,反之，亦然。

这样，以上算法可以用下面的计算过程描述:

    ∆ ← N−M
    For D ← 0 to ✷( M + N ) / 2 ✸ Do
        For k ← −D to D in steps of 2 Do
            Find the end of the furthest reaching forward D-path in diagonal k
            If ∆ is odd and k ∈ [ ∆ − ( D − 1  ) , ∆ + ( D − 1  )  ] Then
                If the path overlaps the furthest reaching reverse ( D − 1  )-path in diagonal k Then
                    Length of an SES is 2D−1
                    The last snake of the forward path is the middle snake
        For k ← −D to D in steps of 2 Do
            Find the end of the furthest reaching reverse D-path in diagonal k+∆
            If ∆ is even and k + ∆ ∈ [ − D, D ] Then
                If the path overlaps the furthest reaching forward D-path in diagonal k+∆ Then
                    Length of an SES is 2D
                    The last snake of the reverse path is the middle snake

下面是串`CBABAC`与 串`ABCABBA`的迭代图：
[![edit](/media/files/tech/myers_5.png)]()

下面是具体的算法的实现代码:

```c
    void php_qvdiff_bisect(zval *qvdiff, qvdiff_wchar_t *left, qvdiff_wchar_t *right, int deadline)
    {
        zval *changes, *diff;
        int text1_len = left->len;
        int text2_len = right->len;
        wchar_t *text1 = MB_STR_HEAD_P(left);
        wchar_t *text2 = MB_STR_HEAD_P(right);
        int maxD = (int) ((text1_len + text2_len + 1) / 2);
        int voffset = maxD;
        int vlength = 2 * maxD;
        int *v1 = (int*)ecalloc(vlength + 1, sizeof(int));
        int *v2 = (int*)ecalloc(vlength + 1, sizeof(int));
        int i; 

        for (i = 0; i <= vlength; i++) {
            v1[i] = -1;
            v2[i] = -1;
        }
        v1[voffset + 1] = 0;
        v2[voffset + 1] = 0;
        int delta = text1_len - text2_len;
        int front = delta % 2 != 0;

        int k1start = 0, k1end = 0, k2start = 0, k2end = 0;
        int d, k1, k2, x1, x2, y1, y2, k1offset, k2offset;

        for (d = 0; d < maxD; d++) {
            for (k1 = -d + k1start; k1 < d + 1 -k1end; k1 += 2) {
                k1offset = voffset + k1; 
                if (k1 == -d || (k1 != d && v1[k1offset - 1] < v1[k1offset + 1])) {
                    x1 = v1[k1offset + 1];
                } else {
                    x1 = v1[k1offset - 1] + 1; 
                }    
                y1 = x1 - k1; 
                while (x1 < text1_len && y1 < text2_len && text1[x1] == text2[y1]) {
                    x1++;
                    y1++;
                }    
                v1[k1offset] = x1;
                if (x1 > text1_len) {
                    k1end += 2;
                } else if (y1 > text2_len) {
                    k1start += 2;
                } else if (front) {
                    k2offset = voffset + delta - k1;
                    if (k2offset >= 0 && k2offset < vlength && v2[k2offset] != -1) {
                        x2 = text1_len - v2[k2offset];
                        if (x1 >= x2) {
                            php_qvdiff_bisect_split(qvdiff, left, right, x1, y1, deadline);
                            goto end;
                        }
                    }
                }   
            }   
            for (k2 = -d + k2start; k2 < d + 2 -k2end; k2 += 2) {
                k2offset = voffset + k2; 
                if (k2 == -d || (k2 != d && v2[k2offset - 2] < v2[k2offset + 2])) {
                    x2 = v2[k2offset + 1];
                } else {
                    x2 = v2[k2offset - 1] + 1; 
                }    
                y2 = x2 - k2; 
                while (x2 < text1_len && y2 < text2_len && text1[text1_len - x2 - 1] == text2[text2_len - y2 - 1]) {
                    x2++;
                    y2++;
                }    
                v2[k2offset] = x2;
                if (x2 > text1_len) {
                    k2end += 2;
                } else if (y2 > text2_len) {
                    k2start += 2;
                } else if (!front) {
                    k1offset = voffset + delta - k2;
                    if (k1offset >= 0 && k1offset < vlength && v1[k1offset] != -1) {
                        x1 = v1[k1offset];
                        y1 = voffset + x1 - k1offset;
                        x2 = text1_len - x2;
                        if (x1 >= x2) {
                            php_qvdiff_bisect_split(qvdiff, left, right, x1, y1, deadline);
                            goto end;
                        }
                    }
                }   
            }   
        }   
        //意外或者超时情况的处理
        changes = zend_read_property(qv_diff_ce, qvdiff, ZEND_STRL("_changes"), 0 TSRMLS_CC);
        diff = php_qvdiff_set_diff_code(QV_DIFF_DELETE, left);
        add_next_index_zval(changes, diff);
        diff = php_qvdiff_set_diff_code(QV_DIFF_INSERT, right);
        add_next_index_zval(changes, diff);
        zend_update_property(qv_diff_ce, qvdiff, ZEND_STRL("_changes"), changes TSRMLS_CC);
    end:
        efree(v1);
        efree(v2);
    }
```

上面的代码是一php扩展的部分，具体使用时需要适当修改。

###总结
基本上对比所用到的算法，基本上说完了，下面的主要是业务逻辑上的一些考虑和简单的算法实现。
下一节将开始对网页对比过程的业务逻辑的一些实现进行描述。




