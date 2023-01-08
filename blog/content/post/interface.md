---
title: interface
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["golang"]
tags: ["golang"]
---

# interface
#技术/golang/interface
[golang/4、interface.md at main · aceld/golang · GitHub](https://github.com/aceld/golang/blob/main/4%E3%80%81interface.md)

## interface的内部构造(非空接口iface情况)
### 以下代码打印出来什么内容，说出为什么。
```go
package main

import (
	“fmt”
)

type People interface {
	Show()
}

type Student struct{}

func (stu *Student) Show() {

}

func live() People {
	var stu *Student
	return stu
}

func main() {
	if live() == nil {
		fmt.Println(“AAAAAAA”)
	} else {
		fmt.Println(“BBBBBBB”)
	}
}
```
### 结果
BBBBBBB

### 分析：
我们需要了解interface 的内部结构，才能理解这个题目的含义。
interface在使用的过程中，共有两种表现形式
1. 空接口(empty interface)
```
var MyInterface interface{}
```
2. 非空接口(non-empty interface)
```
type MyInterface interface {
		function()
}
```

这两种interface类型分别用两种
struct表示空接口为eface，非空接口为iface

#### 空接口eface
 空接口eface结构，由两个属性构成，一个是类型信息_type，一个是数据信息。其数据结构声明如下：
```go
type eface struct {      //空接口
    _type *_type         //类型信息
    data  unsafe.Pointer //指向数据的指针(go语言中特殊的指针类型unsafe.Pointer类似于c语言中的void*)
}
```
_type属性是GO语言中所有类型的公共描述，Go语言几乎所有的数据结构都可以抽象成 _type，是所有类型的公共描述，**type负责决定data应该如何解释和操作，**type的结构代码如下:
```go
type _type struct {
    size       uintptr  //类型大小
    ptrdata    uintptr  //前缀持有所有指针的内存大小
    hash       uint32   //数据hash值
    tflag      tflag
    align      uint8    //对齐
    fieldalign uint8    //嵌入结构体时的对齐
    kind       uint8    //kind 有些枚举值kind等于0是无效的
    alg        *typeAlg //函数指针数组，类型实现的所有方法
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}
```

data属性:表示指向具体的实例数据的指针，他是一个unsafe.Pointer类型，相当于一个C的万能指针

#### 非空接口iface
iface 表示 non-empty interface 的数据结构，非空接口初始化的过程就是初始化一个iface类型的结构，其中data的作用同eface的相同，这里不再多加描述。
```go
type iface struct {
  tab  *itab
  data unsafe.Pointer
}
```
iface结构中最重要的是itab结构（结构如下），每一个 itab 都占 32 字节的空间。itab可以理解为pair<interface type, concrete type> 。itab里面包含了interface的一些关键信息，比如method的具体实现。
```go
type itab struct {
  inter  *interfacetype   // 接口自身的元信息
  _type  *_type           // 具体类型的元信息
  link   *itab
  bad    int32
  hash   int32            // _type里也有一个同样的hash，此处多放一个是为了方便运行接口断言
  fun    [1]uintptr       // 函数指针，指向具体类型所实现的方法
}
```
其中值得注意的字段，个人理解如下：
1. interface type包含了一些关于interface本身的信息，比如package path，包含的method。这里的interfacetype是定义interface的一种抽象表示。
2. type表示具体化的类型，与eface的 *type类型相同。*
3. hash字段其实是对_type.hash的拷贝，它会在interface的实例化时，用于快速判断目标类型和接口中的类型是否一致。另，Go的interface的Duck-typing机制也是依赖这个字段来实现。
4. fun字段其实是一个动态大小的数组，虽然声明时是固定大小为1，但在使用时会直接通过fun指针获取其中的数据，并且不会检查数组的边界，所以该数组中保存的元素数量是不确定的。

所以，People拥有一个Show方法的，属于非空接口，People的内部定义应该是一个iface结构体
```go
type People interface {
    Show()  
}
```
```go
func live() People {
    var stu *Student
    return stu      
}
``` 

stu是一个指向nil的空指针，但是最后
return stu 会触发匿名变量 People = stu 值拷贝动作，所以最后
live()放回给上层的是一个
People insterface{}类型，也就是一个
iface struct{}类型。 stu为nil，只是
iface中的data 为nil而已。 但是
iface struct{}本身并不为nil.


## 下面代码结果为什么？
```go
func Foo(x interface{}) {
	if x == nil {
		fmt.Println(“empty interface”)
		return
	}
	fmt.Println(“non-empty interface”)
}
func main() {
	var p *int = nil
	Foo(p)
}
```
### 结果
non-empty interface
### 分析
### 不难看出，
Foo()的形参x interface{} 是一个空接口类型eface struct{} 。
在执行Foo(p)的时候，触发x interface{} = p语句，所以 x 结构体本身不为nil，而是data指针指向的p为nil。

## 值接收方法、指针接收方法
如果实现了接收者是值类型的方法，会隐含地也实现了接收者是指针类型的方法，反之不然
**如果方法的接收者是值类型，无论调用者是对象还是对象指针，修改的都是对象的副本，不影响调用者；如果方法的接收者是指针类型，则调用者修改的是指针指向的对象本身**。

当实现了一个接收者是值类型的方法，就可以自动生成一个接收者是对应指针类型的方法，因为两者都不会影响接收者。但是，当实现了一个接收者是指针类型的方法，如果此时自动生成一个接收者是值类型的方法，原本期望对接收者的改变（通过指针实现），现在无法实现，因为值类型会产生一个拷贝，不会真正影响调用者。
```go
func main() {
	setName()
}

func setName() {
	s := &Student{}
	s.SetName("123")
	fmt.Println(s.Name)
}

type Student struct {
	Name string
}

func (s Student) SetName(name string) {
	s.Name = name
}
```
输出””

- 对于值接收者，如果调用者也是值对象，那么会将调用者的值拷贝一份，并执行方法，方法的调用不会影响到调用者值。 如果调用者是指针对象，那么会解引用指针对象为值，然后将解引的对象拷贝一份，然后 执行方法。
- 对于指针接收者，如果调用者是值对象，会使用值的引用来调用方法，上例中，yuting.growUp() 实际上是 、(&yuting).growUp(), 所以传入指针接收者方法的对象地址和 调用者地址一样。如果调用者是指针对象，实际上也是“传值”，方法里的操作会影响到调用者，类似于指针传参，拷贝了一份指针，但是指针指向同一个对象。


使用指针作为方法的接收者的理由：
	* 方法能够修改接收者指向的值。
	* 避免在每次调用方法时复制该值，在值的类型为大型结构体时，这样做会更加高效。


