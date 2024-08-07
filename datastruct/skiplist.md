### redis zset是如何实现的？

redis 的zset由skiplist和hash共同组成
范围查找用的skiplist，比如

* zrange （升序）
* zrevrange（降序）

#### 源代码位于

<https://github.com/redis/redis/blob/unstable/src/t_zset.c>

#### skiplist

* golang 翻译的实现
<https://github.com/antlabs/gstl/blob/master/skiplist/skiplist.go>

* <https://zhuanlan.zhihu.com/p/532488152>

* skiplist从结构上来看，有点像多层链表。最下面的链表包含全部元素，第二层包含一半元素，第三层包含第二层一半的元素，以此类推。。。 所以搜索性能近似一个二叉树
数值越大越接近这值，想下古典概率抛硬币的实验。

* skiplist 通过抛硬盘决定，建立的节点层次

```go
func (s *SkipList[K, T]) rand() int {
 level := 1
 for {

  if s.r.Int()%2 == 0 {
   break
  }
  level++
 }

 if level < SKIPLIST_MAXLEVEL {
  return level
 }

 return SKIPLIST_MAXLEVEL
}
```

* skiplist 结构结点

```go
type Node[K constraints.Ordered, T any] struct {
 score K
 elem  T
 // 后退指针
 backward  *Node[K, T]
 NodeLevel []struct {
  // 指向前进节点, 是指向tail的方向
  forward *Node[K, T]
  span    int
 }
}
```

* 插入过程
  * 先根据score找到插入位置，插入节点. 每一层的指针都保存到update数组中
  * 抛硬币决定插入节点的层高
  * 修改前后指针
