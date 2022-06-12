---
title: "Goleveldb源码分析（四）日志"
date: 2022-04-05T09:13:33+08:00
categories: [goleveldb]
tags: [go, 读点源码]
draft: true
---

为了防止写入内存的数据库因为进程异常、系统掉电等情况发生丢失，Leveldb在写内存之前会将本次写操作的内容写入日志文件中。

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/two_log.jpeg)

在Leveldb中，有两个memory db，以及对应的两份日志文件。其中一个memory db是可读写的，当这个db的数据量超过预定的上限时，便会转换成一个不可写的memory db，与此同时，与之对应的日志文件也变成一份frozen log。

而新生成的immutable memory db则会由后台的minor compaction进程将其转换成一个sstable文件进行持久化，持久化完成，与之对应的frozen log被删除。

在本文中主要分析日志的结构、写入读取操作。

## 日志结构

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/journal.jpeg)

为了增加读取效率，日志文件中按照block进行划分，每个block的大小为32KiB。每个block中包含了若干个完整的chunk。

一条日志记录包含一个或多个chunk。每个chunk包含了一个7字节大小的header，前4字节是该chunk的校验码，紧接的2字节是该chunk数据的长度，以及最后一个字节是该chunk的类型。其中checksum校验的范围包括chunk的类型以及随后的data数据。

chunk共有四种类型：full，first，middle，last。一条日志记录若只包含一个chunk，则该chunk的类型为full。若一条日志记录包含多个chunk，则这些chunk的第一个类型为first, 最后一个类型为last，中间包含大于等于0个middle类型的chunk。

由于一个block的大小为32KiB，因此当一条日志文件过大时，会将第一部分数据写在第一个block中，且类型为first，若剩余的数据仍然超过一个block的大小，则第二部分数据写在第二个block中，类型为middle，最后剩余的数据写在最后一个block中，类型为last。

## 日志写

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/journal_write.jpeg)

日志写入流程较为简单，在leveldb内部，实现了一个journal的writer。首先调用Next函数获取一个singleWriter，这个singleWriter的作用就是写入**一条journal记录**。

singleWriter开始写入时，标志着第一个chunk开始写入。在写入的过程中，不断判断writer中buffer的大小，若超过32KiB，将chunk开始到现在做为一个完整的chunk，为其计算header之后将整个chunk写入文件。与此同时reset buffer，开始新的chunk的写入。

若一条journal记录较大，则可能会分成几个chunk存储在若干个block中。

```go
type Writer struct {
	w io.Writer
	// 充当逻辑时钟
	seq int
	f flusher
	// buf[i:j] 表示当前的chunk，i下标包括chunk header
	i, j int
	// 已经写入的下标
	written int
	// 是否为第一个chunk
	first bool
	// 是否数据已经写入buf，但是还没有落盘（当前block未满32KiB）
	pending bool
	err error
	// 每个block为32KiB
	buf [blockSize]byte
}

// chunk header:
// +-------------------+-----------------+--------------------+
// | Checksum(4-bytes) | Length(2-bytes) | Chunk type(1-byte) |
// +-------------------+-----------------+--------------------+
func (w *Writer) fillHeader(last bool) {
	if w.i+headerSize > w.j || w.j > blockSize {
		panic("leveldb/journal: bad writer state")
	}
	if last {
		if w.first {
			w.buf[w.i+6] = fullChunkType
		} else {
			w.buf[w.i+6] = lastChunkType
		}
	} else {
		if w.first {
			w.buf[w.i+6] = firstChunkType
		} else {
			w.buf[w.i+6] = middleChunkType
		}
	}
	binary.LittleEndian.PutUint32(w.buf[w.i+0:w.i+4], util.NewCRC(w.buf[w.i+6:w.j]).Value())
	binary.LittleEndian.PutUint16(w.buf[w.i+4:w.i+6], uint16(w.j-w.i-headerSize))
}

// 将当前block写入文件
func (w *Writer) writeBlock() {
	_, w.err = w.w.Write(w.buf[w.written:])
	w.i = 0
	w.j = headerSize
	w.written = 0
}

// 将buf中的数据写入文件
func (w *Writer) writePending() {
	if w.err != nil {
		return
	}
	if w.pending {
		w.fillHeader(true)
		w.pending = false
	}
	_, w.err = w.w.Write(w.buf[w.written:w.j])
	w.written = w.j
}

// 将当前block写入文件
func (w *Writer) writeBlock() {
	_, w.err = w.w.Write(w.buf[w.written:])
	w.i = 0
	w.j = headerSize
	w.written = 0
}

// 返回回一个singleWriter，用来写入一条日志
func (w *Writer) Next() (io.Writer, error) {
	w.seq++
	if w.err != nil {
		return nil, w.err
	}
	if w.pending {
		// 为上一个chunk赋header
		w.fillHeader(true)
	}
	w.i = w.j
	w.j = w.j + headerSize
	// Check if there is room in the block for the header.
	if w.j > blockSize {
		// 剩余空间不够写入一个header
		// 将剩余空间置空
		// Fill in the rest of the block with zeroes.
		for k := w.i; k < blockSize; k++ {
			w.buf[k] = 0
		}
		// 将32KiB的数据写入文件
		w.writeBlock()
		if w.err != nil {
			return nil, w.err
		}
	}
	w.first = true
	w.pending = true
	return singleWriter{w, w.seq}, nil
}

type singleWriter struct {
	w   *Writer
	seq int // sequence，充当逻辑时钟的作用
}

// 写入一条日志记录
// 返回值：写入的字节长度，错误
func (x singleWriter) Write(p []byte) (int, error) {
	w := x.w
	// 判断当前写入是否ok
	if w.seq != x.seq {
		return 0, errors.New("leveldb/journal: stale writer")
	}
	if w.err != nil {
		return 0, w.err
	}
	n0 := len(p)
	for len(p) > 0 {
		// Write a block, if it is full.
		if w.j == blockSize {
			// 32KiB已满
			// 写入文件
			w.fillHeader(false)
			w.writeBlock()
			if w.err != nil {
				return 0, w.err
			}
			w.first = false
		}
		// Copy bytes into the buffer.
		n := copy(w.buf[w.j:], p)
		w.j += n
		p = p[n:]
	}
	return n0, nil
}
```

日志的内容为**写入的batch编码后的信息**。

具体的格式为：

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/journal_content.jpeg)

一条日志记录的内容包含：

-   Header
-   Data

其中header中有（1）当前db的sequence number（2）本次日志记录中所包含的put/del操作的个数。

```go
// batch header:
// +----------+--------------+
// | sequence | entry number |
// +----------+--------------+
func encodeBatchHeader(dst []byte, seq uint64, batchLen int) []byte {
	dst = ensureBuffer(dst, batchHeaderLen)
	binary.LittleEndian.PutUint64(dst, seq)
	binary.LittleEndian.PutUint32(dst[8:], uint32(batchLen))
	return dst
}

// 写入日志文件
// 写入的内容:
// +--------------+------+-----+------+
// | batch header | data | ... | data |
// +--------------+------+-----+------+
func writeBatchesWithHeader(wr io.Writer, batches []*Batch, seq uint64) error {
	if _, err := wr.Write(encodeBatchHeader(nil, seq, batchesLen(batches))); err != nil {
		return err
	}
	for _, batch := range batches {
		if _, err := wr.Write(batch.data); err != nil {
			return err
		}
	}
	return nil
}

// 写日志
func (db *DB) writeJournal(batches []*Batch, seq uint64, sync bool) error {
	// 获取singleWrite
	wr, err := db.journal.Next()
	if err != nil {
		return err
	}
	// 写入日志文件
	if err := writeBatchesWithHeader(wr, batches, seq); err != nil {
		return err
	}
	// 确保落盘
	if err := db.journal.Flush(); err != nil {
		return err
	}
	if sync {
		return db.journalWriter.Sync()
	}
	return nil
}
```

## 日志读

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/journal_read.jpeg)

同样，日志读取也较为简单。为了避免频繁的IO读取，每次从文件中读取数据时，按block（32KiB）进行块读取。

每次读取一条日志记录，reader调用Next函数返回一个singleReader。singleReader每次调用Read函数就返回一个chunk的数据。每次读取一个chunk，都会检查这批数据的校验码、数据类型、数据长度等信息是否正确，若不正确，且用户要求严格的正确性，则返回错误，否则丢弃整个chunk的数据。

循环调用singleReader的read函数，直至读取到一个类型为Last的chunk，表示整条日志记录都读取完毕，返回。

读日志的代码和写的差不多，只不过做多更多的校验，这里不再贴出来。
