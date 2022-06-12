---
title: "Tinykv（一）Standalone KV"
date: 2022-04-30T16:19:01+08:00
categories: [tinykv]
tags: [go, raft, database]
draft: true
---

[TinyKV](https://github.com/wuliuqii/tinykv) 是基于 [TiKV](https://github.com/tikv/tikv) 的 mini 版分布式 kv 数据库存储部分，是 tikv Talent Plan 课程的一部分。

TinyKV 的架构如下图所示：

![overview](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/20220504173437.png)

Project1 主要是实现 Standalone Storage 以及 Raw API 这两部分。

## Standalone Storage 

Standalone KV 基于 [badger](https://github.com/dgraph-io/badger)，实现了 Storage 接口

```go
type Storage interface {
	Start() error
	Stop() error
	Write(ctx *kvrpcpb.Context, batch []Modify) error
	Reader(ctx *kvrpcpb.Context) (StorageReader, error)
}
```

这里需要说一下的是，增加和删除的操作都是通过 Write 方法实现，区别这两种操作的方法是 batch 参数不一样

```go
// Modify is a single modification to TinyKV's underlying storage.
type Modify struct {
	Data interface{}
}

type Put struct {
	Key   []byte
	Value []byte
	Cf    string
}

type Delete struct {
	Key []byte
	Cf  string
}
```

Reader 方法返回的是一个 StorageReader 接口，可以实现单个 key 以及 iteration 的查询方式

```go
type StorageReader interface {
	// When the key doesn't exist, return nil for the value
	GetCF(cf string, key []byte) ([]byte, error)
	IterCF(cf string) engine_util.DBIterator
	Close()
}
```

这里实现增删查改都需要通过 engine_util 包来对 cf 和 key 进行拼接成一个新 key，以此来模拟 cf 的查询。

## Raw API

通过对 Standalone Storage 的封装实现对外的增删查改服务

```go
// RawGet return the corresponding Get response based on RawGetRequest's CF and Key fields
func (server *Server) RawGet(_ context.Context, req *kvrpcpb.RawGetRequest) (*kvrpcpb.RawGetResponse, error) {
	return nil, nil
}

// RawPut puts the target data into storage and returns the corresponding response
func (server *Server) RawPut(_ context.Context, req *kvrpcpb.RawPutRequest) (*kvrpcpb.RawPutResponse, error) {
	return nil, nil
}

// RawDelete delete the target data from storage and returns the corresponding response
func (server *Server) RawDelete(_ context.Context, req *kvrpcpb.RawDeleteRequest) (*kvrpcpb.RawDeleteResponse, error) {
	return nil, nil
}

// RawScan scan the data starting from the start key up to limit. and return the corresponding result
func (server *Server) RawScan(_ context.Context, req *kvrpcpb.RawScanRequest) (*kvrpcpb.RawScanResponse, error) {
	return nil, nil
}
```

## 结果

Project1 整体来说还是比较简单的，最后贴个结果

![image-20220504174655177](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/image-20220504174655177.png)
