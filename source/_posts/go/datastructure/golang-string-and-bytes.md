---
title: 深入浅出 Go 语言中的字符串与字节数组
date: 2024-12-10 23:16:31
tags:
- Go
- String
- Bytes
---

# 基础知识

## `byte` 类型

在 `Go` 语言中，`byte` 是 `uint8` 的别名，用来区分字节和 `8` 位无符号整数。

```Go
// src/builtin/builtin.go

// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8
```

我们可以把 `byte` 类型的变量看做是 `C` 语言中的 `char` 类型，用来表示一个 `ASCII` 字符：

```Go
package main

import "fmt"

func main() {
    var a1 byte = 'a'
    fmt.Printf("a1 = %c\n", a1)
    
    var a2 byte = 97
    fmt.Printf("a2 = %c\n", a2)
    
    var a3 byte = 0x61
    fmt.Printf("a3 = %c\n", a3)
    
    var a4 byte = '\x61'
    fmt.Printf("a4 = %c\n", a4)
}
```

显然，上述结果都是 `a`。

## `[]byte` 类型

`[]byte` 类型是 `Go` 语言中用来表示字节数组的类型，更确切地表述方式应该是“一个 `byte` 类型的切片”。

切片的底层表示是一个结构体，本质上是一个数组：

```Go
// src/runtime/slice.go

type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

从 `slice` 结构体可知，切片内维护了一个数组指针，当发生扩缩容时，这个指针指向的底层数组会发生改变。因此数组的长度是固定的，但切片的长度是可变的。

问题：怎么证明一个 `[]byte` 变量在底层是用切片类型来表示的呢？

## `string` 类型

在 `Go` 语言中，`string` 是字符串类型，它是一个不可变的字节序列。

```Go
// src/builtin/builtin.go

// string is the set of all strings of 8-bit bytes, conventionally but not
// necessarily representing UTF-8-encoded text. A string may be empty, but
// not nil. Values of string type are immutable.
type string string
```

字符串的底层表示是一个结构体，本质上是一个 `byte` 数组：

```Go
// src/runtime/string.go

type stringStruct struct {
	str unsafe.Pointer
	len int
}
```

在实例化一个 `string` 变量时，`Go` 语言会调用如下方法：

```Go
//go:nosplit
func findnull(s *byte) int {
	if s == nil {
		return 0
	}

	...

	// pageSize is the unit we scan at a time looking for NULL.
	// It must be the minimum page size for any architecture Go
	// runs on. It's okay (just a minor performance loss) if the
	// actual system page size is larger than this value.
	const pageSize = 4096

	offset := 0
	ptr := unsafe.Pointer(s)
	// IndexByteString uses wide reads, so we need to be careful
	// with page boundaries. Call IndexByteString on
	// [ptr, endOfPage) interval.
	safeLen := int(pageSize - uintptr(ptr)%pageSize)

	for {
		t := *(*string)(unsafe.Pointer(&stringStruct{ptr, safeLen}))
		// Check one page at a time.
		if i := bytealg.IndexByteString(t, 0); i != -1 {
			return offset + i
		}
		// Move to next page
		ptr = unsafe.Pointer(uintptr(ptr) + uintptr(safeLen))
		offset += safeLen
		safeLen = pageSize
	}
}

//go:nosplit
func gostringnocopy(str *byte) string {
	ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)}
	s := *(*string)(unsafe.Pointer(&ss))
	return s
}
```

## `string` 类型和 `[]byte` 类型区别

由上述分析可知，`string` 类型底层本质上就是一个 `byte` 数组，那 `string` 类型为什么要在 `byte` 数组的基础上再封装一层呢？因为 `byte` 数组本身是支持修改的，在并发读写的场景下需要加锁保证安全，而 `string` 类型在上层被设计为是不可变的，进而能够做到在不加锁的情况下实现并发安全。

`[]byte` 类型底层本质上也是一个 `byte` 数组，但是具有 `slice` 类型的特性，因此可以直接对其进行修改。

# 进阶知识

## `string` 类型和 `[]byte` 类型标准转换

`Go` 语言提供了标准方式对 `string` 类型和 `[]byte` 类型进行转换。

### 从 `string` 类型转换到 `[]byte` 类型

```Go
package main

func main() {
    s := "hello world"
    b := []byte(s)
    println(b)
}
```

`string` 到 `[]byte` 类型的标准转换非常简单，那 `Go` 语言底层是如何做的呢？尝试对上述代码进行编译：

```Shell
# -N
# -l    disable inlining
# -S    print assembly listing
go tool compile -N -l -S main.go
```

在 `Windows` 平台下，编译结果如下：

```
0x0023 00035 (D:/workspace/github.com/yobol/assembly-study/go/string_to_bytes/main.go:5)        JMP     37
0x0025 00037 (D:/workspace/github.com/yobol/assembly-study/go/string_to_bytes/main.go:5)        JMP     39
0x0027 00039 (D:/workspace/github.com/yobol/assembly-study/go/string_to_bytes/main.go:5)        LEAQ    go:string."hello world"(SB), AX
0x002e 00046 (D:/workspace/github.com/yobol/assembly-study/go/string_to_bytes/main.go:5)        MOVQ    AX, main.b+40(SP)
0x0033 00051 (D:/workspace/github.com/yobol/assembly-study/go/string_to_bytes/main.go:5)        MOVQ    $11, main.b+48(SP)        
0x003c 00060 (D:/workspace/github.com/yobol/assembly-study/go/string_to_bytes/main.go:5)        MOVQ    $11, main.b+56(SP)
```

未得到预期的结果，待进一步补充。不过可以从源码中找到相应实现：

```Go
// src/runtime/string.go

func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
	}
	copy(b, s)
	return b
}

// rawbyteslice allocates a new byte slice. The byte slice is not zeroed.
func rawbyteslice(size int) (b []byte) {
	cap := roundupsize(uintptr(size), true)
	p := mallocgc(cap, nil, false)
	if cap != uintptr(size) {
		memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size))
	}

	*(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
	return
}
```

可以看到，`string` 类型到 `[]byte` 类型的转换是本质上是通过 `copy` 方法实现的。

### 从 `[]byte` 类型转换到 `string` 类型

```Go
package main

func main() {
    b := []byte{'h', 'e', 'l', 'l', 'o', ' ', 'w', 'o', 'r', 'l', 'd'}
    s := string(b)
}
```

再次回到 `src/runtime/string.go` 源码中，找到 `[]byte` 类型到 `string` 类型的转换实现：

```Go
// src/runtime/string.go

// slicebytetostring converts a byte slice to a string.
// It is inserted by the compiler into generated code.
// ptr is a pointer to the first element of the slice;
// n is the length of the slice.
// Buf is a fixed-size buffer for the result,
// it is not nil if the result does not escape.
//
// slicebytetostring should be an internal detail,
// but widely used packages access it using linkname.
// Notable members of the hall of shame include:
//   - github.com/cloudwego/frugal
//
// Do not remove or change the type signature.
// See go.dev/issue/67401.
//
//go:linkname slicebytetostring
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) string {
	if n == 0 {
		// Turns out to be a relatively common case.
		// Consider that you want to parse out data between parens in "foo()bar",
		// you find the indices and convert the subslice to string.
		return ""
	}
	
    ...

	if n == 1 {
		p := unsafe.Pointer(&staticuint64s[*ptr])
		if goarch.BigEndian {
			p = add(p, 7)
		}
		return unsafe.String((*byte)(p), 1)
	}

	var p unsafe.Pointer
	if buf != nil && n <= len(buf) {
		p = unsafe.Pointer(buf)
	} else {
		p = mallocgc(uintptr(n), nil, false)
	}
	memmove(p, unsafe.Pointer(ptr), uintptr(n))
	return unsafe.String((*byte)(p), n)
}
```

可以看到，`[]byte` 类型到 `string` 类型的转换是通过 `memmove` 方法实现的。

总结，标准转换方式本质上都是需要进行内存拷贝的，所以性能上有一定损耗，这样做的好处也显而易见，就是可以最大程度保证内存安全。

## `string` 类型和 `[]byte` 类型强制转换

根据上述分析，`stringStruct` 和 `slice` 结构字段是非常相似的：

```Go
type stringStruct struct {
	str unsafe.Pointer
	len int
}

type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

两者相比，唯一的区别是 `slice` 结构体最后才会多一个 `cap` 字段，所以内存布局是对齐的，可以直接通过 `unsafe.Pointer` 进行转换。

### 从 `string` 类型转换到 `[]byte` 类型

```Go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	str := "hello world"

	// 这两步可以理解为拿到 string 底层数据结构的指针
	stringPointer := unsafe.Pointer(&str) // unsafe.Pointer 类型等价于 C 语言中的 void* 类型，可以存储任意类型的指针
	stringHeader := (*reflect.StringHeader)(stringPointer)

	// 这两步可以理解为构造一个 slice 的底层数据结构，并返回指针
	sliceHeader := &reflect.SliceHeader{
		Data: stringHeader.Data, // 没有进行内存拷贝，直接使用底层数据结构的指针
		Len:  stringHeader.Len,
		Cap:  stringHeader.Len,
	}
	slicePointer := unsafe.Pointer(sliceHeader)

	bytes := *(*[]byte)(slicePointer)
	fmt.Printf("%#v\n", bytes)
}
```

理解了上述原理后，我们再来看 `Go 1.20` 版本后官方更推荐的强制转换形式：

```Go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	str := "hello world"

	stringPointer := unsafe.StringData(str)
	bytes := unsafe.Slice(stringPointer, len(str))

	fmt.Printf("%#v\n", bytes)

	// bytes[0] = 'H' // 会直接报错 unexpected fault address，为什么呢？
}
```

也可以直接使用：

```Go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	s := "hello world"

	bytes := *(*[]byte)(unsafe.Pointer(&s))

	fmt.Printf("%#v\n", bytes)

	// bytes[0] = 'H' // 会直接报错 unexpected fault address，为什么呢？
}
```

### 从 `[]byte` 类型转换到 `string` 类型

```Go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	bytes := []byte{'h', 'e', 'l', 'l', 'o', ' ', 'w', 'o', 'r', 'l', 'd'}

	// 这两步可以理解为拿到 slice 底层数据结构的指针
	bytesPointer := unsafe.Pointer(&bytes)
	sliceHeader := (*reflect.SliceHeader)(bytesPointer)

	// 这两步可以理解为构造一个 string 的底层数据结构，并返回指针
	stringHeader := &reflect.StringHeader{
		Data: sliceHeader.Data,
		Len:  sliceHeader.Len,
	}
	stringPointer := unsafe.Pointer(stringHeader)

	str := *(*string)(stringPointer)
	fmt.Printf("%#v\n", str)
}
```

理解了上述原理后，我们再来看 `Go 1.20` 版本后官方更推荐的强制转换形式：

```Go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	bytes := []byte{'h', 'e', 'l', 'l', 'o', ' ', 'w', 'o', 'r', 'l', 'd'}

	bytesPointer := unsafe.SliceData(bytes)
	str := unsafe.String(bytesPointer, len(bytes))

	fmt.Printf("%#v\n", str)
}
```

也可以直接使用：

```Go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	bytes := []byte{'h', 'e', 'l', 'l', 'o', ' ', 'w', 'o', 'r', 'l', 'd'}

	str := *(*string)(unsafe.Pointer(&bytes))

	fmt.Printf("%#v\n", str)
}
```

# 总结

`string` 之所以被设计为不可修改的，是因为在并发读写的场景下，不用加锁就可以实现并发安全。即使将 `string` 强制转换为 `[]byte`，由于 `string` 的不可变性，所以其结果也不能被修改。

在不确定是否存在安全隐患的情况下，尽量采用标准方式进行数据转换，以保证代码的健壮性。当然，在某些场景下，已经确定对数据只有只读需求，且存在频繁转换的情况，也可以采用强制转换的方式进行优化。