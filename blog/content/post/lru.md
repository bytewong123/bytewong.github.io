---
title: lru
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# lru
#算法

lru实现的要点：

1. 需要使用O(1)定位到要查找的元素
2. 每次get/put操作，都需要将元素插入到队首
3. 若put后，容量已满，需要将队尾元素移除
4. 若put一个相同key的已存在元素，需要将该元素先删除，再插入到队首
5. 每次get操作，若元素存在，需要将该元素先从链表中删除，再插入到队首
6. 删除节点操作时需要判断是否删除的是尾节点，若是的话需要更新尾节点为当前节点的前一个节点，但若前一个节点是头节点，则更新为nil



```go
type LRUCache struct {
    nodes map[int]*linkedNode
    head *linkedNode
    tail *linkedNode
    capacity int
}

type linkedNode struct {
    key int
    val int
    pre *linkedNode
    next *linkedNode
}

func Constructor(capacity int) LRUCache {
    nodes := make(map[int]*linkedNode)
    return LRUCache{
        nodes: nodes,
        head: new(linkedNode),
        capacity: capacity,
    }
}


func (this *LRUCache) Get(key int) int {

    if node, ok := this.nodes[key]; ok {
        //从链表中删除该元素，插入到头部
        this.deleteNode(node)
        this.insertToHead(node)
        
        return node.val
    } else {
        return -1
    }
}

func (this *LRUCache) Put(key int, value int)  {
    node := &linkedNode{key:key, val:value}
    //如果存在相同key，需要在链表中删除该节点，再重新插入到头部
    if node, ok := this.nodes[key]; ok {
        this.deleteNode(node)
    }
    //插入头部，并插入到集合中
    this.insertToHead(node)
    this.nodes[key] = node

    //达到容量限制，删除尾部元素，并从集合中删除
    if len(this.nodes) > this.capacity {
        this.dropOne()
    } 
}

func (this *LRUCache) dropOne() {
    delete(this.nodes, this.tail.key)
    oldTail := this.tail
    this.tail = this.tail.pre
    this.tail.next = nil
    oldTail.pre = nil
}

func (this *LRUCache)insertToHead(node *linkedNode) {
    n := this.head.next
    node.next = n
    node.pre = this.head
    if n == nil {
        this.tail = node
    }
    this.head.next = node
    if n != nil {
        n.pre = node
    }
}

func (this *LRUCache)deleteNode(node *linkedNode) {
        //删除节点时，需要操作尾部元素
        //若删除节点为队尾元素
        if node == this.tail {
            //如果删除的元素的前一个节点是头节点，则不用操作
            //否则需要将尾节点改为删除节点的前一个节点
            if node.pre != this.head {
                this.tail = node.pre
            } else {
				   this.tail = nil		
			  }
        }
        node.pre.next = node.next
        if node.next != nil {
            node.next.pre = node.pre
        }
        node.next = nil
        node.pre = nil
}

/**
 * Your LRUCache object will be instantiated and called as such:
 * obj := Constructor(capacity);
 * param_1 := obj.Get(key);
 * obj.Put(key,value);
 */
```

刷题记录：
2021-02-12