---
layout:       post
title:        "Go闭包及其陷阱"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Go
---

GO是支持函数式编程的，在GO中，函数是一等公民，它可以作为变量被赋值，作为参数被传递到其它函数中。

那么既然能够将函数作为变量，GO就一定是支持`闭包`的。

## 闭包概念

什么是闭包呢？其实很简单，我们都知道，变量是有其`作用域`的。例如，在函数中声明定义的变量就只能够在当前函数中使用，当函数结束时，该变量空间就会被释放，外部无法再使用。

但是，闭包允许我们将变量的作用域进行扩充。当我们的函数会返回另外一个函数时（不管是直接返回出去，还是保存在其他变量中间接返回出去），这个被返回的函数如果引用到一些内部的变量，那么这些内部的变量值将会和被返回的函数一起“打包”，作为一个整体返回出去。

这样，在外部，不仅可以使用到被返回的函数，也可以使用到本应该是不可见的，那些被一起打包返回的内部变量的值。

通俗地说，这其实是一个“借刀杀人”的过程。例如你想访问函数`fun1`的某个内部变量`a`的值，但是`fun1`没有直接返回这个值。这在一般情况下就无法实现了：

```go
package main

import "fmt"

func fun1() {
	a := 10
	fmt.Println("fun1 a =", a)
}


func main() {
	// 这里无法直接访问fun1中的a
	fmt.Println("fun1 a =", a)   // 编译错误：undefined: a
}
```

但是，如果我们让`fun1`返回另外一个函数（假设为`fun2`），这个函数使用到了变量`a`。因为`fun2`被返回出去了，外部能够访问了，那么变量`a`就会和`fun2`作为闭包一起被返回出去，外部就能获取`a`的值了。我们通过第三者`fun2`访问到了`a`：

```go
package main

import "fmt"

// 需要被闭包返回出去的函数
type fun2 func() int

func fun1() fun2 {
	a := 10
	return func() int {
		// 将fun1中的a返回出去
		return a
	}
}

func main() {
	// 先获取fun2
	fun2 := fun1()
	// 从fun2获取a
	a := fun2()

	fmt.Println("a =", a)
}
```

这就是闭包最简单的用法了。

## 闭包的作用（通过例子学习）

如果这样，你可能会想：那闭包有什么用呢？我为什么不直接返回`a`，还需要费劲地通过一个函数来把它返回出去？

当然，上面的例子没有什么意义，只是为了让大家简单地认识闭包。我们来看一个更加具体的例子：通过闭包实现求算斐波那契数列。

和传统的求斐波那契不一样，我们想实现这么个效果：

- 获取一个fib()函数，每次调用该函数，都能返回下一个斐波那契数。
- 例如，运行`fmt.Println(fib(), fib(), fib(), fib(), fib(), fib())`，可以输出：`1 1 2 3 5 8`。

我们的这个`fib`函数似乎有记忆功能，它能够“记住”我们之前的调用，根据之前的结果来推算出现在的斐波那契值。

我们可以定义两个全局变量来实现：

```go
import "fmt"

var front, next = 1, 0


func fib() int {

	result := front + next
	front = next
	next = result

	return result

}

func main() {

	fmt.Println(fib(), fib(), fib(), fib(), fib())

}
```

但是，`front`、`next`这两个全局变量只在`fib`函数中被使用，强行把它们定义为全局的似乎并不妥当。

那更好的做法就是不定义全局变量，通过闭包将函数内的变量转换为全局的。为了实现这样的效果，`fib`函数必须通过其它函数返回，这样就可以在其它函数定义这样的“类全局”的变量了：

```go
package main

import "fmt"

func fibFunc() func() int {
	front, next := 1, 0

	return func() int {

		result := front + next
		front = next
		next = result

		return result
	}
}

func main() {

	fib := fibFunc()

	fmt.Println(fib(), fib(), fib(), fib(), fib(), fib())
}
```

闭包的另一个好处就是我们可以创建不同的`fib`实体，来产生互不影响的独立的`fib`序列：

```go
func main() {

	fib1 := fibFunc()
	fib2 := fibFunc()

	// fib1和fib2是两个完全不同的fib序列，它们不会互相影响
	fmt.Println(fib1(), fib1(), fib1())
	fmt.Println(fib2(), fib2())

}
```

这会输出：

```go
1 1 2
1 1
```

以后在遇到一些全局变量仅被少数函数使用的场景，就可以考虑是否使用闭包会更好。

## 闭包陷阱

闭包的使用存在一些陷阱，导致它的输出和我们想象的可能不大一样，在开发过程中一定要特别小心。

### 引用陷阱

千万注意闭包是一个引用，引用了外部变量，而不是拷贝。这意味着外部域对变量的改变会影响内部：

```go
package main

import "fmt"

func fun() func() {
	num := 10
	fun := func() {
		fmt.Println("num =", num)
	}
	num = 20
	num = 30
	return fun
}

func main() {
	fun := fun()
	fun()
}
```

这段代码输出`num = 30`，因为外部对`num`的赋值影响了内部域，不管赋值出现在了哪里。

当然，如果在声明变量函数之后马上调用，情况就不一样了：

```go
package main

import "fmt"

func fun() func() {
	num := 10
	fun := func() {
		fmt.Println("num =", num)
	}
	fun()    // 这里调用了第一次
	num = 20
	num = 30
	return fun
}

func main() {
	fun := fun()
	fun()   // 这里调用第二次
}
```

第一次输出`num = 10`。第二次仍然是`num = 30`。因为第一次调用的时候直接使用了`num`，这时候`num`还没有被赋值，仍然是10。

我们可以通过打印地址的方式来证明闭包变量是引用：

```go
package main

import "fmt"

func fun() func() {
	num := 10
	fmt.Println("num in fun:", &num)
	return func() {
		fmt.Println("num in closure:", &num)
	}
}

func main() {
	fun()()
}
```

在我的机器上输出：

```text
num in fun: 0xc000094008
num in closure: 0xc000094008
```

在其它机器上结果可能不一样，但是地址是相同的，这就更加验证了闭包是引用。

### 迭代闭包陷阱

闭包有一个常见的错误，那就是迭代配合闭包的使用。我们来直接看代码：

```go
package main

import "fmt"

func main() {

	funcs := make([]func(), 0)
	for i := 0; i < 5; i++ {
		funcs = append(funcs, func() {
			fmt.Println(i)
		})
	}

	for _, fun := range funcs {
		fun()
	}
}
```

运行这段代码，输出：

```text
5
5
5
5
5
```

这是一个常见的错误，在第一个`for`循环中，`i`是一个迭代对象，它在整个循环代码块中都是同一个变量。而我们知道，闭包实际上是一个引用，它引用的是`i`，那么，`funcs`中所有的函数的`i`实际上都引用了一个变量。

而当闭包函数调用时，`i`的迭代已经结束，这时候闭包函数中对`i`的输出当然是迭代最后的那个`i`了，也就是5了。

要解决这个问题，只需要让闭包引用不引用迭代变量即可，在闭包前进行一次赋值即可：

```go
package main

import "fmt"

func main() {

	funcs := make([]func(), 0)
	for i := 0; i < 5; i++ {
		ii := i   // 加了一行赋值
		funcs = append(funcs, func() {
			fmt.Println(ii)  // 不再对i进行引用，引用ii
		})
	}

	for _, fun := range funcs {
		fun()
	}
}
```

这样每个函数引用的变量就是不同的变量了，输出：

```text
0
1
2
3
4
```

在遇到`迭代`+`闭包`这样的组合的时候，一定要特别特别注意这个问题，这里特别容易犯错。
