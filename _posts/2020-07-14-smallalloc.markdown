---
layout:     post
title:      "TCMalloc中小对象分配流程"
subtitle:   "小对象分配"
date:       2020-07-17 21:21:00
author:     "Jfchen"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - tcmalloc
---

### 对象大小定义
tcmalloc中按照用户申请内存的大小，把申请空闲内存对象分为小对象，中对象和大对象。
小于256k的为小对象，256k到1M是中对象，大于1M为大对象。

### 小对象分配流程
- 根据实际申请的内存大小，查看ThreadCache对象中的list_链表中查找是否有可用对象，如果有可用对象，则从对应的FreeList链表中取走第一个空闲内存块返回给用户。而需要从哪一个FreeList链表中取，则需要配合sizemap()来操作，这一步完全没有加解锁的操作，也是tcmalloc效率较高的一个重要原因。
    - list_是一个FreeList数组，而每个FreeList是一个链表
    - 每个FreeList是固定大小的空闲内存的集合，比如按照8
    byte组成链表，或者按照16btye组成链表
- 如果上一步没有找到可用内存，此时会从TransferCache中查找是否有可用内存
    - slots_是空闲内存的链表，类似FreeList的结构，TransferCache会优先从该结构中查找是否有可用内存，如果有，则批量将一部分内存转移到FreeList中（其中第一块内存返回给用户，剩余的才链接到FreeList中），至于批量转移多少内存，转移到哪个FreeList链表中，则需要配合sizemap()来操作。
- 如果上一步仍然没有找到合适的内存块，则开始从CentralFreeList中查询是否有可用内存，CentralFreeList中管理的对象是SpanList(span组成的链表，span是连续page集合，这里的page不同于os中的page，tcmalloc中的page固定大小为4k或者8k，将虚拟内存按照4k或者8k进行切分，并进行编号pageid，在实际内存分配过程中，需要不断将pageid和内存地址进行转换),
    - nonempty_是一个Spanlist，先查看该链表是否为空，是的话调用CentralFreeList::Populate()创建新的链表，并按照sizemap()将span切割成大小一致的内存块
    - 查询nonempty_是否有合适的span，有的话则填充FreeList
    - 如果填充完FreeList之后，span中可用内存为0，则从SpanList中移除