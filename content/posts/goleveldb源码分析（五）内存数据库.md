---
title: "Goleveldb源码分析（五）内存数据库"
date: 2022-04-07T08:13:20+08:00
categories: [goleveldb]
tags: [go, 读点源码]
draft: true
---

leveldb中内存数据库用来维护有序的key-value对，其底层是利用跳表实现，绝大多数操作（读／写）的时间复杂度均为O(log n)，有着与平衡树相媲美的操作效率，但是从实现的角度来说简单许多，因此在本文中将介绍一下内存数据库的实现细节。

## 跳表

关于跳表的详细介绍这里不再展开，想了解的可以去看论文[Skip Lists: A Probabilistic Alternative to Balanced Trees](https://15721.courses.cs.cmu.edu/spring2018/papers/08-oltpindexes1/pugh-skiplists-cacm1990.pdf)，这里只讲Leveldb中的实现。

### 结构

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/skiplist_arch.png)

一个跳表的结构示意图如上所示。

跳表是按层建造的。底层是一个普通的**有序**链表。每个更高层都充当下面链表的**快速通道**，这里在层 $i$ 中的元素按某个固定的概率 $p$ (通常为0.5或0.25)出现在层 $i+1$ 中。平均起来，每个元素都在 $1/(1-p)$ 个列表中出现，而最高层的元素（通常是在跳跃列表前端的一个特殊的头元素）在 $O((log1/p)*n)$ 个列表中出现。

goleveldb中的跳表并没有单独抽象为一个结构体，而是嵌入到了DB结构中，同时它使用的实现方式是数组而非链表。这里的kvData数组存储key-value数据项，nodeData数组存储每个节点的链接信息，包括当前节点key-value值在kvData数组中的偏移量，层高，以及每层对应的下一个节点在nodeData中的索引值（下个节点的kv offset的索引值）。

```go
const (
	nKV     = iota // nodeData[0]
	nKey           // [1]
	nVal           // [2]
	nHeight        // [3]
	nNext          // [4]
)

// 内存数据库
type DB struct {
	// 用来比较key值
	cmp comparer.BasicComparer
	rnd *rand.Rand

	mu sync.RWMutex
	// 存储每一条数据
	// +------+--------+------+--------+-----+-------+---------+
	// | key1 | value1 | key2 | value2 | ... | key n | value n |
	// +------+--------+------+--------+-----+-------+---------+
	kvData []byte
	// 存储每个跳表节点的链接信息
	// +------------+-------------+---------------+--------+----------+-----+-----+---------+------------+-------------+---------------+--------+----------+-----+-----+---------+
	// | kv1 offset | key1 length | value1 length | height | 1st next | 2nd | ... | heights | kv2 offset | key2 length | value2 length | height | 1st next | 2nd | ... | heights |
	// +------------+-------------+---------------+--------+----------+-----+-----+---------+------------+-------------+---------------+--------+----------+-----+-----+---------+
	nodeData []int
	// 所有的操作共用，使用前需要持锁
	prevNode [tMaxHeight]int
	// 最大层高
	maxHeight int
	// key-value对个数
	n int
	// key-value的总大小
	kvSize int
}
```

### 查找

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/skiplist_search.jpeg)

查找操作是跳表其他操作的基础。

上图展示了查找一个值为17的链表节点的过程：

-   首先根据跳表的高度选取最高层的头节点；
-   若跳表中的节点内容小于查找节点的内容，则取该层的下一个节点继续比较；
-   若跳表中的节点内容等于查找节点的内容，则直接返回；
-   若跳表中的节点内容大于查找节点的内容，且层高不为0，则降低层高，且从前一个节点开始，重新查找低一层中的节点信息；若层高为0，则返回当前节点，该节点的key大于所查找节点的key。

综合来说，就是利用稀疏的高层节点，快速定位到所需要查找节点的大致位置，再利用密集的底层节点，具体比较节点的内容。

```go
// 找到第一个大于等于给定key的节点的索引
func (p *DB) findGE(key []byte, prev bool) (int, bool) {
	// 头节点
	node := 0
	// 最高层
	h := p.maxHeight - 1
	for {
		// 当前层的下个节点索引
		next := p.nodeData[node+nNext+h]
		cmp := 1
		if next != 0 {
			// 当前层的下个节点
			o := p.nodeData[next]
			// 比较当下个节点的key和给定key的大小
			// 1为大于，0为等于，-1小于
			cmp = p.cmp.Compare(p.kvData[o:o+p.nodeData[next+nKey]], key)
		}
		if cmp < 0 {
			// Keep searching in this list
			// 在当前层继续搜索
			node = next
		} else {
			// 当前层节点以及大于等于给定key了
			if prev {
				p.prevNode[h] = node
			} else if cmp == 0 {
				// 等于
				return next, true
			}
			if h == 0 {
				// 大于
				return next, cmp == 0
			}
			// 往下层搜索
			h--
		}
	}
}
```

### 插入

插入操作借助于查找操作实现。

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/skiplist_insert.jpeg)

-   在查找的过程中，不断记录**每一层**的**前任节点**，如图中红色圆圈所表示的；
-   为新插入的节点随机产生层高（随机产生层高的算法较为简单，依赖最高层数和概率值p，可见下文中的代码实现）；
-   在合适的位置插入新节点（例如图中节点12与节点19之间），并依据查找时记录的前任节点信息，在每一层中，**以链表插入**的方式，将该节点插入到每一层的链接中。

**链表插入**指：将当前节点的Next值置为前任节点的Next值，将前任节点的Next值替换为当前节点。

```go
func (p *DB) randHeight() (h int) {
	const branching = 4
	h = 1
	for h < tMaxHeight && p.rnd.Int()%branching == 0 {
		// h层的概率为0.25^h
		h++
	}
	return
}

// 插入节点
func (p *DB) Put(key []byte, value []byte) error {
	p.mu.Lock()
	defer p.mu.Unlock()

	if node, exact := p.findGE(key, true); exact {
		// 当前跳表中已有给定key的node
		// 覆盖掉原来的
		kvOffset := len(p.kvData)
		p.kvData = append(p.kvData, key...)
		p.kvData = append(p.kvData, value...)
		p.nodeData[node] = kvOffset
		m := p.nodeData[node+nVal]
		p.nodeData[node+nVal] = len(value)
		p.kvSize += len(value) - m
		return nil
	}

	// 随机高度
	h := p.randHeight()
	if h > p.maxHeight {
		for i := p.maxHeight; i < h; i++ {
			// 每层链表的初始前驱节点为头节点
			p.prevNode[i] = 0
		}
		p.maxHeight = h
	}

	// 插入节点
	kvOffset := len(p.kvData)
	p.kvData = append(p.kvData, key...)
	p.kvData = append(p.kvData, value...)
	// Node
	node := len(p.nodeData)
	p.nodeData = append(p.nodeData, kvOffset, len(key), len(value), h)
	for i, n := range p.prevNode[:h] {
		// 更新每层的前后节点
		m := n + nNext + i
		p.nodeData = append(p.nodeData, p.nodeData[m])
		p.nodeData[m] = node
	}

	p.kvSize += len(key) + len(value)
	p.n++
	return nil
}
```

### 删除

跳表的删除操作较为简单，依赖查找过程找到该节点在整个跳表中的位置后，**以链表删除**的方式，在每一层中，删除该节点的信息。

**链表删除**指：将前任节点的Next值替换为当前节点的Next值，并将当前节点所占的资源释放。

```go
// 删除节点
func (p *DB) Delete(key []byte) error {
	p.mu.Lock()
	defer p.mu.Unlock()

	node, exact := p.findGE(key, true)
	if !exact {
		return ErrNotFound
	}

	// 将前任节点的Next值替换为当前节点的Next值
	// 并将当前节点所占的资源释放
	h := p.nodeData[node+nHeight]
	for i, n := range p.prevNode[:h] {
		m := n + nNext + i
		p.nodeData[m] = p.nodeData[p.nodeData[m]+nNext+i]
	}

	p.kvSize -= p.nodeData[node+nKey] + p.nodeData[node+nVal]
	p.n--
	return nil
}
```

还有一些其他的操作，这里不再赘述，基本操作都以来上面的实现。

## 键值

这里再介绍一下内存数据库中的键值编码和比较规则。

### 键值编码

由于内存数据库本质是一个kv集合，且所有的数据项都是依据key值排序的，因此键值的编码规则尤为关键。

内存数据库中，key称为internalKey，其由三部分组成：

-   用户定义的key：这个key值也就是原生的key值；
-   序列号：leveldb中，每一次写操作都有一个sequence number，标志着写入操作的先后顺序。由于在leveldb中，可能会有多条相同key的数据项同时存储在数据库中，因此需要有一个序列号来标识这些数据项的新旧情况。序列号最大的数据项为最新值；
-   类型：标志本条数据项的类型，为更新还是删除；

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/internalkey.jpeg)

```go
// 内存数据库键值
type internalKey []byte

// 编码
// +------+--------------------------+--------------+
// | uKey | sequence number(7 bytes) | type(1 byte) |
// +------+--------------------------+--------------+
func makeInternalKey(dst, ukey []byte, seq uint64, kt keyType) internalKey {
	if seq > keyMaxSeq {
		panic("leveldb: invalid sequence number")
	} else if kt > keyTypeVal {
		panic("leveldb: invalid type")
	}

	dst = ensureBuffer(dst, len(ukey)+8)
	copy(dst, ukey)
	binary.LittleEndian.PutUint64(dst[len(ukey):], (seq<<8)|uint64(kt))
	return internalKey(dst)
}

// 解码
func parseInternalKey(ik []byte) (ukey []byte, seq uint64, kt keyType, err error) {
	if len(ik) < 8 {
		return nil, 0, 0, newErrInternalKeyCorrupted(ik, "invalid length")
	}
	num := binary.LittleEndian.Uint64(ik[len(ik)-8:])
	seq, kt = uint64(num>>8), keyType(num&0xff)
	if kt > keyTypeVal {
		return nil, 0, 0, newErrInternalKeyCorrupted(ik, "invalid type")
	}
	ukey = ik[:len(ik)-8]
	return
}
```

### 键值比较

内存数据库中所有的数据项都是按照键值比较规则进行排序的。这个比较规则可以由用户自己定制，也可以使用系统默认的。在这里介绍一下系统默认的比较规则。

默认的比较规则：

-   首先按照字典序比较用户定义的key（ukey），若用户定义key值大，整个internalKey就大；
-   若用户定义的key相同，则序列号大的internalKey值就小；

通过这样的比较规则，则所有的数据项首先按照用户key进行升序排列；当用户key一致时，按照序列号进行降序排列，这样可以保证首先读到序列号大的数据项。

```go
type BasicComparer interface {
	// a < b 返回 -1
	// a == b 返回 0
	// a > b 返回 1
	Compare(a, b []byte) int
}

type bytesComparer struct{}

func (bytesComparer) Compare(a, b []byte) int {
	return bytes.Compare(a, b)
}
```

