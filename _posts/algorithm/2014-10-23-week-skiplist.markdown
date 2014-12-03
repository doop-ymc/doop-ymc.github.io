---
title: 一周一算法—— skiplist
layout: post
category: 技术
tag:
  - 技术
  - 算法
  - 数据结构
---

Skip list又称跳跃表，如果对于每一个2^i结点均有一个前向的2^i的指针，那么在构造上，实际上个人认为其与平衡二叉树是有一定的
共性的。但是平衡二叉树对于数据的插入顺序是有较高的要求的，否则其平衡性会受到很大的程度的影响，进而导致树的查找，删除
等基本操作的性能急剧退化，甚至在极端情况下退化为list。而Skip list相较于平衡二叉树，理论上其也是可以将性能控制到O(logn)
。而Skip list相较于平衡二叉树其在实现上，及大量并发情况下访问的性能是有很大优势的。

### skip list的介绍

通常在skip list实现时，不是实现严格意义的skip list，其前向指针并不是完整构建的，层级也不会为logn，而是如下的描述，通过概率性的
确定层级来对其进行优化。

    Skip lists  are data structures  that use probabilistic  balancing rather  than  strictly  enforced balancing. As a result, 
    the algorithms  for insertion  and deletion in skip lists  are much simpler and significantly  faster  than  equivalent  
    algorithms  for balanced trees.  

如上的描述，这样的方式决定层级能够其在插入删除时较平衡二叉树更为简单及快速。下图中的d示，为一个完整构造的skip list, 而通常真正实
现及讨论的图e示的概率构建level的skip list, 其相较于a, b, c有着更高的查找性能，而相较于d,又能有较好的层数，以降低插入删除的复杂性
（完整的skip list在插入，删除时需要考虑链表各层list的重新构建而概率性决定层数的skip list不需要，只需要将对应的结点删除即可，因
为不需要维护完整意义的层级指针)
[![skip](/media/files/tech/skip.png)]()

对于上图a, b, c, d, e每次查找, 其分析如下：  
a. level为1，无其它前向结点加速跳跃，需要便利n个结点。 
b. 每两个结点建立了一个前向指针，在第二层查找时，可以以步长2的速度向后跟踪，为此最多需要遍历n/2+1个结点。  
c. 同上，在遍历时最多遍历n/4+2个结点。    
d. 由于对每2^i个结点均建立前向的指向2^i的结点，因此其类似于十分查找，有logn的查找速度。     
e. 查找性能会略低于d, 但是其由于在查找删除时不需要如d维护前向指针，实际性能应该不会下降太多，却提升了较大的实现简单性。

### skip list算法

#### 初始化
将层级初始化为1，建立key为最大值的哨兵结点。

### 查找

[![skip](/media/files/tech/skip_search.png)]()

[![skip](/media/files/tech/skip_id.png)]()

[![skip](/media/files/tech/skip_level.png)]()
