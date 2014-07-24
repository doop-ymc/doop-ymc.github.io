---
title: 网页版本对比系列二 —— Myers' Diff Algorithm 1
layout: post
category: 技术
tag:
    - 技术
    - 算法
---

本节将详细的对
[Myer's diff algorithm]( https://neil.fraser.name/software/diff_match_patch/myers.pdf )
的算法思想及过程进行详细的介绍。

###介绍

Myer's diff algorithm 首次出是在1986年一篇论文中"An O(ND) Difference Algorithm and Its Variations", 在文中实现上介绍了两种此diff算法
的实现。两种实现的核心思想是一致的，只是在具体的实现过程中，为进一步提升算法的性能及空间利用率，采取了不一致的迭代方式。下面先介绍，论文中
描述的基础算法。

在进行具体的算法描述前，先给出一些文中用到的定义及前提。
###定义

1. 最短编辑距离(SES)   
   最短编辑距离是指，在仅可进行`插入`，`删除`两种操作的情况下，将串A转换为串B所需要的操作次数。

2. 最长公共子序列(LCS)
   最长公共子序列是指，对于两个串，在保持顺序不变的前提下，能够得到的两串间的最大长度。与最长公共子
   串的不同处在于，最长公共子序列不要求是连续的。

3. 编辑图(Edit graph)
    编辑图实现是由两个串`A`,`B`分别为x, y坐标构成一个矩阵，不过对于其中相等的点处会多画出一条斜边。  
    如对于串`CBABAC`, `ABCABBA`, 其编辑图可表示为下图:
[![edit](/media/files/tech/myers_0.png)]()

4. Snakes
    对于Snakes的理解，存在一些歧义，我的理解是一条连续的由斜边组成的路径。如上图的红色路径示即为一条snake。

5. k斜线(k lines)
   k斜线是指对于由`x-y`具有相同的值的点组成一条直线，若`k=x-y`, 则称由这些点组成的线为k斜线。如下图示，为
   编辑图上的k lines。
[![edit](/media/files/tech/myers_1.png)]()

6. d等距线(d contours)  
  d contours 是指到起始点处，具有相同的编辑距离的点构成的线。如下图示：
[![edit](/media/files/tech/myers_2.png)]()

7. d-path 
 d path 是指从起始点开始, 假设为(0,0), 向前延伸的路径中包含d条非斜边的这样的一条路径。

8 d-path的递归定义
 一条d-path必定由一条(d-1)-path接一条水平或者垂直边，再加上一条snake构成。

###算法描述

基础算法是建立在以下两条定理的基础上的:

*   定理一：任何一条D-path都必定会终止于这些k lines, k ∈ { − D, − D + 2,  . . . D − 2, D  }.
*   定理二: 一条最远的在斜边k上结束的D-path，必定会由一条在k-1上结束的到达最远的(D-1)-path与一条水平边，
    再与一条能够到达的最远的snake构成，或者由一条在k+1上结束的到达(D-1)-path与一条垂直边，再与一条能够
    到达的最远的snake构成。

对于定理一证明过程如下：

    1. 当D=0时， 0-path就是对角线，满足条件
    2. 设i-path，满足定理， 则i-path的终点必定在这些k-lines 上， k ∈ { − Di − Di+ 2,  . . . Di− 2, Di }
       则根据D-path的递归定义，(i+1)-path必定是由i-path后面跟一条垂直边或者一条水平边，再加一snake构成，这样一
       条垂直边 会使(i+1)-path结束终止点的k-lines范围在{-i-1, i}, 一条水平边则将范围更改至{-i-1, i+1}, 而snake
       是不会 更改k-lines的范围的，因此(i+1)-path的终点所在的k-lines必定在{-i-1, i+1},定理得证。

对于定理二，实际可由d-path的递归定义推导得，这儿就不证明了。
经以个两个定理，实际，已经可以迭代的求出能够到达串的编辑图的终止在对角点的D最小的一条D-path,这条D-path实际
就描绘出了最小编辑距离，实际上也就给出两中最长公共子序列（LCS）。

下面说说具体的算法思路：
    
    通过上面的两条定理，就可以通过迭代，不断的求出各条k lines到达最远的D-path。 这样当有一条D-path到达了编辑图的
    右下角， 就找到了两串间的LCS,而最大的D可能为M+N, M, N分别为两串的长度，因为最多两串完全不一致。而由第一条定
    理知，D-path 必定 会终止于-D, D内的k lines上， 因此在计算D-path到达的最远距离时，只需要计算-D，到D的k lines所
    到达的最远的距离，而每次 计算D-path在k lines到达的最远距离时，是可以根据 (D-1)-path迭代来的，为此，通过迭代计
    算可以最终求出能最终到达编辑图 右下角的D-path,且拥有最小的D,即具有最小的编辑距离。

下面是简单迭代过程：

    Constant MAX ∈ [0,M+N]

    Var V: Array [− MAX .. MAX] of Integer

    V[1] ← 0
    For D ← 0 to MAX Do
        For k ← −D to D in steps of 2 Do
            If k = −D or k ≠ D and V[k − 1] < V[k + 1] Then
                x ← V[k + 1]
            Else
                x ← V[k − 1]+1
            y ← x − k
            While x < N and y < M and ax+1 = by+1 Do (x,y) ← (x+1,y+1)
            V[k] ← x
            If x ≥ N and y ≥ M Then
                Length of an SES is D
                Stop
    Length of an SES is greater than MAX

下面是具体的迭代结果图：
[![edit](/media/files/tech/myers_3.png)]()
上图中，蓝色线条为具体的迭代过程，红色为最终求得的D-path.根据这条D-path实现已可求出LCS.不过其时间与空间复杂度均是 O( (N+M) D  ) .
下节将详细介绍论文中提到的O(ND)算法。 











