---
title: 'Unsafe Go'
date: 2021-06-15T13:30:28+08:00
draft: false
toc: true
categories:
  - Golang
tags:
  - unsafe
---

Go 语言 `unsafe` 包的 API 非常简单，只有 Alignof, Offsetof, Sizeof 和 Pointer，却能写出一些性能强悍的代码（文章最后会介绍 Gin 框架中利用 `unsafe.Pointer` 实现 `[]byte` 和 `string` 互相转换的例子）

## Sizeof

```go
func Sizeof(v ArbitraryType) uintptr
```

Sizeof 返回 v 所占用的字节大小。

```go
func ExampleSizeof() {

	var b byte
	var i16 int16
	var i32 int32
	var i64 int64

	fmt.Println(unsafe.Sizeof(b))
	fmt.Println(unsafe.Sizeof(i16))
	fmt.Println(unsafe.Sizeof(i32))
	fmt.Println(unsafe.Sizeof(i64))

	fmt.Println(unsafe.Sizeof(struct {
		x int64
		y int64
	}{}))

	// Output:
	// 1
	// 2
	// 4
	// 8
	// 16
}
```

## Alignof

```go
func Alignof(v ArbitraryType) uintptr
```

Alignof 返回 v 的对齐值。普通类型对齐值为 min(默认对齐值，类型大小 Sizeof 长度)，结构体对齐值为 min(默认对齐值，字段最大类型长度)。一般情况下，32 位系统的默认对齐值为 4，64 位操作系统的默认对齐值为 8。

```go
func ExampleAlignof() {

	var b byte
	var i16 int16
	var i32 int32
	var i64 int64

	fmt.Println(unsafe.Alignof(b))
	fmt.Println(unsafe.Alignof(i16))
	fmt.Println(unsafe.Alignof(i32))
	fmt.Println(unsafe.Alignof(i64))

	fmt.Println(unsafe.Alignof(struct {
		x int64
		y int64
	}{}))

	// Output:
	// 1
	// 2
	// 4
	// 8
	// 8
}
```

> 官方语言规范中对 size 和 align 的描述：[Size and alignment guarantees](https://golang.org/ref/spec#Size_and_alignment_guarantees)

## Offsetof

```go
func Offsetof(v ArbitraryType) uintptr
```

Offsetof 返回 v 所代表的结构中字段的偏移。

```go
func ExampleOffsetof() {

	st := struct {
		x int64
		y int64
	}{}

	fmt.Println(unsafe.Offsetof(st.x))
	fmt.Println(unsafe.Offsetof(st.y))

	// Output:
	// 0
	// 8
}
```

## 内存对齐

在上面的例子中，我刻意避免了内存对齐对结果的影响，但是看接下来的例子：

```go
func ExampleMemAlign() {

	st := struct {
		a byte  // 1 byte
		b int32 // 4 byte
		c byte  // 1 byte
		d int64 // 8 byte
	}{}

	fmt.Println("size =", unsafe.Sizeof(st))
	fmt.Println("align =", unsafe.Alignof(st))

	fmt.Println("a.size =", unsafe.Sizeof(st.a))
	fmt.Println("a.align =", unsafe.Alignof(st.a))
	fmt.Println("a.offset =", unsafe.Offsetof(st.a))

	fmt.Println("b.size =", unsafe.Sizeof(st.b))
	fmt.Println("b.align =", unsafe.Alignof(st.b))
	fmt.Println("b.offset =", unsafe.Offsetof(st.b))

	fmt.Println("c.size =", unsafe.Sizeof(st.c))
	fmt.Println("c.align =", unsafe.Alignof(st.c))
	fmt.Println("c.offset =", unsafe.Offsetof(st.c))

	fmt.Println("d.size =", unsafe.Sizeof(st.d))
	fmt.Println("d.align =", unsafe.Alignof(st.d))
	fmt.Println("d.offset =", unsafe.Offsetof(st.d))

	// Output:
	// size = 24
	// align = 8
	// a.size = 1
	// a.align = 1
	// a.offset = 0
	// b.size = 4
	// b.align = 4
	// b.offset = 4
	// c.size = 1
	// c.align = 1
	// c.offset = 8
	// d.size = 8
	// d.align = 8
	// d.offset = 16
}
```

从输出结果可以发现结构体的大小并不是简单的 1 + 4 + 1 + 8 = 14

结构体中每个字段的偏移量必须为字段对齐值的整数倍，所以会在部分字段间加上填充。

```
a___bbbb
c_______
dddddddd
```

通过改变字段顺序，结构体的大小可以从 24 降到 16：

```go
func ExampleMemAlign() {

	st := struct {
		a byte  // 1 byte
		c byte  // 1 byte
		b int32 // 4 byte
		d int64 // 8 byte
	}{}

	fmt.Println("size =", unsafe.Sizeof(st))
	fmt.Println("align =", unsafe.Alignof(st))

	// Output:
	// size = 16
	// align = 8
}
```

此时的内存结构为：

```
ac__bbbb
dddddddd
```

所以，**合理的字段顺序可以减少内存的开销**

## Pointer & bytesconv

Pointer 代表一个指向任意类型的指针（比 Go 的普通指针更 "C" 一点）

```go
type Student struct {
	Name string
	Age  int
}

func main() {
	var s Student

	namePtr := (*string)(unsafe.Pointer(&s))
	*namePtr = "mll"

	agePtr := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.Age)))
	*agePtr = 20

	fmt.Println(s)
}
```

掌握了 `unsafe` 包的基本用法后，就可以开始尝试用它写一些 “高性能” 的代码了（**最好还是别滥用**）。一个比较简单的例子就是 [Gin](https://github.com/gin-gonic/gin) 的内部包中的 `bytesconv` ，虽然 Go 已经提供了 `[]byte("...")` 和 `string(...)`，但是 `bytesconv` 在基准测试中表现的更好。

> [Gin - internal/bytesconv/bytesconv.go](https://github.com/gin-gonic/gin/blob/master/internal/bytesconv/bytesconv.go)

```go
func StringToBytes(s string) []byte {
	return *(*[]byte)(unsafe.Pointer(
		&struct {
			string
			cap int
		}{s, len(s)},
	))
}

func BytesToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

只看代码中的几个指针强转，可能一头雾水。我们可以用 gdb 打印出 `string` 和 `[]byte` 的真实结构：

```bash
$ go build -o main -gcflags=all="-N -l" main.go # 关闭优化和内联，方便调试
$ gdb ./main
(gdb) l
1       package main
2
3       import "fmt"
4
5       func main() {
6               s := "hello"
7               b := []byte("hello")
8               fmt.Println(s, b)
9       }
(gdb) b 8
Breakpoint 1 at 0x4b6991: file /home/mll/go/src/algo/main.go, line 8.
(gdb) r
Starting program: /home/mll/go/src/algo/main
[New LWP 53002]
[New LWP 53003]
[New LWP 53004]
[New LWP 53005]

Thread 1 "main" hit Breakpoint 1, main.main () at /home/mll/go/src/algo/main.go:8
8               fmt.Println(s, b)
(gdb) ptype s
type = struct string {
    uint8 *str;
    int len;
}
(gdb) ptype b
type = struct []uint8 {
    uint8 *array;
    int len;
    int cap;
}
```

它们真实的结构，只相差了 `cap` 字段，所以 `string` 转 `[]byte` 需要添加上 `cap`，而 `[]byte` 可以直接强转成 `string`。

[benchmark](https://github.com/gin-gonic/gin/blob/master/internal/bytesconv/bytesconv_test.go) 对比：

```bash
$ go test -bench=^BenchmarkBytesConv -benchmem
goos: linux
goarch: amd64
pkg: algo
cpu: Intel(R) Core(TM) i7-7500U CPU @ 2.70GHz
BenchmarkBytesConvBytesToStrRaw-4       29838418                35.77 ns/op           96 B/op          1 allocs/op
BenchmarkBytesConvBytesToStr-4          1000000000               0.2945 ns/op          0 B/op          0 allocs/op
BenchmarkBytesConvStrToBytesRaw-4       26174215                51.52 ns/op           96 B/op          1 allocs/op
BenchmarkBytesConvStrToBytes-4          1000000000               0.4026 ns/op          0 B/op          0 allocs/op
PASS
ok      algo    3.282s
```
