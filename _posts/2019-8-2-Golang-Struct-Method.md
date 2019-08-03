---
layout: post
title: golang方法Receiver的自动类型转换
---

## 背景

go语言的设计哲学与其他面向对象的编程语言不通，语言设计者希望语言的底层架构尽可能简洁，所以go语言的类方法的底层实现是一个普通的函数，只是将其他语言常见的this指针，显式地放到了函数的第一个参数（称之为方法的接收者：Receiver），然后对这个函数（即go语言中的方法表达式）进行一些封装，从而看起来像一个类的方法。这里主要讨论go语言的方法接收者receiver，以及方法调用时的自动类型转换（值与指针的转换）



## struct方法调用时指针Receiver与值Receiver

首先明确一点的是，go语言不管用哪种receiver，都是采用值拷贝的方式将对象/指针拷贝给receiver，然后再在函数中执行；只不过如果是值receiver，拷贝的是struct对象的值；如果是指针类型的receiver，拷贝的struct对象的指针。

- 指针Receiver

  ```go
  func (s *myStruct) Method() {...}
  ```

  这种方式Method对传入的对象的指针，Method内部发生的修改会直接影响到对象（外部可见）。所以通常setter一类的方法会用指针receiver。

- 值Receiver

  ```go
  func (s myStruct) Method() {...}
  ```

  这种方式Method传入的对象值的拷贝，Method内部发生的修改不会影响到对象（外部不可见）。通常Getter一类的方法会用值receiver，以保证安全。



## go语言对值类型和指针类型Receiver的自动转换

不知出于什么考虑，也许是降低gopher的使用难度，golang对与指针和值型的receiver做了自动转换，所以通常情况下，你并不需要管对象是指针还是值，他们都可以调用任何receiver为指针/值的类方法，只需要记住上面一节对于对象修改的可见性。

举个列子，struct定义如下：

``` go
type myStruct struct {
  Message string
}

func (s myStruct) GetMessage() string {
  fmt.Printf("Address of receiver %p\n", &s) // 地址A
  return s.Message
}

func (s *myStruct) SetMessage(message string) {
  fmt.Printf("Address of receiver %p\n", &s) // 地址B
  s.Message = message
}
```

- 以对象的指针调用方法

``` go
s := &myStruct {
  Message : "abc",
}
fmt.Printf("Addrss of s %p\n", s) // 地址C
s.SetMessage("efg") // 不用转换
msg := s.GetMessage() // golang自动转换为(*s).GetMessage()
fmt.Printf("After calling set, the message of s: %v\n", msg)

```

地址A不等于地址C，地址B等于地址C。当对象为指针，receiver为值时，golang会取出对象指针所指的指，并值复制给receiver，所以地址不同。

- 以对象值调用方法

```go
s := myStruct {
  Message : "abc"
}
fmt.Printf("Addrss of s %p\n", &s) // 地址C
s.SetMessage("efg") // golang自动转换为(&s).SetMessage("efg")
msg := s.GetMessage() // 不用转换
fmt.Printf("After calling set, the message of s: %v\n", msg)
```

同样地址A不等于地址C，地址B等于地址C。当对象为值，receiver为指针，golang会自动将对象的地址通过值传递的方式给到receiver，所以receiver对应的地址和对象的地址一致。



## struct组合中的receiver转换

## struct赋值给interface时的receiver转换

之前所有的case，golang都能够做指针和值的自动转换，看起来我们不用关心指针和值都能够调用哪些方法，但放到struct赋值给interface（多态常见的情形）情况有不同，看下面两组情况：

- 情况1

````go
type MyInterface interface {
  Method1()
  Method2()
}

type MyInterfaceImp1 struct {
}

func (m MyInterfaceImp1) Method1() {
  fmt.Printf("Address %p", &m) //地址1
  ...
}

func (m MyInterfaceImp1) Method2() {
  fmt.Printf("Address %p", &m)
  ...
}
````

以上情况，MyInterfaceImp1的两个方法的receiver都是值类型，那么一下两个赋值都可以：

```go
var myInterface MyInterface // 地址2
m1 := MyInterfaceImp1{} // 地址3
m2 := &MyInterfaceImp1{} // 地址4

myInterface = m1 // 赋值成功
myInterface = m2 // 赋值成功
```

将m1赋值给myInterface: m1的值拷贝给myInterface，然后调用函数的时候也是直接将myInterface的值拷贝给方法的receiver；对比一下各个地址：地址1和地址2不同，地址2和地址3不同。

将m2赋值给myInterface：m2的地址拷贝给myInterface，然后调用方法时，golang自动将myInterface的地址对应的值取出，值拷贝给方法的receiver。对比一下各个地址：地址1和地址2不同，地址2和地址4相同。

- 情况2

```go
type MyInterface interface {
  Method1()
  Method2()
}

type MyInterfaceImp1 struct {
}

func (m MyInterfaceImp1) Method1() {
  fmt.Printf("Address %p", &m)
  ...
}

func (m *MyInterfaceImp1) Method2() {
  fmt.Printf("Address %p", &m)//地址1
  ...
}
```

以上情况，MyInterfaceImp1的一个方法的receiver是指针类型，那么将MyInterfaceImp1的实例赋值给MyInterface变量会编译错误：

```go
var myInterface MyInterface // 地址2
m1 := MyInterfaceImp1{} // 地址3
m2 := &MyInterfaceImp1{} // 地址4

myInterface = m1 // 赋值失败
myInterface = m2 // 赋值成功
```

失败的原因是：m1赋值给myInterface之后，myInterface拿到的是m1的值拷贝，当调用Method2的时候，按照golang自动转换原则，会将myInterface的地址拷贝给Method2的receiver，但这个地址和原来m1的地址是不同的。按照Method2的语意是操作m1的地址，但实际上可以看出并没有操作m1的地址，而是操作了myInterface地址。golang为了避免这种奇怪的现象，于是对接口赋值做了限制。

**只有在没有指针receiver的时候，才可以将struct的值赋值给interface**

**struct指针在任何情况下都可以赋值给interface**
