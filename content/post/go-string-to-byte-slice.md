---
title: "Golang string转换[]byte遇到的问题"
date: 2021-07-26T21:23:55+08:00
draft: false
tags: ["go"]
categories: [""]
---


线上的服务一直运行稳定，某一天突然就panic了，于是紧急排查，定位到引起panic的代码。原来是与string转byte slice有关。引发panic的代码大概像下面这样：

```go
func foo(u string){
	hash := md5.Sum(*(*[]byte)(unsafe.Pointer(&uid)))
}
```

panic的错误输出为：

```shell
runtime error: slice bounds out of range [:64] with capacity 0
```

上述这种string转byte slice的实现方法是错误的，虽然它可以做到避免数据拷贝，提升性能。但是是有风险的，这次服务的panic就是让我们遇到了它存在的问题。

两个条件碰到一起导致了这次panic的发生：

1. '*(*[]byte)(unsafe.Pointer(&uid))' 这种转换方式产生了一个length大于0但是capacity等于0的byte slice。

2. 这个产生的byte slice的长度大于等于64，在'md5.Sum()'方法中的'd.Write(data)'逻辑中，遇到了'p[:n]'这样的slice表达式

```go
func (d *digest) Write(p []byte) (nn int, err error) {
	// Note that we currently call block or blockGeneric
	// directly (guarded using haveAsm) because this allows
	// escape analysis to see that p and d don't escape.
	nn = len(p)
	d.len += uint64(nn)
	if d.nx > 0 {
		n := copy(d.x[d.nx:], p)
		d.nx += n
		if d.nx == BlockSize {
			if haveAsm {
				block(d, d.x[:])
			} else {
				blockGeneric(d, d.x[:])
			}
			d.nx = 0
		}
		p = p[n:]
	}
	if len(p) >= BlockSize {
		n := len(p) &^ (BlockSize - 1)
		if haveAsm {
			block(d, p[:n])
		} else {
			blockGeneric(d, p[:n])
		}
		p = p[n:]
	}
	if len(p) > 0 {
		d.nx = copy(d.x[:], p)
	}
	return
}
```

根据go官方对'a[:n]'这种表达式的描述，对于array和string类型来说，中括号中索引的取值范围为"0 <= low <= high <= len(a)" ；然而对于slice类型来说，索引的上限是slice的cap(a)而不是length。

> For arrays or strings, the indices are in range if 0 <= low <= high <= len(a), otherwise they are out of range. For slices, the upper index bound is the slice capacity cap(a) rather than the length. 



在这个场景中，生成的byte slice虽然长度大于0，但是capacity却是0，因此引发了panic。


### 为什么这种string转byte slice的方式是错误的

虽然string与byte slice在底层数据结构上相近，但却有一点区别就是byte slice 比string多了一个cap属性。


```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}

type stringStruct struct {
	str unsafe.Pointer
	len int
}
```


所以当以上述方式进行转换的时候，生成的byte slice取不到对应的cap值，只能是读取那块连续内存后边的值，读到啥cap就是啥了。当此时读到的cap值小于length时就会出问题。

下面的代码可以复现这个问题：

```go
func bar(u string){
	dd := ""
	fmt.Println("u:", &u)
	fmt.Println("dd:", &dd)
	bs := *(*[]byte)(unsafe.Pointer(&u))
	hash := md5.Sum(bs)
	fmt.Println("Hello World", hash, len(bs), cap(bs))
}
```


### 正确的高性能转换方式

借助reflect的'StringHeader'和'SliceHeader'.

```go
func string2byteslice(s string) []byte {
    stringHeader := (*reflect.StringHeader)(unsafe.Pointer(&s))

    bh := reflect.SliceHeader{
        Data: stringHeader.Data,
        Len:  stringHeader.Len,
        Cap:  stringHeader.Len,
    }
    return *(*[]byte)(unsafe.Pointer(&bh))
}
```

> 1. [golang unsafe and uintptr pointers](https://www.programmersought.com/article/57634222415/)

> 2. [golang slice expressions](https://golang.org/ref/spec#Slice_expressions)

> 3. [golang中\[\]byte与string转换](https://segmentfault.com/a/1190000037679588)
