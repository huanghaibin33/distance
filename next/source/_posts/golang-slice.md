---
title: golang slice 剖析
translate_title: anatomy-of-golang-slice
categories:
  - 编程语言
  - go
tags:
  - go
  - slice
date: 2019-09-02 17:17:00
---
### 数组
数组是最简单的数据结构，它拥有一块连续的内存，数组里的每个元素都是按顺序存储，因此可以快速地索引到任意元素（通过下标）。在go语言中，数组具有以下特征：
1. 数组分配的内存是连续的
1. 可以通过下标的方式访问数组内的元素
1. 长度固定，不可改变

<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/go/array.png">

注：
1. 数组在C++里是用指针表示，但在go里面是值，因此函数传递数组是拷贝整一串内容
1. 数组类型包括数组长度和元素的类型，只有这两部分都相同，才能互相比较和赋值，因此也能作为map的key
<!-- more -->

### 切片
切片是围绕动态数组概念构建的一种数据结构，可以按需增长和缩小。切片是一个很小的对象，它对底层数组进行了抽象和封装，并提供了相关的操作方法。
切片由3个字段组成，分别是指向底层数组（该数组可以由多个切片共享）的指针，长度（元素的个数）以及容量（切片在不扩容的情况允许增长到的最大元素个数）。
源码：
``` go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/go/slice.png">

### 切片的创建和初始化
创建切片由几种方式，第一种是用内置的make函数，第二种方法是使用切片字面量，另外一种通过切片创建
``` go 
// make 创建
// T 是数据类型
// len表示切片的长度
// cap是切片的容量，可以省略（默认跟长度一样），cap 不能小于len，否则会panic
slice1 := make([]T, len, cap)

// 切片字面量创建
// 下面创建了一个长度和容量均为3的切片 
slice2 := []T{t1, t2, t3}

// 通过切片创建
slice3 := slice2[i:j:k]
```
源码：
``` go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen() 	// panic(errorString("makeslice: len out of range"))
		}
		panicmakeslicecap()		// panic(errorString("makeslice: cap out of range"))
	}
	return mallocgc(mem, et, true)
}
```

### 切片增长
相对数组而言，切片可以动态地增加容量。内置函数append会返回一个新的切片。append总会增加切片的长度，但是容量有可能不会改变，只有在容量不够用的时候，才会创建一个新的底层数组，并将现有的值复制到新数组里。
源码：
``` go
func growslice(et *_type, old slice, cap int) slice {
	...
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.kind&kindNoPointers != 0 {
		p = mallocgc(capmem, nil, false)
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		p = mallocgc(capmem, et, true)
		if writeBarrier.enabled {
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```

扩容的策略大致如下：
1. 需要的容量如果超过2倍原来容量，则按所需容量扩容
1. 如果元素个数不超过1024个，则每次按照两倍进行扩容
1. 如果元素个数超过1024个，则每次扩25%

注：跟redis一样采用空间预分配的方法，减少内存分配的次数。redis是按修改后长度的2倍扩容，而slice是按修改前容量的两倍来扩。
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/go/slice_sds.png">

### 注意事项
切片在平常开发中不可或缺，它在提供便利性的同时，也引入了一些陷阱，在使用切片时注意避免这些坑，下面举一些比较常见的问题：
#### 数据篡改
切片的底层是共享的，所以修改一个切片的内容，有可能会影响另一个切片
``` go 
// 创建一个整型切片
// 其长度和容量都是5个元素
slice := []int{10, 20, 30, 40, 50}
// 创建一个新切片
// 其长度是2个元素，容量是4个元素 newSlice := slice[1:3]
newSlice := slice[1:3]
// 修改newSlice 索引为1的元素
// 同时也修改了原来的slice 索引为2的元素 newSlice[1] = 35
newSlice[1] = 35
fmt.Println(slice)
```
结果：
```
[10 20 35 40 50]
```

<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/go/edit_slice.png">

append 在容量足够的时候也会修改底层数组的元素
``` go 
// 创建一个整型切片
// 其长度和容量都是5个元素
slice := []int{10, 20, 30, 40, 50}
// 创建一个新切片
// 其长度为2个元素，容量为4个元素newSlice := slice[1:3]
newSlice := slice[1:3]
// 使用原有的容量来分配一个新元素
// 将新元素赋值为60
newSlice = append(newSlice, 60)
fmt.Println(slice)
```
结果：
```
[10 20 30 60 50]
```

<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/go/append_slice_no_grow.png">

如果切片的容量不足时，则会创建一个新的数组，所以在创建切片时，可以限制切片的容量，迫使创建一个新的数组，从而为底层数组提供一定的保护。
``` go
// 创建一个整型切片
// 其长度和容量都是5个元素
slice := []int{10, 20, 30, 40, 50}
// 创建一个新切片
// 其长度和容量为2个元素newSlice := slice[1:3:3]
newSlice := slice[1:3:3]
// 将新元素赋值为60，将创建一个新的数组
newSlice = append(newSlice, 60)
```
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/go/append_slice_grow.png">

#### for-range
当迭代切片时，关键字range 会返回两个值。第一个是索引，第二个值是该位置对应元素的副本，而不是引用（也就意味着会发生拷贝）
``` go
// 创建一个整型切片
// 其长度和容量都是 4 个元素
slice := []int{10, 20, 30, 40}
// 迭代每个元素，并显示值和地址
for index, value := range slice {
	fmt.Printf("Value: %d Value-Addr: %X ElemAddr: %X\n", value, &value, &slice[index])
}
```
结果：
```
Value: 10 Value-Addr: C0000160F0 ElemAddr: C000014280
Value: 20 Value-Addr: C0000160F0 ElemAddr: C000014288
Value: 30 Value-Addr: C0000160F0 ElemAddr: C000014290
Value: 40 Value-Addr: C0000160F0 ElemAddr: C000014298
```
性能测试：
``` go
type Data_1K [1024]byte
type Data_1M [1024 * 1024]byte

func BenchmarkRangeSlice(b *testing.B) {
	const length = 10
	b.Run("index_1K", func(b *testing.B) {
		slice := make([]Data_1K, length)
		for i := 0; i < b.N; i++ {
			for i, _ := range slice {
				_ = slice[i]
			}
		}
	})
	b.Run("value_1K", func(b *testing.B) {
		slice := make([]Data_1K, length)
		for i := 0; i < b.N; i++ {
			for _, v := range slice {
				_ = v
			}
		}
	})
	b.Run("index_1M", func(b *testing.B) {
		slice := make([]Data_1M, length)
		for i := 0; i < b.N; i++ {
			for i, _ := range slice {
				_ = slice[i]
			}
		}
	})
	b.Run("value_1M", func(b *testing.B) {
		slice := make([]Data_1M, length)
		for i := 0; i < b.N; i++ {
			for _, v := range slice {
				_ = v
			}
		}
	})
}
```
测试结果：
```
BenchmarkRangeSlice/index_1K-4         	200000000	         9.38 ns/op	       0 B/op	       0 allocs/op
BenchmarkRangeSlice/value_1K-4         	 5000000	       321 ns/op	       0 B/op	       0 allocs/op
BenchmarkRangeSlice/index_1M-4         	200000000	         9.25 ns/op	       0 B/op	       0 allocs/op
BenchmarkRangeSlice/value_1M-4         	    2000	    997077 ns/op	    5242 B/op	       0 allocs/op
```
结论：
1. 当想修改切片内容需要用index 访问切片
2. 当切片的元素比较大时，建议用index 访问切片

#### nil切片==空切片？
``` go 
// nil切片 var nilSlice []T
// 空切片 emptySlice := make([]T, 0)  or emptySlice := []T{}
var nilSlice []int
emptySlice := make([]int, 0)
fmt.Println(nilSlice, len(nilSlice), cap(nilSlice))
fmt.Println(emptySlice, len(emptySlice), cap(emptySlice))
```
结果:
```
[] 0 0
[] 0 0
```
从上面的输出结果来看，这两种切片似乎是一样的，没什么区别。但实际上是有区别的，具体区别在哪呢？
``` go 
nilSlicePtr := *(*[3]int)(unsafe.Pointer(&nilSlice))
emptySlicePtr := *(*[3]int)(unsafe.Pointer(&emptySlice))
fmt.Println(nilSlicePtr)
fmt.Println(emptySlicePtr)
```
结果：
```
[0 0 0]
[5857408 0 0]
```
分析：
1. 我们通过unsafe.Pointer将slice转换成[3]int，切片的内部结构是由3部分组成，大小是3个机器字大小，所以可以将slice的结构看成是长度为3的整型数组[3]int。
1. 空切片不空：包含了一个底层数组，只不过这个数组没有元素。
1. 所有空切片的底层数组都会共享同一个地址，这个地址是指向的值为0，从源码可以看到它的定义

``` go 
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	...
	return mallocgc(mem, et, true)
}
// base address for all 0-byte allocations
var zerobase uintptr

func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if size == 0 {
		return unsafe.Pointer(&zerobase)
	}
	...
}
```
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/go/nil_empty_slice.png">
不管是nil切片还是空切片，对其调用内置函数append、len 和cap 效果都是一样的。但是它们所表示的含义完全不一样：
* nil切片描述的是一个不存在的切片，比如说要求一个函数返回切片但是发生异常时
* 空切片通常描述的一个空集合，比如说当查询数据库返回0个结果时
注：可以对nil切片进行append，但是不能对nil map赋值

另外，nil切片和空切片在判断nil 和json 的序列化出来的结果也不一样
``` go 
fmt.Println(nilSlice == nil) 		// true
fmt.Println(emptySlice == nil)		// false

nilSliceJson, _ := json.Marshal(nilSlice)
emptySliceJson, _ := json.Marshal(emptySlice)
fmt.Println(string(nilSliceJson))	// null
fmt.Println(string(emptySliceJson))	// []
```
### 总结
slice 是一个使用非常方便的数据结构，源码虽然只有短短的200多行，但在设计上却体现了很多优秀的思想，比如说空间预分配避免频繁分配内存、用位运算（加减法）替换乘除法提高效率、共享底层数据较少内存拷贝。当然，slice也有一些问题，在平常使用过程中需要多加注意，避免采坑。

