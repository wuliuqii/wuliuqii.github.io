---
title: "Goleveldb源码分析（一）编码"
date: 2022-03-31T08:30:25+08:00
categories: [goleveldb]
tags: [go, 读点源码]
draft: false
---

研究一下基于Go语言的[Leveldb](https://github.com/syndtr/goleveldb)的源码，参考作者的配套[wiki](https://leveldb-handbook.readthedocs.io/zh/latest/rwopt.html)。

Leveldb的整体流程如下图所示
![image.png](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/image.png)    

 ## 基础

先从一些外围基础的概念开始看起。

### 编码

Leveldb使用encoding/binary标准库对数据进行编解码，这里将详细介绍它的规则。

 #### [大小端](https://www.wikiwand.com/en/Endianness)

字节序简单来说就是多字节对象在内存中的排列顺序，主要分为两种，大端序和小端序。

大端序，高位存在较大地址处。
<img src="https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/x_image.png" alt="image.png" style="zoom:50%;" /> 
<img src="https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/9_image.png" alt="image.png" style="zoom:67%;" />    

示例中，最高位字节是0x0A 存储在最低的内存地址处。下一个字节0x0B存在后面的地址处。正类似于十六进制字节从左到右的阅读顺序。

小端序，低位存在较大地址处。
<img src="https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/b_image.png" alt="image.png" style="zoom:50%;" /> 
<img src="https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/c_image.png" alt="image.png" style="zoom:67%;" />    

最低位字节是0x0D 存储在最低的内存地址处。后面字节依次存在后面的地址处。   

在encoding/binary中定义了字节序的接口，同时也定义了大小端   

```go
type ByteOrder interface {
	Uint16([]byte) uint16
	Uint32([]byte) uint32
	Uint64([]byte) uint64
	PutUint16([]byte, uint16)
	PutUint32([]byte, uint32)
	PutUint64([]byte, uint64)
	String() string
}

type littleEndian struct{}
var LittleEndian littleEndian

type bigEndian struct{}
var BigEndian bigEndian

```

看下编码的实例   

```go
package main

import (
	"encoding/binary"
	"fmt"
)

func BigEndian() { // 大端序
	// 二进制形式：0000 0000 0000 0000 0001 0002 0003 0004
	var testInt int32 = 0x01020304 // 十六进制表示
	fmt.Printf("%d use big endian: \n", testInt)

	var testBytes []byte = make([]byte, 4)
	binary.BigEndian.PutUint32(testBytes, uint32(testInt)) //大端序模式
	fmt.Println("int32 to bytes:", testBytes)

	convInt := binary.BigEndian.Uint32(testBytes) //大端序模式的字节转为int32
	fmt.Printf("bytes to int32: %d\n\n", convInt)
}

func LittleEndian() { // 小端序
	//二进制形式： 0000 0000 0000 0000 0001 0002 0003 0004
	var testInt int32 = 0x01020304 // 16进制
	fmt.Printf("%d use little endian: \n", testInt)

	var testBytes []byte = make([]byte, 4)
	binary.LittleEndian.PutUint32(testBytes, uint32(testInt)) //小端序模式
	fmt.Println("int32 to bytes:", testBytes)

	convInt := binary.LittleEndian.Uint32(testBytes) //小端序模式的字节转换
	fmt.Printf("bytes to int32: %d\n\n", convInt)
}

func main() {
	BigEndian()
	LittleEndian()
}

// output：
// 16909060 use big endian: 
// int32 to bytes: [1 2 3 4]
// bytes to int32: 16909060

// 16909060 use little endian: 
// int32 to bytes: [4 3 2 1]
// bytes to int32: 16909060
```

 ### varint

varint是一种使用一个或多个字节序列化整数的方法，会把整数编码为变长字节。对于32位整型数据经过varint编码后需要1-5个字节，小的数字使用1个byte，大的数字使用5个bytes。64位整型数据编码后占用1~10个字节。在实际场景中小数字的使用率远远多于大数字，因此通过varint编码对于大部分场景都可以起到很好的压缩效果。   
看下源码：   

```go
package binary

// 编码
func PutUvarint(buf []byte, x uint64) int {
	i := 0
	for x >= 0x80 {
		// 从低位往高位依此取7bit，标志位置为1，组成一个byte
		buf[i] = byte(x) | 0x80
		x >>= 7
		i++
	}
	// 最后一个byte，标志位为0
	buf[i] = byte(x)
	return i + 1
}

// 解码
func Uvarint(buf []byte) (uint64, int) {
	var x uint64
	var s uint
	for i, b := range buf {
		if i == MaxVarintLen64 {
			// Catch byte reads past MaxVarintLen64.
			// See issue https://golang.org/issues/41185
			return 0, -(i + 1) // overflow
		}
		if b < 0x80 {
			if i == MaxVarintLen64-1 && b > 1 {
				return 0, -(i + 1) // overflow
			}
			// 最后一个byte
			return x | uint64(b)<<s, i + 1
		}
		// 取当前byte的后7bit
		x |= uint64(b&0x7f) << s
		s += 7
	}
	return 0, 0
}

```

看个实例：   

```go
func main() {
	buf := make([]byte, binary.MaxVarintLen64)
	var x int64 = 123456
	// 二进制形式：
	// 0001 1110 0010 0100 0000
	// 按照7bit一组：
	// 0000111 1000100 1000000
	// 从低位往高位以此取7bit，重新编码，注意是反转排序，全0的bit舍弃
	// 1000000 1000100 0000111
	// 再加上标志位，每7bit组成一个字节，如果后面还有字节，则标志位为1，反之为0
	// varint编码：
	// 11000000 11000100 00000111
	// 0xc0		0xc4	 0x07
	binary.PutVarint(buf, x)
	fmt.Printf("%b %x\n", buf)
	n, _ := binary.Uvarint(buf)
	fmt.Printf("%b\n", n)
}

// output：
// [11000000 11000100 111 0 0 0 0 0 0 0]
// c0c40700000000000000
// 11110001001000000
```

