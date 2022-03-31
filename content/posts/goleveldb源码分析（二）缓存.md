---
title: "Goleveldb源码分析（二）缓存"
date: 2022-03-31T08:30:43+08:00
categories: [goleveldb]
tags: [go, 读点源码]
draft: false
---

 # 缓存

缓存主要的功能就是减少磁盘IO。Leveldb中使用了一种基于LRUCache的缓存机制，用于缓存：   

- 已打开的sstable文件对象和相关元数据；   
- sstable中的dataBlock的内容。   

 ## 整体结构

Leveldb中使用的Cache是一种LRUcache，其结构由两部分内容组成：   

- Hash table：用来存储数据；   
- LRU：用来维护数据项的新旧信息。
    ![image.png](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/v_image.png)    

缓存系统：   

```go
// 缓存
type Cache struct {
	mu     sync.RWMutex   // 读写锁
	mHead  unsafe.Pointer // *mNode 指向哈希表
	nodes  int32          // 哈希表中的节点个数
	size   int32          // 哈希表大小
	cacher Cacher         // 缓存策略
	closed bool           // 是否关闭
}

```

缓存策略：   

```go
// 缓存策略接口，并发安全
// 以下的缓存皆指缓存策略
type Cacher interface {
	// 缓存容量
	Capacity() int

	// 设置缓存容量
	SetCapacity(capacity int)

	// 添加节点的到缓存中
	Promote(n *Node)

	// 将节点从缓存中删除，并尝试将其从哈希表中删除
	Ban(n *Node)

	// 将节点从缓存中删除
	Evict(n *Node)

	// 将所有给定的命名空间的节点从缓存中删除
	EvictNS(ns uint64)

	// 将缓存中的所有节点删除
	EvictAll()

	// 关闭缓存
	Close() error
}
```

 ## 实现

再依次看一下上述各个结构实现与**核心**方法，以及如何组合在一起对外提供缓存的功能。   

 ### Node

Node是缓存系统中存放键值对数据的对象，它是线程安全的，通过引用计数来判断是否需要真正的删除。它的实现比较简单，这里就不贴代码了。   

```go
// Node is a 'cache node'.
type Node struct {
	r *Cache // 所属的Cache

	hash    uint32 // 哈希值
	ns, key uint64 // 命名空间，键

	mu    sync.Mutex // 读写锁，可能被多线程同时使用
	size  int        // 大小
	value Value      // 值

	ref   int32    // 引用计数，为0才能真正删除
	onDel []func() // 删除时的回调函数

	CacheData unsafe.Pointer // 指向缓存的lruNode
}

// 需要caller持锁
func (n *Node) unref() {
	if atomic.AddInt32(&n.ref, -1) == 0 {
		// 引用计数为0才删除
		n.r.delete(n)
	}
}

func (n *Node) unrefLocked() {
	if atomic.AddInt32(&n.ref, -1) == 0 {
		n.r.mu.RLock()
		if !n.r.closed {
			n.r.delete(n)
		}
		n.r.mu.RUnlock()
	}
}


```

 ### Handle

Handle是指向Node的一个指针，主要就是对外提供Release方法，用来释放当前引用。   

```go
type Handle struct {
	n unsafe.Pointer // *Node
}

// 返回当前引用的节点的值
func (h *Handle) Value() Value {
	// 原子的解引用，得到指向的Node
	n := (*Node)(atomic.LoadPointer(&h.n))
	if n != nil {
		return n.value
	}
	return nil
}

func (h *Handle) Release() {
	nPtr := atomic.LoadPointer(&h.n)
	if nPtr != nil && atomic.CompareAndSwapPointer(&h.n, nPtr, nil) {
		n := (*Node)(nPtr)
		n.unrefLocked()
	}
}

```

 ### lru

lruNode就是一个双向循环链表，提供insert和remove两个方法，双向链表的正常插入删除操作。   

```go
// 双向循环链表
type lruNode struct {
	n   *Node   // 指向哈希表总的节点
	h   *Handle // Node的一个copy
	ban bool    // 是否被删除

	next, prev *lruNode // 前后lruNode节点
}

func (n *lruNode) insert(at *lruNode) {
	x := at.next
	at.next = n
	n.prev = at
	n.next = x
	x.prev = n
}

func (n *lruNode) remove() {
	if n.prev != nil {
		n.prev.next = n.next
		n.next.prev = n.prev
		n.prev = nil
		n.next = nil
	} else {
		panic("BUG: removing removed node")
	}
}
```

lru实现了Cacher接口，这里只挑几个比较重要方法详细解释。   

```go
// lru缓存策略
type lru struct {
	mu       sync.Mutex // 读写锁，需要保证并发安全
	capacity int        // 容量
	used     int        // lru的大小
	recent   lruNode    // 最近的lruNode，双向循环链表头节点
}

// 设置容量
func (r *lru) SetCapacity(capacity int) {
	var evicted []*lruNode

	r.mu.Lock()
	r.capacity = capacity
	for r.used > r.capacity {
		// 当前的lruNode超过容量
		// 删除最旧的节点
		rn := r.recent.prev
		if rn == nil {
			panic("BUG: invalid LRU used or capacity counter")
		}
		rn.remove()
		rn.n.CacheData = nil
		r.used -= rn.n.Size()
		evicted = append(evicted, rn)
	}
	r.mu.Unlock()

	for _, rn := range evicted {
		
		rn.h.Release()
	}
}

// 添加Node到lru
func (r *lru) Promote(n *Node) {
	var evicted []*lruNode

	r.mu.Lock()
	if n.CacheData == nil {
		// 当前Node不在lru中
		if n.Size() <= r.capacity {
			// 还有容量，将Node添加到头部
			rn := &lruNode{n: n, h: n.GetHandle()}
			rn.insert(&r.recent)
			n.CacheData = unsafe.Pointer(rn)
			r.used += n.Size()

			for r.used > r.capacity {
				// 添加后超出容量，删除最旧的lruNode
				rn := r.recent.prev
				if rn == nil {
					panic("BUG: invalid LRU used or capacity counter")
				}
				rn.remove()
				rn.n.CacheData = nil
				r.used -= rn.n.Size()
				evicted = append(evicted, rn)
			}
		}
	} else {
		// 当前Node在lru中
		rn := (*lruNode)(n.CacheData)
		if !rn.ban {
			// 将当前Node移至头节点
			rn.remove()
			rn.insert(&r.recent)
		}
	}
	r.mu.Unlock()

	for _, rn := range evicted {
		// 通过Handle.Release 释放当前Node的引用
		rn.h.Release()
	}
}

// 删除Node
func (r *lru) Ban(n *Node) {
	r.mu.Lock()
	if n.CacheData == nil {
		// 不在lru链表中
		n.CacheData = unsafe.Pointer(&lruNode{n: n, ban: true})
	} else {
		rn := (*lruNode)(n.CacheData)
		if !rn.ban {
			// 将指向Node的lruNode从lru链表中删除
			// 将ban置为true，表示已删除
			rn.remove()
			rn.ban = true
			r.used -= rn.n.Size()
			r.mu.Unlock()

			rn.h.Release()
			rn.h = nil
			return
		}
	}
	r.mu.Unlock()
}

```

 ### mBucket

mBucket即为哈希桶，最小的封锁粒度。核心方法为get和delete，添加通过get实现。添加或删除Node后需要判断当前哈希表是否需要进行收缩或者扩张。   ![image.png](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/y_image.png)    当发生哈希表扩张的条件为：   

1. 整个Cache中，Node的个数超过预定的阈值（默认初始状态下哈希桶的个数为16个，每个桶中可存储32个数据项，即总量的阈值为哈希桶个数乘以每个桶的容量上限）；   
1. 当Cache中出现了数据不平衡的情况。当某些桶的数据量超过了32个数据，即被视作数据发生散列不平衡。当这种不平衡累积值超过预定的阈值（128）个时，就需要进行扩张。   

扩张的过程为：   

1. 计算新哈希表的哈希桶个数（扩大一倍）；   
1. 创建一个空的哈希表，并将旧的哈希表（主要为所有哈希桶构成的数组）转换成一个过渡期的哈希表，表中的每个哈希桶都被**冻结**；   
1. 后台利用过渡期哈希表中的被冻结的哈希桶信息对新的哈希表进行内容构建。   

收缩的条件为：哈希表中Node的个数少于哈希桶的一半。   
收缩的过程与扩张类似，只不过新表中桶的个数为旧表的一半。   

```go
// 哈希桶
type mBucket struct {
	mu     sync.Mutex // 读写锁，哈希表扩张的过程中，最小的封锁粒度为桶级别
	node   []*Node    // 节点指针数组
	frozen bool       // 是否冻结，用于桶中数据转移
}

// 查找Node，也可以通过将noset置为false来添加Node
// 返回值：是否进行查找操作，是否新增Node，对应的Node
// 添加后需要判断当前哈希表是否需要进行扩张
func (b *mBucket) get(r *Cache, h *mNode, hash uint32, ns, key uint64, noset bool) (done, added bool, n *Node) {
	b.mu.Lock()

	if b.frozen {
		// 如果当前桶被冻结了，不操作
		b.mu.Unlock()
		return
	}

	// Scan the node.
	for _, n := range b.node {
		// 遍历当前桶，查找对应的Node
		if n.hash == hash && n.ns == ns && n.key == key {
			// 在当前桶中，引用计数+1，并将其返回
			atomic.AddInt32(&n.ref, 1)
			b.mu.Unlock()
			return true, false, n
		}
	}

	// Get only.
	if noset {
		// 不在当前桶中，noset为true，不新建Node，返回
		b.mu.Unlock()
		return true, false, nil
	}

	// Create node.
	// 新建Node，并放入当前桶中
	n = &Node{
		r:    r,
		hash: hash,
		ns:   ns,
		key:  key,
		ref:  1,
	}
	// Add node to bucket.
	b.node = append(b.node, n)
	bLen := len(b.node)
	b.mu.Unlock()

	// Update counter.
	// Cache中的节点数是是否超出阈值
	grow := atomic.AddInt32(&r.nodes, 1) >= h.growThreshold
	if bLen > mOverflowThreshold {
		// 当前桶Node个数超出32个，为不平衡
		// Cache中不平衡桶的个数超出128个，或者Node数超出阈值，需要扩张
		grow = grow || atomic.AddInt32(&h.overflow, 1) >= mOverflowGrowThreshold
	}

	// Grow.
	if grow && atomic.CompareAndSwapInt32(&h.resizeInProgess, 0, 1) {
		// 扩张，将resizeInProgess标志位设为1
		// 新桶的数量扩大一倍
		nhLen := len(h.buckets) << 1
		nh := &mNode{
			buckets:         make([]unsafe.Pointer, nhLen),
			mask:            uint32(nhLen) - 1,
			pred:            unsafe.Pointer(h), // 新桶的pred指向旧桶
			growThreshold:   int32(nhLen * mOverflowThreshold),
			shrinkThreshold: int32(nhLen >> 1), // 哈希表中Node的个数少于哈希桶的个数时
		}
		// Cache的mHead指针指向新表
		ok := atomic.CompareAndSwapPointer(&r.mHead, unsafe.Pointer(h), unsafe.Pointer(nh))
		if !ok {
			panic("BUG: failed swapping head")
		}
		// 后台进行哈希表的构建（数据转移）
		go nh.initBuckets()
	}

	return true, true, n
}

// 删除Node
// 删除后需要判断当前哈希表是否需要收缩
func (b *mBucket) delete(r *Cache, h *mNode, hash uint32, ns, key uint64) (done, deleted bool) {
	b.mu.Lock()

	if b.frozen {
		b.mu.Unlock()
		return
	}

	// Scan the node.
	var (
		n    *Node
		bLen int
	)
	for i := range b.node {
		n = b.node[i]
		if n.ns == ns && n.key == key {
			if atomic.LoadInt32(&n.ref) == 0 {
				deleted = true

				// Call releaser.
				if n.value != nil {
					if r, ok := n.value.(util.Releaser); ok {
						r.Release()
					}
					n.value = nil
				}

				// Remove node from bucket.
				b.node = append(b.node[:i], b.node[i+1:]...)
				bLen = len(b.node)
			}
			break
		}
	}
	b.mu.Unlock()

	if deleted {
		// Call OnDel.
		for _, f := range n.onDel {
			f()
		}

		// Update counter.
		// 收缩的条件： 哈希表中数据项的个数少于哈希桶的个数时
		atomic.AddInt32(&r.size, int32(n.size)*-1)
		shrink := atomic.AddInt32(&r.nodes, -1) < h.shrinkThreshold
		if bLen >= mOverflowThreshold {
			// 当前桶不平衡，删除了一个Node，所以需要将overflow的Node少了一个
			atomic.AddInt32(&h.overflow, -1)
		}

		// Shrink.
		if shrink && len(h.buckets) > mInitialSize && atomic.CompareAndSwapInt32(&h.resizeInProgess, 0, 1) {
			// 新桶数量为旧桶的一半
			nhLen := len(h.buckets) >> 1
			nh := &mNode{
				buckets:         make([]unsafe.Pointer, nhLen),
				mask:            uint32(nhLen) - 1,
				pred:            unsafe.Pointer(h),
				growThreshold:   int32(nhLen * mOverflowThreshold),
				shrinkThreshold: int32(nhLen >> 1),
			}
			ok := atomic.CompareAndSwapPointer(&r.mHead, unsafe.Pointer(h), unsafe.Pointer(nh))
			if !ok {
				panic("BUG: failed swapping head")
			}
			// 后台构建
			go nh.initBuckets()
		}
	}

	return true, deleted
}

```

 ### mHead

mHead即为哈希表，主要关注initBucket，看如果构建哈希表的。   

当哈希表扩张时，构建一个新的哈希桶其实就是将一个旧哈希桶中的数据拆分成两个新的哈希桶。 

拆分的规则很简单：   

1. 利用散列函数对Node的key值进行计算；   
1. 将第1步得到的结果取哈希桶个数的余，得到哈希桶的ID；   

因此拆分时仅需要将Node的key的散列值对新的哈希桶个数取余即可。

收缩的过程也类似，将两个旧桶合并成一个。   

```go
// 哈希表
type mNode struct {
	buckets         []unsafe.Pointer // []*mBucket 指向表中所有桶的指针数组
	mask            uint32           // 掩码，用于快速找到对应的桶
	pred            unsafe.Pointer   // *mNode 指向旧桶的指针，用于扩张或收缩
	resizeInProgess int32            // 是否正在扩张或收缩

	overflow        int32 // 不平衡Node个数，所有的桶中超过32个Node的个数
	growThreshold   int32 // 扩张阈值，表中Node个数超过则进行扩张
	shrinkThreshold int32 // 收缩阈值，表中Node个数少于则进行收缩
}

// 构建哈希桶
func (n *mNode) initBucket(i uint32) *mBucket {
	if b := (*mBucket)(atomic.LoadPointer(&n.buckets[i])); b != nil {
		// 当前桶以及被构建过了，直接返回
		return b
	}

	p := (*mNode)(atomic.LoadPointer(&n.pred))
	if p != nil {
		var node []*Node
		if n.mask > p.mask {
			// Grow.
			// 需要进行扩张
			// i&p.mask，取余获取旧表对应桶的索引
			pb := (*mBucket)(atomic.LoadPointer(&p.buckets[i&p.mask]))
			if pb == nil {
				pb = p.initBucket(i & p.mask)
			}
			// 将旧桶冻结
			m := pb.freeze()
			// Split nodes.
			for _, x := range m {
				if x.hash&n.mask == i {
					// 重新散列到新桶中
					node = append(node, x)
				}
			}
		} else {
			// Shrink.
			pb0 := (*mBucket)(atomic.LoadPointer(&p.buckets[i]))
			if pb0 == nil {
				pb0 = p.initBucket(i)
			}
			pb1 := (*mBucket)(atomic.LoadPointer(&p.buckets[i+uint32(len(n.buckets))]))
			if pb1 == nil {
				pb1 = p.initBucket(i + uint32(len(n.buckets)))
			}
			m0 := pb0.freeze()
			m1 := pb1.freeze()
			// Merge nodes.
			node = make([]*Node, 0, len(m0)+len(m1))
			node = append(node, m0...)
			node = append(node, m1...)
		}
		b := &mBucket{node: node}
		if atomic.CompareAndSwapPointer(&n.buckets[i], nil, unsafe.Pointer(b)) {
			if len(node) > mOverflowThreshold {
				atomic.AddInt32(&n.overflow, int32(len(node)-mOverflowThreshold))
			}
			return b
		}
	}

	return (*mBucket)(atomic.LoadPointer(&n.buckets[i]))
}

```

 ### Cache

Cache即为缓存系统，作用前文已经介绍了，这里不再赘述。这里要着重说一下getBucket，前面介绍哈希表扩张和收缩的时候提过，最小的封锁粒度为哈希桶级别，**构建新表的过程中，哈希表并不会拒绝服务，所有的操作仍然可以进行**。

当有新的读写请求发生时，若被散列之后得到的哈希桶仍然未构建完成，则**主动**进行构建，并将构建后的哈希桶填入新的哈希表中。后台进程构建到该桶时，发现已经被构建了，则无需重复构建。   

```go
func (r *Cache) getBucket(hash uint32) (*mNode, *mBucket) {
	h := (*mNode)(atomic.LoadPointer(&r.mHead))
	i := hash & h.mask
	b := (*mBucket)(atomic.LoadPointer(&h.buckets[i]))
	if b == nil {
		// 主动进行哈希桶构建
		b = h.initBucket(i)
	}
	return h, b
}

```

Cache对Leveldb提供两个主要的接口，Get和Delete，用来添加和删除数据，当然还有一些其他比较简单的接口，这里也不逐行分析，比较简单。

至此，我们对Leveldb的缓存系统的分析已经完成了，完整的注释代码在[[goleveldb-annotation](https://github.com/wuliuqii/goleveldb-annotation/tree/master/leveldb/cache)。

