- HashMap的数据结构
1.7 数组加链表
哈希冲突 
1.8 数组加链表加红黑树

为什么长度是2^n，为什么是偶数

- HashMap的实现原理
- 一些重要参数
`DEFAULT_INITIAL_CAPACITY  = 1 << 4`初始容量
必须是2的n次方，为什么
2的n次方可以用与运算代替取模运算
(n-1)&hash = hash % n

`MAXIMUM_CAPACITY = 1 << 30`

`DEFAULT_LOAD_FACTOR = 0.75f`负载因子，当数据量与容量的比例大于负载因子就扩容。

`TREEIFY_THRESHOLD = 8`什么时候链表转红黑树
`UNTREEIFY_THRESHOLD = 6`

`MIN_TREEIFY_CAPACITY = 16`
- HashMap源码解析

保证散列尽量均匀，也就是让每个位尽可能的参与运算。于是计算出来的哈希值高位右移16位然后与低位异或运算--扰动函数。

- 问题
    1. 实现的数据结构，为什么
    2.  数组的默认大小？ 什么时候扩容？
    3.  如何计算哈希值 