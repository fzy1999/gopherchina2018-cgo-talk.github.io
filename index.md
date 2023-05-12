

<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->

<!-- *** 横向分隔, --- 竖向分隔, Note: 讲稿注释  -->

<!--


Reveal.js 可能会需要 AJAX 异步加载 Markdown 文件, 可以在当前目录启动一个 http 服务.

以下是常见的临时启动 http 服务器的方式:

	NodeJS
	npm install http-server -g
	http-server
	
	Python2
	python -m SimpleHTTPServer
	
	Python3
	python -m http.server
	
	Golang
	go run server.go

启动后, 本地可以访问 http://127.0.0.1:port, 其中 port 为端口号, 命令行有提示.

幻灯片操作: F键全屏, S键显示注解, ESC大纲模式, ESC退出全屏或大纲模式, ?显示帮助

-->

<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->

<!-- <section data-background="images/gopherchina2018-background.jpg"> -->

# CGO编程 <!-- .element: style="color:DarkSlateGray;" -->
------------

#### 冯昭宇 <!-- .element: style="color:DarkSlateGray;" -->


<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->
***

## 参考资料
------------------
- [《Go语言高级编程》](https://github.com/chai2010/advanced-go-programming-book) 
- [GopherChina 2018报告](https://github.com/chai2010/gopherchina2018-cgo-talk)
- 《程序员的自我修养：链接、装载与库》
- 《C++Primer Plus》
- #### https://golang.org/cmd/cgo/
- #### https://blog.golang.org/c-go-cgo
- #### https://github.com/golang/go/wiki/cgo
- #### https://golang.org/src/runtime/cgocall.go
- #### https://golang.org/misc/cgo/test/

---
## 既不太会C
## 也不太会Go
------------------------
### 若有不对或问题，欢迎中断指出

***

## 内容大纲
----------

- CGO的价值

--------

- 快速入门
- 类型转换
- 函数调用
- Go访问C++对象, Go对象导出为C++对象
- 编译和链接参数
- 编译与链接

--------

<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->
***

## CGO的价值
--------------

1. 没有银弹, Go语言也不是银弹, 无法解决全部问题
2. 通过CGO可以继承C/C++的软件遗产

            比如IEDA软件中的“iDB”模块
3. 通过CGO可以用Go给其它系统写C接口的共享库
4. CGO是Go和其它语言直接通讯的桥梁


<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->
***


## 快速入门
---------

```go
import "C"

func main() {
	println("hello cgo")
}
```

-----

- `import "C"` 表示启用 CGO
- 最简单的CGO程序

---
## 快速入门
---------

examples/hello-v1/main.go:

```go
package main

//#include <stdio.h>
import "C"

func main() {
	C.puts(C.CString("你好, GopherChina 2018!\n"))
}
```

-----

编译运行:

```
$ go run examples/hello-v1/main.go
你好, GopherChina 2018!
```

---

### 简单说明
----------

- `import "C"` 表示启用 CGO
- `import "C"` 前的注释表示包含C头文件: `<stdio.h>`
- `C.CString` 表示将 Go 字符串转为 C 字符串
- `C.puts` 调用C语言的puts函数输出 C 字符串

Note: 小问题: C 字符串什么时候释放?

---
### 调用自定义的C函数
-----------------

examples/hello-v2/main.go:

```go
package main

/*
#include <stdio.h>

static void SayHello(const char* s) {
	puts(s);
}
*/
import "C"

func main() {
	C.SayHello(C.CString("Hello, World\n"))
}
```

------------

---
### C代码模块化
------------

examples/hello-v3/hello.h:

```c
extern void SayHello(const char* s);
```

examples/hello-v3/hello.c:

```c
#include "hello.h"

#include <stdio.h>

void SayHello(const char* s) {
	puts(s);
}
```

examples/hello-v3/main.go:

```go
//#include "./hello.h"
import "C"
```

----------

---
### C代码模块化 - 改用Go重写C模块
----------------------------

examples/hello-v4/hello.h:

```c
extern void SayHello(/* const */ char* s);
```

examples/hello-v4/hello.go:

```go
package main
import "C"
import "fmt"
//export SayHello
func SayHello(s *C.char) {
	fmt.Print(C.GoString(s))
}
```

------
- hello.c => hello.go
- SayHello 函数被声明为 //export，遵循C语言的ABI（Application Binary Interface）

---
### 手中无剑, 心中有剑
-------------------

examples/hello-v5/hello.go:

```go
package main

// extern void SayHello(char* s);
import "C"
import "fmt"

func main() {
	C.SayHello(C.CString("Hello, World\n"))
}

//export SayHello
func SayHello(s *C.char) {
	fmt.Print(C.GoString(s))
}
```

------

- C 语言版本 SayHello 函数实现只存在于心中
- 面向纯 C 接口的 Go 语言编程


---
### 忘掉心中之剑
--------------

```go
// +build go1.10

package main

// extern void SayHello(_GoString_ s);
import "C"
import "fmt"

func main() {
	C.SayHello("Hello, World\n")
}

//export SayHello
func SayHello(s string) {
	fmt.Print(s)
}
```

----------

- GoString 也是一种 C 字符串
- Go的一切都可以从C理解


Note:
为何要引入 `C._GoString_` 类型?
为何不使用 `C.GoString` 类型?


---


### 思考题: SayHello还在主Goroutine吗
---------

```go

func main() {
	C.SayHello("Hello, World\n")
}

//export SayHello
func SayHello(s string) {
	fmt.Print(s)
}
```

---------

- 具体分析请参考稍后的 **内存模型** 一节

<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->
***

## 类型转换
----------

- 指针 - unsafe 包的灵魂
- Go字符串和切片的结构

--------

- Go指针和C指针之间的转换
- 数值和指针之间的转换
- 不同类型指针转换
- 字符串和切片转换
- ...


---
### 指针 - unsafe 包的灵魂
---------------------------

Go版无类型指针和数值化的指针:

```go
var p unsafe.Pointer = nil        // unsafe
var q uintptr        = uintptr(p) // builtin
```

C版无类型指针和数值化的指针:

```c
void     *p = NULL;
uintptr_t q = (uintptr_t)(p); // <stdint.h>
```

-----------

- `unsafe.Pointer` 是 Go指针 和 C指针 转换的中介
- `uintptr` 是 Go 中 数值 和 指针 转换的中介

---
### unsafe 包
------------

```go
type ArbitraryType int
type Pointer *ArbitraryType

func Sizeof(x ArbitraryType) uintptr
func Alignof(x ArbitraryType) uintptr

func Offsetof(x ArbitraryType) uintptr
```

-------

- Pointer: 面向编译器无法保证安全的指针类型转换
- Sizeof: 值所对应变量在内存中的大小
- Alignof: 值所对应变量在内存中地址几个字节对齐
- Offsetof: 结构体中成员的偏移量

---
### unsafe 包 - C语言版本
----------------------

```c
typedef void* Pointer;

sizeof(type or expression); // C
offsetof(type, member);     // <stddef.h>
alignof(type-id);           // C++ 11
```

-------

- C指针的安全性永远需要自己负责
- sizeof 是关键字, 语义和 Go 基本一致
- offsetof 是宏, 展开为表达式, 语义和 Go 基本一致
- alignof 是新特性, 可忽略

---
### Go字符串和切片的结构
--------------------

```go
type reflect.StringHeader struct {
	Data uintptr
	Len  int
}
type reflect.SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

```c
typedef struct { const char *p; GoInt n; } GoString;
typedef struct { void *data; GoInt len; GoInt cap; } GoSlice;
```

-------

- reflect 包定义的结构和CGO生成的C结构是一致的
- GoString 和 GoSlice 和头部结构是兼容的


---
### 实战: `int32` 和 `*C.char` 相互转换(A)
------------------------------------

#### ![](images/int32-to-char-ptr.uml.png) <!-- .element: style="width:75%;" -->


---
### 实战: `int32` 和 `*C.char` 相互转换(B)
------------------------------------

```go
// int32 => *C.char
var x = int32(9527)
var p *C.char = (*C.char)(unsafe.Pointer(uintptr(x)))

// *C.char => int32
var y *C.char
var q int32 = int32(uintptr(unsafe.Pointer(y)))
```
-----

1. 第一步: `int32` => `uintprt`
1. 第二步: `uintptr` => `unsafe.Pointer`
1. 第三步: `unsafe.Pointer` => `*C.char`
1. 反之亦然


---
### 实战: `*X` 和 `*Y` 相互转换(A)
----------------------------

#### ![](images/x-ptr-to-y-ptr.uml.png) <!-- .element: style="width:60%;" -->


---
### 实战: `*X` 和 `*Y` 相互转换(B)
----------------------------

```go
var p *X
var q *Y

q = (*Y)(unsafe.Pointer(p)) // *X => *Y
p = (*X)(unsafe.Pointer(q)) // *Y => *X
```
-----

1. 第一步: `*X` => `unsafe.Pointer`
1. 第二步: `unsafe.Pointer` => `*Y`
1. 反之亦然

---
### 实战: `[]X` 和 `[]Y` 相互转换(A)
------------------------------------

#### ![](images/x-slice-to-y-slice.uml.png) <!-- .element: style="width:60%;" -->

---
### 实战: `[]X` 和 `[]Y` 相互转换(B)
------------------------------------

```go
var p []X
var q []Y // q = p

pHdr := (*reflect.SliceHeader)(unsafe.Pointer(&p))
qHdr := (*reflect.SliceHeader)(unsafe.Pointer(&q))

pHdr.Data = qHdr.Data
pHdr.Len = qHdr.Len * unsafe.Sizeof(q[0]) / unsafe.Sizeof(p[0])
pHdr.Cap = qHdr.Cap * unsafe.Sizeof(q[0]) / unsafe.Sizeof(p[0])
```

------

- 所有切片拥有相同的头部 `reflect.SliceHeader`
- 重新构造切片头部即可完成转换

---
### 示例: float64 数组排序优化
--------------------------

```go
func main() {
	// []float64 强制类型转换为 []int
	var a = []float64{4, 2, 5, 7, 2, 1, 88, 1}
	var b []int = ((*[1 << 20]int)(unsafe.Pointer(&a[0])))[:len(a):cap(a)]

	// 以int方式给float64排序
	sort.Ints(b)
}
```

-----

- float64遵循IEEE754浮点数标准特性
- 当浮点数有序时对应的整数也必然是有序的


<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->
***

## 函数调用
----------

- Go调用C函数
- C调用Go导出函数
- 深度调用: Go => C => Go => C

---
### Go调用C函数(A)
----------------

```go
/*
static int add(int a, int b) {
	return a+b;
}
*/
import "C"

func main() {
	C.add(1, 1)
}
```
----------------

- `C.add` 通过的 C 虚拟包访问
- 最终会转为 `_Cfunc_add` 名字


---
### Go调用C函数(B)
----------------

```go
/*
static int add(int a, int b) {
	return a+b;
}
*/
import "C"
import "fmt"

func main() {
	v, err := C.add(1, 1)
	fmt.Println(v, err)

	// Output:
	// 4 <nil>
}
```
----------------

- 任何C函数都可以带2个返回值
- 第二个返回值是 `errno`, 对应 `error` 接口类型

---
### Go调用C函数(C)
----------------

```go
/*
#include <errno.h>

static void seterrno(int v) {
	errno = v;
}
*/
import "C"
import "fmt"

func main() {
	_, err := C.seterrno(9527)
	fmt.Println(err)

	// Output:
	// errno 9527
}
```
----------------

- 即使没有返回值, 依然可以通过第二个返回值获取 errno
- 对应 void 类型函数, 第一个返回值可以用 `_` 占位


---
### Go调用C函数(D)
----------------

```go
// static void noreturn() {}
import "C"
import "fmt"

func main() {
	x, _ := C.noreturn()
	fmt.Printf("%#v\n", x)

	// Output:
	// main._Ctype_void{}
}
```
----------------

- 甚至可以获取一个 void 类型函数的返回值
- 返回值类型: `type _Ctype_void [0]byte`

<!-- 不同包都cgo类型是不通用的, 因为是私有类型 -->

---
### 导出Go函数(A)
---------------

main.go:

```go
import "C"

//export GoAdd
func GoAdd(a, b C.int) C.int {
	return a+b
}
```
-------------

- 可以导出私有函数
- 导出C函数名没有名字空间约束, 需保证全局没有重名
- main 包的导出函数会在 `_cgo_export.h` 声明

---
### 导出Go函数(B)
---------------

add.h:

```h
int c_add(int a, int b);
```

add.c:

```c
#include "add.h"
#include "_cgo_export.h"

int c_add(int a, int b) {
	return GoAdd(a, b)
}
```

---------------

- 在C文件中使用 `_cgo_export.h` 头文件
- C文件必须在同一个包, 否则会找不到头文件

---
### 导出Go函数(C)
---------------

main.go:

```go
//export int GoAdd(int a, int b);
//#include "add.h"
import "C"

func main() {
	C.GoAdd(1, 1)
	C.c_add(2, 2)
}
```
---------------

- 无法在Go文件引用导出头文件, 因为还未生成
- GoAdd 是Go导出函数, 无法通过  `_cgo_export.h` 引用
- c_add 是C定义函数, 可以通过 `add.h` 头文件引用
- 可手写函数声明, 不会形成循环依赖

---
### 导出Go函数(D)
---------------

```go
// extern void SayHello(GoString s); // GoString 在哪定义?
import "C"

//export SayHello
func SayHello(s string) {
	fmt.Print(s)
}
```
---------------

- 导出函数的参数是Go字符串
- C类型为 `GoString`, 在 `_cgo_export.h` 文件定义
- 要使用 `GoString` 类型就要引用  `_cgo_export.h` 文件
- 这时候该如何手写 SayHello 函数的声明?

---
### 导出Go函数(E)
---------------

```go
// +build go1.10

// extern void SayHello(_GoString_ s);
import "C"

//export SayHello
func SayHello(s string) {
	fmt.Print(s)
}
```
---------------

- Go 1.10 增加了 `_GoString_` 类型
- `_GoString_` 是预定义的类型, 和 `GoString` 等价
- 避免手写函数声明时出现循环依赖


---
### 深度调用: Go => C => Go => C (A)
-------------------------------

```go
/*
static int c_add(int a, int b) {
	return a+b;
}

static int go_add_proxy(int a, int b) {
	extern int GoAdd(int a, int b);
	return GoAdd(a, b);
}
*/
import "C"
```
----------------

- `go_add_proxy` 调用Go导出的 `GoAdd`


---
### 深度调用: Go => C => Go => C (B)
-------------------------------

```go
func main() {
	C.c_add(1, 1)
}

//export GoAdd
func GoAdd(a, b C.int) C.int {
	return a + b
}
```
----------------

- Go:`main` => C:`go_add_proxy` => Go:`GoAdd`


<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->


***

## 内存模型
----------

- 内存是如何对齐
- 内存是如何布局的(堆/栈)
- GC/动态栈有何影响
- 顺序一致性内存模型
- CGO指针的使用原则
- 逃逸分析是银弹吗
- 思考题: SayHello还在主Goroutine吗

------

- 理论武装代码...

---
### 结构体对齐(A)
---------------

- CPU对基础类型有对齐要求, 比如int32要求4字节对齐

-----------
- 推论1: 数组中的每个元素也要对齐
- 推论2: 结构体中的每个成员也要对齐
- 推论3: 结构体数组中每个元素的每个成员都要对齐


---
### 结构体对齐(B)
---------------

#### ![](images/mem-addr-align.png) <!-- .element: style="width:70%;" -->
-----------

- 目前内存的顺序和定义的顺序是一致的
- 为了对齐, 出现了一些占位用的垃圾空间
- 安排适当的定义顺序可能会减少内存缝隙(阅读不友好)
- 在未来, Go或许默认会自动调整内存布局顺序

<!-- 重绘图片 -->


---
### 内存是如何布局的(堆/栈)(A)
--------------------------

- Go中有堆也有栈
- 但是不知道它们在哪里
- 不仅仅如此, 栈还会到处乱窜...

--------------

- 推论1: 无法知道变量是在栈上还是在堆上
- 推论2: 任何变量都可能在内存到处乱窜

--------------

- 以上对Go语言码农是透明的
- 但 cgo 不是 Go


---
### 内存是如何布局的(堆/栈)(B)
--------------------------

```go
func main() {
	p := intptr(getv()) // p 安全吗?
	v := *(*int)(unsafe.Pointer(p))
	println(v)
}
func getv() *int {
	// x 在对上还是栈上?
	var x = 42
	return &x
}
```

------------

- 此时x应该在堆上
- 原因: getv 返回时, x 地址依然有效, 而栈帧已经被销毁

------------

- 既然是在堆上, x 地址不会变化了吧?
- 依然不知道 x 是否会变化 (语言规范没有说不会变)
- intptr 中转的指针依然是不安全的


---
### GC/动态栈有何影响(A)
---------------------

- GC目前(Go1.10)对CGO内存布局没有影响
- GC 可能导致 Go 内存被提前回收

------

- Go语言的栈始终是可以动态伸缩的
- Go1.4前是分段式动态栈实现, 栈内存地址是固定的
- 分段栈的缺点是内存太分散, 导致CPU缓存效果降低
- Go1.4之后, 开始采用连续拷贝栈实现
- 连续拷贝栈栈伸缩时会导致栈内存的变化


---
### GC/动态栈有何影响(B)
---------------------

- GC 导致 Go语言内存生命周期不固定
- 连续拷贝动态栈 导致 内存地址易变

------

- 这是 CGO 编程要小心处理的地方


---
### 顺序一致性内存模型
-------------------

- 同一个Goroutine符合顺序一致性内存模型
- 不同Goroutine不符合顺序一致性内存模型(乱序执行)

------

- 详细内容可以参考我的《Go语言并发编程》报告
- 并发的内存模型对CGO影响不大


---
### CGO指针的使用原则
-------------------

- cgo调用的C函数返回前, 传入的Go内存有效
- cgo调用的C函数返回后, Go内存对C语言失效

------

- C调用的Go函数返回值不得包含Go内存地址(绝对吗?)
- `GODEBUG=cgocheck=0 go run x.go`
- `cgocheck` 可关闭此检查

------

- https://golang.org/issue/12416

---
### 逃逸分析是银弹吗
-----------------

```go
func foo(x int) { return &x }
func bar(x int) { C.foo(&x) }
```

--------

- 函数返回前, 参数x是否还在栈上?
- 函数返回后, 参数x是否还在栈上?

--------

- `C.foo`返回前x应该还是在函数调用的栈帧中
- CGO的指针原则要求`C.foo`返回前x地址不得变化
- 间接地导致 foo 函数的栈被 `C.foo` 调用临时冻结!


---
### 思考题: SayHello还在主Goroutine吗
----------------------------------

```go
func main() {
	C.SayHello("Hello, World\n")
}

//export SayHello
func SayHello(s string) {
	fmt.Print(s)
}
```

----

- 前一节分析, C函数返回前, 将临时冻结Go的动态栈

----

- 如果 `Go.SayHello` 还在主 Goroutine 中,
- 将导致 `Go.SayHello` 不再支持栈的动态伸缩!
- 为了支持动态栈, `SayHello` 必须不在主 Goroutine 中!


---
### C 内存到 Go 内存(A)
---------------------

- C中栈内存不能返回(函数调用返回就被回收)
- C堆中的内存地址是稳定的

--------------

- 可以将C堆内存转为任意其它数值类型, 转回依然在

---
### C 内存到 Go 内存(B)
---------------------

```go
type VoidPointer uintptr

func NewVoidPointerFrom(p unsafe.Pointer) VoidPointer {
	return VoidPointer(p)
}

func Malloc(n int) VoidPointer {
	return VoidPointer(C.malloc(C.size_t(n)))
}
func (p VoidPointer) Free() {
	C.free(unsafe.Pointer(p))
}

func (p VoidPointer) ByteSlice(n int) []byte {
	return ((*[1 << 31]byte)(unsafe.Pointer(p)))[0:n:n]
}
```
------------

- 通过 VoidPointer 包装类型表示C语言指针
- 通过 ByteSlice 等方法将C指针转为Go对应的切片类型
- 为何不直接使用 `unsafe.Pointer` ?


---
### Go内存临时到C内存(A)
---------------------

- cgo规定: Go函数返回指针不能包含Go内存
- cgo规定: 调用C函数, 可以传入Go内存, 返回失效

---------

- 问题: 调用C函数, 回调Go函数返回Go内存, 如何?


---
### Go内存临时到C内存(B) - 01
---------------------

```go
/*
#include <stdio.h>
#include <stdint.h>

extern int* go_get_arr_ptr_v1(int *arr, int idx);

static void print_array_v1(int *arr, int n) {
	int i;
	for(i = 0; i < n; i++) {
		int *p = (int*)(go_get_arr_ptr_v1(arr, i));
		printf("%d ", *p);
	}
	printf("\n");
}
*/
import "C"
```

-------------

- `go_get_arr_ptr_v1` Go实现, 可能返回Go内存


---
### Go内存临时到C内存(B) - 02
---------------------

```go
func main() {
	values := []int32{42, 9, 101, 95}

	// panic: runtime error: cgo result has Go pointer
	C.go_get_arr_ptr_v1(
		(*C.int)(unsafe.Pointer(&values[0])),
		C.int(0),
	)
}

//export go_get_arr_ptr_v1
func go_get_arr_ptr_v1(arr *C.int, idx C.int) *C.int {
	base := uintptr(unsafe.Pointer(arr))
	p := (*C.int)(unsafe.Pointer(base + uintptr(idx)*4))
	return p
}
```

-------------

- `go run main.go`
- `GODEBUG=cgocheck=0 go run main.go`


---
### Go内存临时到C内存(C)
---------------------

```go
//export go_get_arr_ptr_v2
func go_get_arr_ptr_v2(arr *C.int, idx C.int) C.uintptr_t {
	base := C.uintptr_t(uintptr(unsafe.Pointer(arr)))
	p := base + C.uintptr_t(idx)*4
	return p
}
```

-------------

- 返回值避免用指针类型, 避免cgo检测指针
- 这是回调上下文, 返回的Go内存被锁定了

---
### C长期持有Go内存(A)
--------------------

```go
type ObjectId int32

var refs struct {
	sync.Mutex
	objs map[ObjectId]interface{}
	next ObjectId
}

func init() {
	refs.Lock()
	defer refs.Unlock()

	refs.objs = make(map[ObjectId]interface{})
	refs.next = 1000
}
```

------------

- 将Go内存对象映射为int类型的id
- id是稳定的, 不会发生变化

---
### C长期持有Go内存(B)
--------------------

```go
func NewObjectId(obj interface{}) ObjectId {
	refs.Lock()
	defer refs.Unlock()

	id := refs.next
	refs.next++

	refs.objs[id] = obj
	return id
}
func (id ObjectId) Get() interface{} {
	refs.Lock()
	defer refs.Unlock()

	return refs.objs[id]
}
```
------------

- 可以从id解包出真实的Go对象

---
### C长期持有Go内存(C)
--------------------

```go
func (id *ObjectId) Free() interface{} {
	refs.Lock()
	defer refs.Unlock()

	obj := refs.objs[*id]
	delete(refs.objs, *id)
	*id = 0

	return obj
}
```

------------

- 不需要时需要释放id, 避免资源泄露


---
### C长期持有Go内存(D)
--------------------

```go
//extern char* NewGoString(char* s);
//extern void FreeGoString(char* s);
import "C"

//export NewGoString
func NewGoString(s *C.char) *C.char {
	gs := C.GoString(s)
	id := NewObjectId(gs)
	return (*C.char)(unsafe.Pointer(uintptr(id)))
}

//export FreeGoString
func FreeGoString(p *C.char) {
	id := ObjectId(uintptr(unsafe.Pointer(p)))
	id.Free()
}
```

------------

- id是数值类型, 可以当一个伪指针使用
- 类比: C++中的this也可以当作id


<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->
***

## Go访问C++对象
---------

1. 准备一个C++类
2. 去除命名空间
3. C++类转C接口
4. C接口函数到Go接口函数
5. Go接口函数到Go对象

---
### Go访问C++对象: 准备一个C++类
------------------

```c++

namespace ifp {
    class FpApi
    {
     public:
      static FpApi* getInstance();
      
      static void  destroyInst();
      
      // function
      bool initDie(double die_lx, double die_ly, double die_ux, double die_uy);
      
      private:
      	static FpApi* _instance;	
    }
}
```

-------

- 值类型风格的对象(并不推荐)
- 推荐 new/delete 风格

---
### Go访问C++对象: C++中的使用方式
------------------

```c++

int main() {
	ifp::FpApi* obj = ifp::FpApi::getInstance();
    obj.initDie(1,2,3,4);
    obj.destroyInst();
}
```

-----



---
### Go访问C++对象: 想象为C风格接口
------------------
##### C语言版本
```c
int main() {
	ifpFpApi* obj = FpApi_create();
	FpApi_initDie(obj,0.0,0.0, 2843,  2843)
	FpApi_free(obj)
}
```
-----------
##### GO语言版本
```go
import "C"
func main() {
	obj:=C.FpApi_create()
	C.FpApi_initDie(obj,0.0,0.0, 2843,  2843)
	C.FpApi_free(obj)
}
```
-------------


---
### Go访问C++对象: 无命名空间的C++  
------------------

```c++
class ifpFpApi{
    public:
        ifpFpApi(){}
        
        bool ifpinitDie(double die_lx, double die_ly, double die_ux, double die_uy){
            return ifp::FpApi::getInstance()->initDie( die_lx,  die_ly,  die_ux,  die_uy);
        }
        
        ~ifpFpApi(){
        	ifp::FpApi::getInstance()->destroyInst();
        }
};
```

-----

- extern“C”貌似不支持命名空间


---
### Go访问C++对象: C接口实现
------------------

```c
typedef struct ifpFpApi ifpFpApi;

ifpFpApi *FpApi_create(){
    return new ifpFpApi();
}

void FpApi_initDie(ifpFpApi*p, double die_lx, double die_ly, double die_ux, double die_uy){
     p->ifpinitDie( die_lx,  die_ly,  die_ux,  die_uy);
    
}

void FpApi_free(ifpFpApi*p){
    delete p;
    p = 0;
}
```

----------

- 通过 new 创建对象, 避免以值或引用的方式使用对象


---
### Go访问C++对象:Go调用C接口
------------------

```go
# include"ifp_api_wrapperC.h"
*/
import "C"

func main() {

	obj:=C.FpApi_create()
	C.FpApi_initDie(obj,0.0,0.0, 2843,  2843)
	C.FpApi_free(obj)
}
```


<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->
***

### Go对象导出为C++对象
--------------------

1. 准备一个Go对象
1. Go对象映射为一个id
1. Go对象对应的id到Go接口函数
1. Go接口函数到C接口函数
1. C接口函数到C++类
1. id就是this指针

---
### 准备一个Go对象
----------------

```go
type Person struct {
	name string
	age  int
}

func NewPerson(name string, age int) *Person {
	return &Person{name: name, age: age}
}

func (p *Person) Set(name string, age int) {
	p.name, p.age = name, age
}

func (p *Person) Get() (name string, age int) {
	return p.name, p.age
}
```

-----------

- 完全是普通的 Go 对象

---
### 对应的C接口
----------------

```c
// person_capi.h
#include <stdint.h>

typedef uintptr_t person_handle_t;

person_handle_t person_new(char* name, int age);
void person_delete(person_handle_t p);

void person_set(person_handle_t p, char* name, int age);
char* person_get_name(person_handle_t p, char* buf, int size);
int person_get_age(person_handle_t p);
```

--------

- 以 C 接口的方式重新抽象前面的 Go 对象
- `person_handle_t` 类似对象指针, 只是类似
- `uintptr_t` 类似指针, 但不是指针

---
### Go实现C接口函数(A)
----------------

```go
//export person_new
func person_new(name *C.char, age C.int) C.person_handle_t {
	id := NewObjectId(NewPerson(C.GoString(name), int(age)))
	return C.person_handle_t(id)
}

//export person_delete
func person_delete(h C.person_handle_t) {
	ObjectId(h).Free()
}

//export person_set
func person_set(h C.person_handle_t, name *C.char, age C.int) {
	p := ObjectId(h).Get().(*Person)
	p.Set(C.GoString(name), int(age))
}
```

----------

- Go对象指针如何转为 `person_handle_t` ?

---
### Go实现C接口函数(B)
----------------

```go
//export person_get_name
func person_get_name(h C.person_handle_t, buf *C.char, size C.int) *C.char {
	p := ObjectId(h).Get().(*Person)
	name, _ := p.Get()

	n := int(size) - 1
	bufSlice := ((*[1 << 31]byte)(unsafe.Pointer(buf)))[0:n:n]
	n = copy(bufSlice, []byte(name))
	bufSlice[n] = 0

	return buf
}
```

----------

- `person_handle_t` 如何还原 Go 对象指针 ?


---
### Go实现C接口函数(C)
----------------

```go
//export person_get_age
func person_get_age(h C.person_handle_t) C.int {
	p := ObjectId(h).Get().(*Person)
	_, age := p.Get()
	return C.int(age)
}
```
----------

- Go 对象指针移动了会如何 ?


---
### C接口到C++类(A)
----------------

```c
extern "C" {
	#include "./person_capi.h"
}

struct Person {
	person_handle_t goobj_;

	Person(const char* name, int age) {
		this->goobj_ = person_new((char*)name, age);
	}
	~Person() {
		person_delete(this->goobj_);
	}
```

-----------

- C++ 的槽点: 每次包装时都需要一层内存隔离

---
### C接口到C++类(B)
----------------

```c
	void Set(char* name, int age) {
		person_set(this->goobj_, name, age);
	}
	char* GetName(char* buf, int size) {
		return person_get_name(this->goobj_ buf, size);
	}
	int GetAge() {
		return person_get_age(this->goobj_);
	}
}
```

------

- 这是各种 C++ 教程推荐的技术
- 但是我不喜欢!

---
### C接口到C++类(C)
----------------

```c
int main() {
	auto p = new Person("gopher", 10);

	char buf[64];
	char* name = p->GetName(buf, sizeof(buf)-1);
	int age = p->GetAge();

	printf("%s, %d years old.\n", name, age);
	delete p;

	return 0;
}
```

------

- 大家能看出问题在哪里吗?
- 内部的 `person_new` 已经为对象分配了内存
- `new Person` 再分配一次, 只是用于保存对象的指针
- 最大的问题是将指针包装为类, 失去了自由转换的权利

---
### id就是this
----------------

```c
truct Person {
	static Person* New(const char* name, int age) {
		return (Person*)person_new((char*)name, age);
	}
	void Delete() {
		person_delete(person_handle_t(this));
	}

	void Set(char* name, int age) {
		person_set(person_handle_t(this), name, age);
	}
	char* GetName(char* buf, int size) {
		return person_get_name(person_handle_t(this), buf, size);
	}
};
```

---------

- C++ 中的 this 是什么 (用Go的思维)?
- this 就是一个函数参数
- class 是否也是必须的?

---
### 还我自由的指针(Go)
----------------

```go
type Int int

func (p Int) Twice() int {
	return int(p)*2
}
```

```go
func main() {
	var x = int(42)
	fmt.Println(Int(x).Twice())
}
```
------

- 不需要额外的类包装, 仅是类型转换就可扩展方法
- Go 的方法是绑定到类型的

---
### 还我自由的指针(C++)
----------------

```c
struct Int {
	int Twice() {
		const int* p = (int*)(this);
		return (*p) * 2;
	}
};
```

```c
int main() {
	int x = 42;
	int v = ((Int*)(&x))->Twice();
	printf("%d\n", v);
	return 0;
}
```

-------

- C++ 的方法其实也可以用于普通非 class 类型
- C++ 到普通成员函数其实也是绑定到类型的
- 只有纯虚方法是绑定到对象, 那就是接口

<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->
***
## 静态库和动态库
---------------

- 如何使用静态库
- 如何使用动态库
- 如何导出静态库
- 如何导出动态库
- 动态库的风险

---
### 如何使用静态库(A)
------------------

```c
// number/number.h

int number_add_mod(int a, int b, int mod);
```

```c
// number/number.c
int number_add_mod(int a, int b, int mod) {
	return (a+b)%mod;
}
```

```
$ cd ./number
$ gcc -c -o number.o number.c
$ ar rcs libnumber.a number.o
```

---------------

- cgo最终使用gcc的ld链接命令
- 只要是gcc兼容的`libxxx.a`格式的动态库均可使用


---
### 如何使用静态库(B)
------------------

```go
//#cgo CFLAGS: -I./number
//#cgo LDFLAGS: -L${SRCDIR}/number -lnumber
//
//#include "number.h"
import "C"
import "fmt"

func main() {
	fmt.Println(C.number_add_mod(10, 5, 12))
}
```

--------------

- `-lnumber`对应链接`libnumber.a`库
- `-L${SRCDIR}/number`指定链接库所在目录
- `-L`参数必须绝对路径(ld历史遗留问题导致)


---
### 如何使用动态库 - so 格式
------------------

```c
// number/number.h

int number_add_mod(int a, int b, int mod);
```

```c
// number/number.c
int number_add_mod(int a, int b, int mod) {
	return (a+b)%mod;
}
```

```
$ cd ./number
$ gcc -shared -o libnumber.so number.c
```

---------------

- 链接时 `libnumber.so` 和 `libnumber.a` 等价
- macOS 需要设置 `DYLD_LIBRARY_PATH` 环境变量
- Linux 需要设置 `LD_LIBRARY_PATH` 环境变量
---
### iEDA中 go调用ifp 使用的动态库和静态库
---------
```
package main

/*
#cgo CFLAGS:  -fPIC -fopenmp -O0 -Wall -g2 -ggdb -I/home/fsy/EDA/20230427/irefactor/src/operation/iFP/api 
#cgo LDFLAGS:-L/home/fsy/EDA/20230427/irefactor/src/third_party/abseil/lib/unix -L/home/fsy/EDA/20230427/build/lib -Wl,-rpath,/usr/lib/x86_64-linux-gnu:/lib/x86_64-linux-gnu:/home/fsy/EDA/20230427/irefactor/src/third_party/abseil/lib/unix:/home/fsy/EDA/20230427/build/lib:/usr/local/lib -fsanitize=address /home/fsy/EDA/20230427/build/lib/libgeometry_db.a /home/fsy/EDA/20230427/build/lib/libIdb.a /home/fsy/EDA/20230427/build/lib/libIdbBuilder.a /home/fsy/EDA/20230427/build/lib/libdef_builder.a /home/fsy/EDA/20230427/build/lib/liblef_builder.a /home/fsy/EDA/20230427/build/lib/libdef_service.a /home/fsy/EDA/20230427/build/lib/liblef_service.a /home/fsy/EDA/20230427/build/lib/libgeometry_db.a /home/fsy/EDA/20230427/build/lib/libIdb.a /home/fsy/EDA/20230427/build/lib/libIdbBuilder.a /home/fsy/EDA/20230427/build/lib/libdef_builder.a /home/fsy/EDA/20230427/build/lib/liblef_builder.a /home/fsy/EDA/20230427/build/lib/libdef_service.a /home/fsy/EDA/20230427/build/lib/liblef_service.a /home/fsy/EDA/20230427/build/lib/libifp_api.a -lm -lstdc++ /home/fsy/EDA/20230427/build/lib/libtool_manager.a /home/fsy/EDA/20230427/build/lib/libifp_init.a /home/fsy/EDA/20230427/build/lib/libifp_io_placer.a /home/fsy/EDA/20230427/build/lib/libifp_tapcell.a /home/fsy/EDA/20230427/build/lib/libtool_api_icts.a /home/fsy/EDA/20230427/build/lib/libtool_api_idrc.a /home/fsy/EDA/20230427/build/lib/libtool_api_ieval.a /home/fsy/EDA/20230427/build/lib/libtool_api_ifp.a /home/fsy/EDA/20230427/build/lib/libtool_api_ipdn.a /home/fsy/EDA/20230427/build/lib/libtool_api_ipl.a /home/fsy/EDA/20230427/build/lib/libtool_api_ipm.a /home/fsy/EDA/20230427/build/lib/libtool_api_irt.a /home/fsy/EDA/20230427/build/lib/libtool_api_ista.a /home/fsy/EDA/20230427/build/lib/libtool_api_ito.a /home/fsy/EDA/20230427/build/lib/libtool_api_ino.a /home/fsy/EDA/20230427/build/lib/libipm_api.a /home/fsy/EDA/20230427/build/lib/libipl-api.a /home/fsy/EDA/20230427/build/lib/libidm.a /home/fsy/EDA/20230427/build/lib/libito_source.a /home/fsy/EDA/20230427/build/lib/libino_source.a /home/fsy/EDA/20230427/build/lib/libfile_manager.a /home/fsy/EDA/20230427/build/lib/libidrc_api.a /home/fsy/EDA/20230427/build/lib/libista-engine.a /home/fsy/EDA/20230427/build/lib/libipm_source.a /home/fsy/EDA/20230427/build/lib/libipl-source.a /home/fsy/EDA/20230427/build/lib/libflow.a /home/fsy/EDA/20230427/build/lib/libeval_module.a /home/fsy/EDA/20230427/build/lib/libito_io.a /home/fsy/EDA/20230427/build/lib/libito_data.a /home/fsy/EDA/20230427/build/lib/libito_module.a /home/fsy/EDA/20230427/build/lib/libito_utility.a /home/fsy/EDA/20230427/build/lib/libino_io.a /home/fsy/EDA/20230427/build/lib/libino_module.a /home/fsy/EDA/20230427/build/lib/libicts_config.a /home/fsy/EDA/20230427/build/lib/libicts_database.a /home/fsy/EDA/20230427/build/lib/libicts_io.a /home/fsy/EDA/20230427/build/lib/libicts_balancer.a /home/fsy/EDA/20230427/build/lib/libicts_evaluator.a /home/fsy/EDA/20230427/build/lib/libicts_optimizer.a /home/fsy/EDA/20230427/build/lib/libicts_router.a /home/fsy/EDA/20230427/build/lib/libicts_synthesis.a /home/fsy/EDA/20230427/build/lib/libicts_api.a /home/fsy/EDA/20230427/build/lib/libicts_slew_aware.a /home/fsy/EDA/20230427/build/lib/libicts_cost_calculator.a /home/fsy/EDA/20230427/build/lib/libicts_timing_calculator.a /home/fsy/EDA/20230427/build/lib/libipm_top_manager.a /home/fsy/EDA/20230427/build/lib/libeval_api.a /home/fsy/EDA/20230427/build/lib/libieda_tcl.a /home/fsy/EDA/20230427/build/lib/libeval_congestion.a /home/fsy/EDA/20230427/build/lib/libeval_drc.a /home/fsy/EDA/20230427/build/lib/libeval_gds_wrapper.a /home/fsy/EDA/20230427/build/lib/libeval_power.a /home/fsy/EDA/20230427/build/lib/libeval_timing.a /home/fsy/EDA/20230427/build/lib/libeval_wirelength.a /home/fsy/EDA/20230427/build/lib/libeval_wrapper.a /home/fsy/EDA/20230427/build/lib/libito_api.a /home/fsy/EDA/20230427/build/lib/libino_api.a /home/fsy/EDA/20230427/build/lib/libipm_point_checker.a /home/fsy/EDA/20230427/build/lib/libipm_point_report.a /home/fsy/EDA/20230427/build/lib/libipm_point_generator.a /home/fsy/EDA/20230427/build/lib/libipm_point_sorter.a /home/fsy/EDA/20230427/build/lib/libipl-module-detail_placer.a /home/fsy/EDA/20230427/build/lib/libipl-module-legalizer.a /home/fsy/EDA/20230427/build/lib/libipl-module-wrapper.a /home/fsy/EDA/20230427/build/lib/libirt_api.a /home/fsy/EDA/20230427/build/lib/libtcl_config.a /home/fsy/EDA/20230427/build/lib/libtcl_idb.a /home/fsy/EDA/20230427/build/lib/libtcl_flow.a /home/fsy/EDA/20230427/build/lib/libtcl_icts.a /home/fsy/EDA/20230427/build/lib/libtcl_idrc.a /home/fsy/EDA/20230427/build/lib/libtcl_irt.a /home/fsy/EDA/20230427/build/lib/libtcl_ipm.a /home/fsy/EDA/20230427/build/lib/libtcl_ifp.a /home/fsy/EDA/20230427/build/lib/libtcl_ipdn.a /home/fsy/EDA/20230427/build/lib/libtcl_inst.a /home/fsy/EDA/20230427/build/lib/libtcl_ipl.a /home/fsy/EDA/20230427/build/lib/libtcl_ito.a /home/fsy/EDA/20230427/build/lib/libtcl_ista.a /home/fsy/EDA/20230427/build/lib/libtcl_report.a /home/fsy/EDA/20230427/build/lib/libtcl_ino.a /home/fsy/EDA/20230427/build/lib/libeval_data.a /home/fsy/EDA/20230427/build/lib/libipl_module_evaluator_wirelength.a /home/fsy/EDA/20230427/build/lib/libipl-analytical_placer.a /home/fsy/EDA/20230427/build/lib/libirt_source.a /home/fsy/EDA/20230427/build/lib/libieda_report.a /home/fsy/EDA/20230427/build/lib/libipdn_api.a /home/fsy/EDA/20230427/build/lib/libirt_data_manager.a /home/fsy/EDA/20230427/build/lib/libreport_basic.a /home/fsy/EDA/20230427/build/lib/libieda_report_db.a /home/fsy/EDA/20230427/build/lib/libieda_report_evaluator.a /home/fsy/EDA/20230427/build/lib/libieda_report_route.a /home/fsy/EDA/20230427/build/lib/libieda_report_place.a /home/fsy/EDA/20230427/build/lib/libieda_report_drc.a /home/fsy/EDA/20230427/build/lib/libipdn_plan.a /home/fsy/EDA/20230427/build/lib/libipdn_via.a /home/fsy/EDA/20230427/build/lib/libirt_detailed_router.a /home/fsy/EDA/20230427/build/lib/libirt_early_global_router.a /home/fsy/EDA/20230427/build/lib/libirt_gds_plotter.a /home/fsy/EDA/20230427/build/lib/libirt_global_router.a /home/fsy/EDA/20230427/build/lib/libirt_pin_accessor.a /home/fsy/EDA/20230427/build/lib/libirt_region_manager.a /home/fsy/EDA/20230427/build/lib/libirt_track_assigner.a /home/fsy/EDA/20230427/build/lib/libirt_violation_repairer.a /home/fsy/EDA/20230427/build/lib/libirt_universal_router.a /home/fsy/EDA/20230427/build/lib/libirt_logger.a /home/fsy/EDA/20230427/build/lib/libirt_monitor.a /home/fsy/EDA/20230427/build/lib/libirt_reporter.a /home/fsy/EDA/20230427/build/lib/libipdn_db.a /home/fsy/EDA/20230427/build/lib/libifp_api.a /home/fsy/EDA/20230427/build/lib/libtool_manager.a /home/fsy/EDA/20230427/build/lib/libifp_init.a /home/fsy/EDA/20230427/build/lib/libifp_io_placer.a /home/fsy/EDA/20230427/build/lib/libifp_tapcell.a /home/fsy/EDA/20230427/build/lib/libtool_api_icts.a /home/fsy/EDA/20230427/build/lib/libtool_api_idrc.a /home/fsy/EDA/20230427/build/lib/libtool_api_ieval.a /home/fsy/EDA/20230427/build/lib/libtool_api_ifp.a /home/fsy/EDA/20230427/build/lib/libtool_api_ipdn.a /home/fsy/EDA/20230427/build/lib/libtool_api_ipl.a /home/fsy/EDA/20230427/build/lib/libtool_api_ipm.a /home/fsy/EDA/20230427/build/lib/libtool_api_irt.a /home/fsy/EDA/20230427/build/lib/libtool_api_ista.a /home/fsy/EDA/20230427/build/lib/libtool_api_ito.a /home/fsy/EDA/20230427/build/lib/libtool_api_ino.a /home/fsy/EDA/20230427/build/lib/libipm_api.a /home/fsy/EDA/20230427/build/lib/libipl-api.a /home/fsy/EDA/20230427/build/lib/libidm.a /home/fsy/EDA/20230427/build/lib/libito_source.a /home/fsy/EDA/20230427/build/lib/libino_source.a /home/fsy/EDA/20230427/build/lib/libfile_manager.a /home/fsy/EDA/20230427/build/lib/libidrc_api.a /home/fsy/EDA/20230427/build/lib/libista-engine.a /home/fsy/EDA/20230427/build/lib/libipm_source.a /home/fsy/EDA/20230427/build/lib/libipl-source.a /home/fsy/EDA/20230427/build/lib/libflow.a /home/fsy/EDA/20230427/build/lib/libeval_module.a /home/fsy/EDA/20230427/build/lib/libito_io.a /home/fsy/EDA/20230427/build/lib/libito_data.a /home/fsy/EDA/20230427/build/lib/libito_module.a /home/fsy/EDA/20230427/build/lib/libito_utility.a /home/fsy/EDA/20230427/build/lib/libino_io.a /home/fsy/EDA/20230427/build/lib/libino_module.a /home/fsy/EDA/20230427/build/lib/libicts_config.a /home/fsy/EDA/20230427/build/lib/libicts_database.a /home/fsy/EDA/20230427/build/lib/libicts_io.a /home/fsy/EDA/20230427/build/lib/libicts_balancer.a /home/fsy/EDA/20230427/build/lib/libicts_evaluator.a /home/fsy/EDA/20230427/build/lib/libicts_optimizer.a /home/fsy/EDA/20230427/build/lib/libicts_router.a /home/fsy/EDA/20230427/build/lib/libicts_synthesis.a /home/fsy/EDA/20230427/build/lib/libicts_api.a /home/fsy/EDA/20230427/build/lib/libicts_slew_aware.a /home/fsy/EDA/20230427/build/lib/libicts_cost_calculator.a /home/fsy/EDA/20230427/build/lib/libicts_timing_calculator.a /home/fsy/EDA/20230427/build/lib/libipm_top_manager.a /home/fsy/EDA/20230427/build/lib/libeval_api.a /home/fsy/EDA/20230427/build/lib/libieda_tcl.a /home/fsy/EDA/20230427/build/lib/libeval_congestion.a /home/fsy/EDA/20230427/build/lib/libeval_drc.a /home/fsy/EDA/20230427/build/lib/libeval_gds_wrapper.a /home/fsy/EDA/20230427/build/lib/libeval_power.a /home/fsy/EDA/20230427/build/lib/libeval_timing.a /home/fsy/EDA/20230427/build/lib/libeval_wirelength.a /home/fsy/EDA/20230427/build/lib/libeval_wrapper.a /home/fsy/EDA/20230427/build/lib/libito_api.a /home/fsy/EDA/20230427/build/lib/libino_api.a /home/fsy/EDA/20230427/build/lib/libipm_point_checker.a /home/fsy/EDA/20230427/build/lib/libipm_point_report.a /home/fsy/EDA/20230427/build/lib/libipm_point_generator.a /home/fsy/EDA/20230427/build/lib/libipm_point_sorter.a /home/fsy/EDA/20230427/build/lib/libipl-module-detail_placer.a /home/fsy/EDA/20230427/build/lib/libipl-module-legalizer.a /home/fsy/EDA/20230427/build/lib/libipl-module-wrapper.a /home/fsy/EDA/20230427/build/lib/libirt_api.a /home/fsy/EDA/20230427/build/lib/libtcl_config.a /home/fsy/EDA/20230427/build/lib/libtcl_idb.a /home/fsy/EDA/20230427/build/lib/libtcl_flow.a /home/fsy/EDA/20230427/build/lib/libtcl_icts.a /home/fsy/EDA/20230427/build/lib/libtcl_idrc.a /home/fsy/EDA/20230427/build/lib/libtcl_irt.a /home/fsy/EDA/20230427/build/lib/libtcl_ipm.a /home/fsy/EDA/20230427/build/lib/libtcl_ifp.a /home/fsy/EDA/20230427/build/lib/libtcl_ipdn.a /home/fsy/EDA/20230427/build/lib/libtcl_inst.a /home/fsy/EDA/20230427/build/lib/libtcl_ipl.a /home/fsy/EDA/20230427/build/lib/libtcl_ito.a /home/fsy/EDA/20230427/build/lib/libtcl_ista.a /home/fsy/EDA/20230427/build/lib/libtcl_report.a /home/fsy/EDA/20230427/build/lib/libtcl_ino.a /home/fsy/EDA/20230427/build/lib/libeval_data.a /home/fsy/EDA/20230427/build/lib/libipl_module_evaluator_wirelength.a /home/fsy/EDA/20230427/build/lib/libipl-analytical_placer.a /home/fsy/EDA/20230427/build/lib/libirt_source.a /home/fsy/EDA/20230427/build/lib/libieda_report.a /home/fsy/EDA/20230427/build/lib/libipdn_api.a /home/fsy/EDA/20230427/build/lib/libirt_data_manager.a /home/fsy/EDA/20230427/build/lib/libreport_basic.a /home/fsy/EDA/20230427/build/lib/libieda_report_db.a /home/fsy/EDA/20230427/build/lib/libieda_report_evaluator.a /home/fsy/EDA/20230427/build/lib/libieda_report_route.a /home/fsy/EDA/20230427/build/lib/libieda_report_place.a /home/fsy/EDA/20230427/build/lib/libieda_report_drc.a /home/fsy/EDA/20230427/build/lib/libipdn_plan.a /home/fsy/EDA/20230427/build/lib/libipdn_via.a /home/fsy/EDA/20230427/build/lib/libirt_detailed_router.a /home/fsy/EDA/20230427/build/lib/libirt_early_global_router.a /home/fsy/EDA/20230427/build/lib/libirt_gds_plotter.a /home/fsy/EDA/20230427/build/lib/libirt_global_router.a /home/fsy/EDA/20230427/build/lib/libirt_pin_accessor.a /home/fsy/EDA/20230427/build/lib/libirt_region_manager.a /home/fsy/EDA/20230427/build/lib/libirt_track_assigner.a /home/fsy/EDA/20230427/build/lib/libirt_violation_repairer.a /home/fsy/EDA/20230427/build/lib/libirt_universal_router.a /home/fsy/EDA/20230427/build/lib/libirt_logger.a /home/fsy/EDA/20230427/build/lib/libirt_monitor.a /home/fsy/EDA/20230427/build/lib/libirt_reporter.a /home/fsy/EDA/20230427/build/lib/libipdn_db.a /home/fsy/EDA/20230427/build/lib/libicts_model.a /home/fsy/EDA/20230427/build/lib/libidrc_src.a /home/fsy/EDA/20230427/build/lib/libidrc_multi_pattern.a /home/fsy/EDA/20230427/build/lib/libidrc_spot_parser.a /home/fsy/EDA/20230427/build/lib/libidrc_shape_check.a /home/fsy/EDA/20230427/build/lib/libidrc_db.a /home/fsy/EDA/20230427/build/lib/libidrc_region_query.a /home/fsy/EDA/20230427/build/lib/libidrc_spacing_check.a /home/fsy/EDA/20230427/build/lib/libidrc_db.a /home/fsy/EDA/20230427/build/lib/libidrc_region_query.a /home/fsy/EDA/20230427/build/lib/libidrc_spacing_check.a /home/fsy/EDA/20230427/build/lib/libidrc_config.a /home/fsy/EDA/20230427/build/lib/libipl-configurator.a /usr/lib/gcc/x86_64-linux-gnu/10/libgomp.so /usr/lib/x86_64-linux-gnu/libpthread.so /home/fsy/EDA/20230427/build/lib/libipl-layout_checker.a /home/fsy/EDA/20230427/build/lib/libipl-module-filler.a /home/fsy/EDA/20230427/build/lib/libipl-module-grid_manager.a /home/fsy/EDA/20230427/build/lib/libipl-module-logger.a /home/fsy/EDA/20230427/build/lib/libipl-module-macro_placer.a /home/fsy/EDA/20230427/irefactor/src/operation/iPL/source/module/macro_placer/libs/libmetis.a /home/fsy/EDA/20230427/build/lib/libipl-center_placer.a /home/fsy/EDA/20230427/build/lib/libipl-module-topology_manager.a /home/fsy/EDA/20230427/build/lib/libeval_config.a /home/fsy/EDA/20230427/build/lib/libipm_logger.a /home/fsy/EDA/20230427/build/lib/libipl_module_evaluator_density.a /home/fsy/EDA/20230427/build/lib/libtcl_util.a /home/fsy/EDA/20230427/build/lib/libshell-cmd.a /home/fsy/EDA/20230427/build/lib/libeval_util.a /home/fsy/EDA/20230427/build/lib/libipl-solver-nesterov.a /home/fsy/EDA/20230427/build/lib/libusage.a /home/fsy/EDA/20230427/build/lib/libisr.a /home/fsy/EDA/20230427/build/lib/libflute.a /home/fsy/EDA/20230427/build/lib/libslute.a /home/fsy/EDA/20230427/build/lib/libsta.a /home/fsy/EDA/20230427/build/lib/libgraph.a /home/fsy/EDA/20230427/build/lib/libaocv-parser.a  -lyaml-cpp /home/fsy/EDA/20230427/build/lib/libreport.a /home/fsy/EDA/20230427/build/lib/libdelay.a /home/fsy/EDA/20230427/build/lib/libsdc-cmd.a /home/fsy/EDA/20230427/build/lib/libsdc.a /home/fsy/EDA/20230427/build/lib/libnetlist.a /home/fsy/EDA/20230427/build/lib/libtcl.a -ltcl8.6 /home/fsy/EDA/20230427/build/lib/libtime.a -labsl_base -labsl_int128 -labsl_city -labsl_hash -labsl_hashtablez_sampler -labsl_random_seed_gen_exception -labsl_random_seed_sequences -labsl_raw_hash_set -labsl_malloc_internal -labsl_spinlock_wait -labsl_synchronization -labsl_raw_logging_internal -labsl_time -labsl_time_zone -lstdc++fs /home/fsy/EDA/20230427/build/lib/libfort.a /home/fsy/EDA/20230427/build/lib/libliberty.a -labsl_throw_delegate /home/fsy/EDA/20230427/build/lib/libsta-solver.a /home/fsy/EDA/20230427/build/lib/libIdbBuilder.a /home/fsy/EDA/20230427/build/lib/libverilog_builder.a /home/fsy/EDA/20230427/build/lib/libutility.a /home/fsy/EDA/20230427/build/lib/libIdbBuilder.a /home/fsy/EDA/20230427/build/lib/libverilog_builder.a /home/fsy/EDA/20230427/build/lib/libutility.a /home/fsy/EDA/20230427/build/lib/libgds_builder.a /home/fsy/EDA/20230427/build/lib/libgdsii-parser.a /home/fsy/EDA/20230427/build/lib/libverilog-parser.a -Wl,-Bstatic -labsl_throw_delegate /home/fsy/EDA/20230427/build/lib/liblog.a -Wl,-Bdynamic -lgflags -lglog -lpthread -lunwind /home/fsy/EDA/20230427/build/lib/libdef_builder.a /home/fsy/EDA/20230427/irefactor/src/database/manager/parser/lefdef/def/lib/libdef.a /home/fsy/EDA/20230427/irefactor/src/database/manager/parser/lefdef/def/lib/libdefzlib.a /usr/local/lib/libz.so /home/fsy/EDA/20230427/build/lib/liblef_builder.a /home/fsy/EDA/20230427/irefactor/src/database/manager/parser/lefdef/lef/lib/liblef.a /home/fsy/EDA/20230427/build/lib/libdef_service.a /home/fsy/EDA/20230427/build/lib/liblef_service.a /home/fsy/EDA/20230427/build/lib/libIdb.a /home/fsy/EDA/20230427/build/lib/libtech_db.a /home/fsy/EDA/20230427/build/lib/libgeometry_db.a /home/fsy/EDA/20230427/build/lib/libstr.a -fsanitize=address /home/fsy/EDA/20230427/irefactor/src/third_party/abseil/lib/unix/libabsl_strings.so

# include"ifp_api_wrapperC.h"
*/
import "C"
```


---
### 如何导出静态库
----------------

```go
package main

import "C"

func main() {}

//export number_add_mod
func number_add_mod(a, b, mod C.int) C.int {
	return (a + b) % mod
}
```

```
$ go build -buildmode=c-archive -o number.a
```

-------------

- cgo还会生成一个 `number.h` 文件

---
### 如何导出动态库(A)
----------------

```
$ go build -buildmode=c-shared -o number.so
```
----------

- 动态库和静态库的生成方式类似
- Go1.9 之前不支持 Windows
- Windows 下可从静态库手工生成动态库



---
### 动态库的风险
--------------

- Linux 下共享一个 libc.so, 问题不大
- Windows下有多个 msvcrt.dll, 问题非常严重
- Windows下gcc和vc有各自的实现

------

- 比如从VC6实现的dll中 malloc 了一块内存
- 然后在cgo中 `C.free` 内存, 将导致崩溃

------

- cgo 打开了一个文件指针 `*C.FILE`
- 然后传入 VC 实现的动态库的 `C.fclose(f)` 关闭
- 因为 `C.FILE` 在 vc 和 gcc 中实现是不同的, 导致崩溃

----

- 原则: 跨动态库时, 尽量避免使用 malloc 和 free



<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->
***

## 编译和链接参数
--------------

- CFLAGS/CPPFLAGS/CXXFLAGS
- LDFLAGS




---
### CFLAGS/CPPFLAGS/CXXFLAGS
---------------------------

- CFLAGS 只包含纯 C 代码(`*.c`)
- CPPFLAGS 包含 C/C++ 代码(`*.c`,`*.cc`,`*.cpp`,`*.cxx`)
- CXXFLAGS 只包含纯 C++ 代码(`*.cc`,`*.cpp`,`*.cxx`) 
			不能使用类、命名空间等高级特性

---
### LDFLAGS
----------

- 链接阶段没有C和C++之分
- 链接库的路径必须是绝对路径(ld历史依赖问题)
- cgo 中的 `${SRCDIR}` 为当前目录的绝对路径
---------
- cgo只支持gcc编译链接 gcc和g++区别不大 
- 手动加上c++标准库

```
-lm -lstdc++
```

<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  -->
***
## 编译与链接
-------------
1.编译与链接过程


2.目标文件、动态库、可执行文件和静态库

3.链接——符号解析与重定位

4.API&ABI
---

### 1.编译与链接过程
-------------------
![](https://buaa007.oss-cn-beijing.aliyuncs.com/images%E7%BC%96%E8%AF%91%E4%B8%8E%E9%93%BE%E6%8E%A5.png)<!-- .element: style="width:50%;" -->
---
### 2.目标文件、动态库、可执行文件和静态库
-------------------
目标文件、动态库、可执行文件其存储格式都是elf，只是elf文件格式不同

----------------
静态库：
多个目标文件捆绑成一个文件

一个包含很多目标文件的压缩包

ar -t libm.a 查看包含哪些.o文件

---
### 3.链接——符号修饰与函数签名
--------------
#### C 的符号修饰非常简单
基本上就是函数或变量名本身

-------------
#### C++的符号修饰非常复杂
C++拥有类、继承、重载、命名空间等特性，符号管理极其复杂

![](https://buaa007.oss-cn-beijing.aliyuncs.com/imagesc.png)<!-- .element: style="width:50%;" -->

保持与C的兼容性引入extern"C"{}:

		在括号内C++的符号修饰机制将不会起作用

---
### 4.API&ABI
------
ABI （Application Binary Interface）
决定目标文件之间是否二进制兼容
- 内置类型
- 组合类型
- 符号修饰
- 函数调用方式
- 等等
