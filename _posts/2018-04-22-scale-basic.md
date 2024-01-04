---
layout:       post
title:        "Scala 基础语法"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Scala
---

Scala是多范式的编程语言,本教程着重介绍函数式编程.

本教程适用于已经掌握了Java编程语言的人.

## 常量和变量

var可以用来声明一个变量:

```scala
var var_name[:type] = xxx
```

其中,属性的类型声明可以省略.那么scala会自动推测属性的类型.即使类型可以省略,scala和python不一样,还是有类型的区别的.

Scala官方不建议定义过多变量,最好不要定义任何变量(所有变量的效果通过函数完成).

val可以用来声明一个常量:

```scala
val val_name[:type] = xxx
```

常量不可以被修改.

## 基本类型

Scala提供以下的基本数据类型:

类型 | 说明
---|---
Byte | 字节类型
Short | 短整型
Int | 整型
Long | 长整型
Char | 字符
String | 字符串
Float | 浮点型
Double | 双精度
Boolean | 布尔

除了String,位于java.lang中,其余的基本类型都处于scala的包下.Scala没有所有真正基本类型的概念,实际上全部都是封装类.但是这些基本类型可以直接在scala中赋值(不需要通过new来赋值).

scala和java.lang包会自动被scala引入,所以使用这些类不需要任何声明.

例如:

```scala
val flag = true
// 等价于
val flag: Boolean = true
```

在Scala中,String支持三引号的写法."""xxx"""这样可以支持原生的字符串,在三引号中的字符串会被直接当作原生字符串处理,不需要任何转义.

Scala为基本类型提供了富包装类RichXxx.并且可以实现自动转换.

## 操作符

Scala中,操作符即方法,方法即操作符.scala中没有真正意义上的操作符,操作符实际上可以通过函数的方式调用:

```scala
// 以下两种写法是一样的:
val num1 = 2 + 3
val num2 = 2.+(3)
```

字面量2作为对象存在,对象中有+方法.

对于一个方法,可以用操作符的方法去调用:

```scala
// 以下两种写法一样:
val str1 = "123456".charAt(1)
val str2 = "123456" charAt 1
```

Scala没有真正意义上的操作符,所有的操作符都会被转换为方法的调用.

如果方法接收1个以上的参数,括号不能被省略,如果没有参数,可以省略点和括号:

```scala
val str1 = "123456".substring(2, 3)
val str2 = "123456" substring (2, 3)
val str3 = "a".toUpperCase()
val str4 = "a" toUpperCase
```

Scala中方法是有优先级的.在表达式中体现为操作符的优先级.对于一连串的方法调用,优先级高的方法会先得到调用.

优先级是按照方法的名字确定的,是按照方法的名称的第一个字符确定的,以下字符越靠前则优先级越高:

1. \* / %
2. \+ -
3. :
4. = !
5. <>
6. &
7. ^
8. |
9. 所有字母
10. 所有赋值操作符

可以通过括号改变优先级,对于优先级相同的方法,是按照从左到右来调用的.

操作符分为三类:

- 中缀操作符,操作符在两个操作数之间
- 后缀操作符,操作符位于一个唯一的操作数的后面
- 前缀操作符,操作符位于一个唯一的操作数的前面,Scala只提供四种: -, +, ~, %.前缀操作符会被翻译为"unary_x",例如"-3"会被翻译为"3.unary_-()",前缀操作符我们无法自己定义.

特殊的,以":"字符为结尾的方法,是右操作数调用,传入左操作数.例如"a::b"会被翻译为"(b).::(a)"

## 内建控制结构

if可以用作判断,用法和java基本一样.但是scala中if有返回值,if在执行完毕后会将最后一个表达式的值给返回,例如:

```scala
val num = 10

val res = if (num > 100) {
    "num大于100"
}
else if (num < 100 && num > 5) {
    "num在5到100之间"
}
else {
    "num小于5"
}

println(res)
```

res的结果应该是"num在5到100之间".if实际上是一种函数,不在函数中进行打印满足函数式编程的理念.

while和do...while可以用于循环,和java用法基本一样.scala中的while的返回值是Unit,表示没有返回值.while需要用到变量,并且可能产生额外的影响,不符合函数式编程的思想,所以官方不建议使用while编写循环,建议使用递归替换.

例如,使用递归实现1到100的计算:

```scala
def sum100(x: Int, sum: Int): Int = {
    if (x <= 100) {
        sum100(x + 1, sum + x)
    }
    else {
        sum
    }
}

val res = sum100(1, 0)
println(res)
```

所有的循环控制都应该被写成这样的递归形式.

for的使用和java也是类似的.可以使用for来遍历集合:

```scala
val list = List(1, 3, 4, 5, 12)
for (x:Int <- list) {
    println(x)
}
```

to运算符用于生成一个范围的数组,可以直接用于一定范围的循环:

```scala
for (x <- 1 to 5) {
    println(x)
}
```

for还可以实现过滤,例如可以通过下面代码遍历0-100的偶数:

```scala
for (x <- 0 to 100; if x % 2 == 0) {
    println(x)
}
```

使用多个";"可以实现多个条件的过滤:

```scala
for (x <- 0 to 100; if x > 20; if x < 80) {
    println(x)
}
```

使用for可以实现嵌套循环.

```scala
for (i <- 1 to 9; j <- 1 to i) {
    print(i * j + ",")
}
```

这打印了10以内的所有乘法的结果.

for还支持流间变量定义,可以在多循环中定义一个共享的变量.流间变量支持使用if进行赋值,例如通过下面的代码打印完整的九九乘法表:

```scala
for (i <- 1 to 9; j <- 1 to i; next = if (i == j) "\n" else "\t") {
    print(i + "*" + j + "=" + i * j + next)
}
```

for的返回值也是Unit.但是yield可以让for把每次循环最后一次返回值组装为一个集合返回.所以上面的代码可以转换为:

```scala
val res = for (i <- 1 to 9; j <- 1 to i; next = if (i == j) "\n" else "\t") yield {
    i + "*" + j + "=" + i * j + next
}

println(res)
```

这就满足了函数式编程的要求.

scala继承了java的异常机制,通过throw关键字可以抛出异常,通过try...catch...finally可以捕获异常.

catch的写法和java不同,为:

```scala
try {
    ...
} catch {
    case e: Exception1 => ...
    case e: Exception2 => ...
    ...
} finally {
    ...
}
```

try可以带有返回值.例如:

```scala
val res = try {
    ...
    "no exception"
} catch {
    case e: Exception => "have an exception"
}

println(res)
```

注意如果在finally中返回,这个返回值不会对try和catch的返回值产生影响,这点和java不一样(java的话无论如何都会返回finally的返回值).

match ... case类似于java的switch ... case.下面是示例:

```scala
val str = "abc"
val res = str match {
    case "abc" => "我是abc"
    case "bbb" => "我是bbb"
    case _ => "我既不是abc也不是bbb"
}

println(res)
```

case中不需要编写break,"_"表示任何数据.

scala中不存在continue和break.如果想实现跳出或者是继续循环,可以使用递归来实现.

## 函数

scala支持函数式编程,函数是scala中最为重要的概念.

函数是一段具有特定功能的代码的集合,由函数修饰符,函数名,参数列表,返回值声明,函数体组成.

```scala
[private | protected] [override] [final] def fun_name(param: type[,param: type]...) [:retType] = {
    ...
}
```

权限可以控制函数的可调用域,默认是public.override和final是OOP中使用的概念.

返回值类型可以省略,由scala自动推断.

如果函数体只有一行内容,则函数体的大括号可以省略.

在函数体中返回一个值可以不使用return关键字,则默认使用最后一个语句的返回作为函数的返回值.

Unit表示没有返回值,如果一个函数声明既没有返回值声明,函数体前面也没有"=",则返回值默认为Unit.

函数可以通过直接量的方式去声明,语法是: (参数列表) => {函数体}.如果函数只有一行,花括号可以省略.

如果参数只有一个,并且不产生歧义,则参数列表的小括号可以不写.

函数直接量的声明方式没有方法名,不可以直接调用.这种定义方式主要用于函数值和高阶函数.

函数是scala的一等公民,具有完整的功能:

- 成为类的成员
- 赋值给某个变量
- 作为参数传给另外一个函数
- 从一个函数中返回另外一个函数

### 函数作为类的成员

和java一样,函数可以作为自定义类的一个成员,用于描述类的某种行为.

```scala
class Person {

  def eat(food: String): Unit = {
    println("eating " + food)
  }

}

object Main {
  def main(args: Array[String]): Unit = {
    val p = new Person()
    p.eat("苹果")
  }
}
```

### 函数赋值给某个变量

函数可以像一般的值一样,赋值给一个变量(包括常量).随后可以通过变量来调用这个函数.

```scala
val add = (x: Int, y: Int) => x + y
println(add(1, 2))
```

给一个变量赋值的时候一般使用直接量.

在方法中定义方法可以达到类似的效果:

```scala
object Hello {
  def main(args: Array[String]): Unit = {
    val test = new Test
    println(test.foo())
  }
}

class Test {
  def foo(): String = {
    def hello(): String = {
      "hello, world"
    }

    hello()
  }
}
```

我们甚至可以先定义好一个函数,随后再赋值给一个变量,这需要在参数中写"_"表示参数在随后调用的额时候指定:

```scala
object Hello {

  def sum(x: Int, y: Int, z: Int): Int = {
    x + y + z
  }

  def main(args: Array[String]): Unit = {
    val sumFun = sum(_, _, _)
    println(sumFun(1, 2, 3))
  }
}
```

"val sumFun = sum(_, _, _)"可以简化为"val sumFun = sum _".

在定义函数变量的时候,可以先指定一些已经确定的参数,对于未确定的参数使用"_":

```scala
object Hello {

  def sum(x: Int, y: Int, z: Int): Int = {
    x + y + z
  }

  def main(args: Array[String]): Unit = {
    val sumFun = sum(_: Int, _: Int, 6)
    println(sumFun(2, 4))
  }
}
```

利用"_"可以实现一些非常灵活的函数调用.

### 高阶函数

高阶函数指的是函数可以在作为参数传给另外一个函数或是将一个函数从另外一个函数中返回.利用高阶函数可以实现超级灵活的编程.这类似于C语言的函数指针的功能,但是更加安全.

先看将函数作为参数使用,这需要明确声明函数的返回值和参数,这利用"(paramTypes) => (returnType)"来定义,对于小括号中的内容只需要写参数的类型即可,下面是简单的示例:

```scala
def operate2Int(x: Int, y: Int,
                opera: (Int, Int) => Int): Int = {
    opera(x, y)
}

val add = (x: Int, y: Int) => x + y

println(operate2Int(5, 1, add))
```

如果要返回一个函数,在声明返回值类型的时候也需要完整声明返回的函数的样式:

```scala
def opt2NumFun(funName: String): (Int, Int)=>Int = {
    funName match {
        case "add" => (x, y) => x + y
        case "sub" => (x, y) => x - y
        case "mul" => (x, y) => x * y
        case "div" => (x, y) => x / y
        case _ => (_, _) => 0
    }
}

println(opt2NumFun("add")(5, 1))
println(opt2NumFun("sub")(10, 5))
println(opt2NumFun("mul")(3, 10))
println(opt2NumFun("div")(30, 3))
```

返回的时候可以省略参数类型,因为在定义的时候已经声明返回函数的结构了.可以自动推断参数的类型.

在返回函数的时候,有更加简便的写法.如果你返回的函数的函数体调用的参数是按照顺序调用的并且每个参数只使用1次,可以省略参数列表声明,将所有的参数改为"_".

上面的opt2NumFun可以简化为:

```scala
def opt2NumFun(funName: String): (Int, Int)=>Int = {
    funName match {
        case "add" => _ + _
        case "sub" => _ - _
        case "mul" => _ * _
        case "div" => _ / _
        case _ => (_, _) => 0
    }
}
```

### 占位符

之前已经涉及到很多使用占位符"_"的例子了,下面做一个简单的总结.

可以在函数直接量中用"_"作为一个或多个参数的使用,从而不必再声明函数的参数列表了.

例如,add函数可以简化为:

```scala
val add = (_:Int) + (_:Int)
println(add(10, 1))
```

add仍然是一个函数,但是参数列表和返回值都不需要声明了.使用这种方式的前提是函数的所有参数都使用一次且严格按照声明的顺序使用.

"_"的类型如果可以推断出来,那么可以省略类型的声明,否则必须声明类型.

例如,下面的情况就可以自动推断出"_"的类型:

```scala
def opt3Num(func: (Int, Int, Int) => Int,
            x: Int, y: Int, z: Int): Int = {
  func(x, y, z)
}

println(opt3Num(_ + _ + _, 1, 3, 4))
println(opt3Num(_ + _ * _, 1, 2, 5))
println(opt3Num(_ / _ / _, 30, 10, 3))
```

上面的代码可以进一步简化:

```scala
val opt3Num = (_:(Int, Int, Int)=>Int)(_: Int, _: Int, _: Int)
println(opt3Num(_ + _ + _, 1, 3, 4))
println(opt3Num(_ + _ * _, 1, 2, 5))
println(opt3Num(_ / _ / _, 30, 10, 3))
```

占位符还可以在将函数传给某个变量的时候,替换所有或部分的参数列表.这个在前面已经演示过了,不再演示.

### 闭包

闭包是一个重要的概念.拥有高阶函数功能的编程语言一般都有闭包的现象.

如果一个函数在定义时,用到了外部的变量.因为是变量,函数调用的时候是不确定这个变量的值的.那么在调用函数的时候,如果用到了某个外部的变量,调用时把这个变量的值拿到函数里面,成为函数内部的值,这个过程就叫做闭包.

闭包可能带来一些问题.因为函数只有在需要外部变量的时候,才会使用闭包来获取变量的值.但是外部变量在函数调用的过程中可能会随时改变,这样如果函数在某次使用了外部变量,过了一会又使用了一次,那么这两次使用的外部变量的值是有可能不一样的.

同样,闭包可能会改变外部变量的值,这也有可能改变其它函数使用这个变量的情况,产生未定义的行为.

并且,因为函数可能持有外部变量的引用,外部变量可能迟迟不被GC释放,可能会产生内存泄漏的问题.

在开发中,应该尽量避免闭包带来的危害.比如,在函数内部引用外部的数据时,应该尽量引用常量而不是变量.

### 可变参数

在scala中,可以让最后一个参数是可重复的,从而让函数的参数是可以改变的.写法是在参数后面加一个"*".

```scala
def add(nums: Int*): Int = {
    var sum = 0
    for (num <- nums) {
        sum += num
    }
    sum
}

println(add(1, 2, 3))
println(add(4, 5, 6, 7, 8))
```

可变参数会被当做数组来处理.

### 尾递归

在Scala中应该尽量使用递归来替换循环.但是递归的效率我们知道是不高的.

如果一个递归调用仅出现在函数的最后一个表达式,那么scala可以对这个递归进行优化.因为这个递归调用不需要用到上一次调用的结果,所以在底层可以转化为一个简单的迭代,而不需要再用到压栈的行为了.

例如,计算1到100中偶数的和,大于100则结束调用,可以优化为一个尾递归:

```scala
def foo(x: Int, sum: Int): Int = {
    if (sum > 100 || x > 100) {
        sum
    }
    else if (x % 2 == 0) {
        foo(x + 1, sum)
    }
    else {
        foo(x + 1, sum + x)
    }
}
println(foo(1, 0))
```

因为每次递归都是发生在函数的最后一个语句,所以这就是一个尾递归.

## 自定义控制结构

scala使用高阶函数和科里化可以实现自定义控制结构.

高阶函数在上面介绍过了.所谓的科里化,就是把一个函数的参数列表拆分成多个参数列表的过程.

下面是一个简单的"增加并且打印"函数:

```scala
def addAndPrint(num1: Int, num2: Int, pf: (Int)=>Unit): Unit = {
    val sum = num1 + num2
    pf(sum)
}
addAndPrint(5, 7, println)
```

我们实际上可以对addAndPrint进行拆分.因为我们发现这个函数的参数列表由两类组成,一类是两个数,用于增加;另一类是一个打印函数:

```scala
def addAndPrint(num1: Int, num2: Int)(pf: (Int)=>Unit): Unit = {
    val sum = num1 + num2
    pf(sum)
}

addAndPrint(5, 7)(println)
```

addAndPrint的参数列表变成了两个.在调用的时候,也需要传两份参数才行.

当把高阶函数和科里化放在一起使用,可以构建一个自定义的控制结构.在对一个函数进行科里化后,如果某个参数只有一个值要传,就可以把小括号改写成为大括号.

例如,上面的addAndPrint,我们可以这么调用:

```scala
addAndPrint(10, 12) {
    println(_)
}
```

这就好像一个控制结构一样.其实本质就是一个高阶函数的调用.

List的foreach就是一个自定义控制器,允许我们对一个列表的每一个数字做一些行为.例如,下面的代码将列表的每一个元素加10后打印:

```scala
val li = List(1, 3, 5, 10)

li.foreach {
    (num: Int)=> {
        println(num + 10)
    }
}
```

以上代码可以简化为:

```scala
val li = List(1, 3, 5, 10)

li.foreach {
    println(_)
}
```

## OOP

Scala是一门多范式编程语言,所以除了函数式编程,scala还支持OOP.

scala类的定义和java类似.通过class可以声明类,类中可以包含属性和方法,通过new来实例化对象.所有成员和方法默认是public的,使用private和proteced可以改变权限.scala没有default权限.

scala不可以定义静态成员.因为scala认为静态成员会破坏OOP的封装性.静态成员是从对象的层面上剥离的,是过程化编程的产物,而且,多个线程抢占一个静态成员,会带来线程安全的问题.静态成员也会一直占用内存,永远不会被GC释放.

在scala中如果需要静态的功能,可以使用单例对象来实现.

### 单例对象

使用object,定义的就是一个单例对象.使用单例对象的方法不需要创建对象,使用单例对象点方法调用.效果就好像静态成员一样.

Scala会自动帮助我们实例化单例对象并且确保它在全局是唯一的,我们在调用单例对象的方法的时候可以不去明确的new.

单例对象可以单独存在,也可以绑定到一个类上.当一个单例对象和某一个类写在同一个文件中,并且名称一样,那么这二者就互相绑定了.单例对象是类的伴生对象.这二者可以互相访问对方的私有成员.

那么我们调用单例对象中的方法,不需要实例化,但是调用类中的方法,一定需要实例化对象.

例如:

```scala
class Person {
  def eat(): Unit = {
    println("吃饭饭")
  }
}

object Person {
  def sleep(): Unit = {
    println("睡觉觉")
  }
}
```

如果我们想要调用sleep()方法,可以直接调用,但是eat()方法必须实例化后调用:

```scala
Person.sleep()
val p = new Person
p.eat()
```

以上就实现了类似于静态的功能.

入口程序必须写在单例对象中.单例对象存放在一个特殊的predef包下.这个包会由scala自动导入.因此我们使用单例对象不需要额外导包.

### 类

scala中定义类的格式如下:

```scala
class class_name [(construct_list)] {
    class_body
}
```

构造参数可以省略,如果没有类体,花括号可以省略.

和java不一样,构造方法是放在类名之后的,构造函数体是直接写在类体中的.既不是属性也不是方法的地方,会被自动解释为构造函数体.

构造参数列表中的变量会被自动加到类的属性中,不需要我们像java一样再去写"this.xxx=xxx"了.这些属性默认都是private的.

下面是一个简单的示范:

```scala
object Test {
  def main(args: Array[String]): Unit = {
    val p = new Person("zhang", 20)
    p.show()
  }
}

class Person (name: String, age: Int) {

  println("construct a person")

  def show(): Unit = {
    println("name = " + name + ",age = " + age)
  }
}
```

这段代码的输出是:

```text
construct a person
name = zhang,age = 20
```

如果要提供多个构造函数,需要定义辅助构造器,这个构造器是单独定义的.定义为"def this()".scala每个辅助构造器的第一个动作,永远是调用其它构造器,要么是调用主构造器,要么调用其它辅助构造器.

也就是说主构造器无论如何都会被调用.

```scala
object Test {
  def main(args: Array[String]): Unit = {
    val p1 = new Person()
    val p2 = new Person("zhang")
  }
}

class Person (name: String, age: Int) {

  def this() {
    this("default", 0)
    println("call this()")
  }

  def this(name: String) {
    this(name, 0)
    println("call this(String)")
  }

  println("construct a person")
}
```

代码输出:

```text
construct a person
call this()
construct a person
call this(String)
```

### 重写和重载

scala支持重写,也就是覆写父类的方法,只需要在方法的定义前面加override即可.这样的语法和C#很类似.

如果方法不加上override,会报错.例如:

```scala
class Fruit {
  def eat(): Unit = {
    println("吃水果")
  }
}

class Apple extends Fruit {
  override def eat(): Unit = {
    println("吃苹果")
  }
}
```

我们可以理所当然地覆写toString方法:

```scala
object Test {
  def main(args: Array[String]): Unit = {
    val apple = new Apple
    println(apple)
  }
}

class Apple {
  override def toString: String = "一个小苹果"
}
```

重载的用法和java是类似的,定义多个同名方法即可,这里不再演示.

### 无参方法

在scala中,无参方法有点特殊.一个无参方法实际上可以是一个属性,一个属性也可以是一个无参方法.

```scala
object Test {
  def main(args: Array[String]): Unit = {
    val p = new Person
    println(p.say1)
    println(p.say2())
    println(p.say3)

  }
}

class Person {
  def say1 = "LOL"

  // 以上写法和以下一样
  def say2(): String = {
    "LOL"
  }

  // 也和下面的一样
  val say3 = "LOL"
}
```

如果一个方法是无参的,那么对于外部来说,它和一个一般的属性是一样的.所以上面打印了3次LOL.其实,在scala中,属性也是作为函数存在的.上述的say3实际上会被转换为say1的写法.

这也验证了,在scala中,万物皆为函数.

在定义无参函数的时候,如果省略小括号,调用的时候就不能省略小括号.如果省略了,就可以省略小括号.

一般使用以下策略决定是否对无参方法加上小括号:

- 如果方法是封闭的,不会对外部产生影响,那么最好省略小括号.这样外部调用者可以把它当做属性处理,简化编程模型.
- 如果方法不是封闭的,对外部产生了影响,例如读取了文件等,最好加上小括号.因为不加的话会让人产生疑惑:仅仅是使用了一个属性却导致了外部环境的改变.

### 包

类似java,scala允许将类分开存放,以实现区分命名空间的功能.可以像java一样,使用package关键字指定当前代码所在的包.

但是,scala的包和类真正存放的位置没有什么关系,所以这里的包的概念其实和java不一样,和C++和C#的namespace的概念倒是很像.但是编译后的class文件会真正存放在包的文件夹下.

那么和C++和C#一样,scala允许在同一个文件中声明多个包:

```scala
package cn {
    package lazycat {
        // 此处的包为cn.lazycat
    }

    package hi {
        // 此处的包为cn.hi
    }
}
```

但是一般不建议这么写,会导致包的管理非常混乱.还是建议在一个文件中只声明一个包.

要想引入包,使用import即可.和java不同,import可以出现在代码的任何位置(java仅允许import出现在代码的前面)

如果要引用一个包下的所有东西,使用"import xx.xx._"即可.我们甚至可以使用一个import来批量引入一些包:

```scala
import java.util.{List, Map}
```

这比java方便,java需要编写多条import才能完成多个成员的导入.

我们甚至可以为导入的一些类起一些别名:

```scala
import java.util.{SimpleDataFormat => Sdf}
```

这样在下面实例化这个类对象的时候就可以使用别名替换了:

```scala
val s: Sdf = null
```

### 额外访问权限

除了public,private,proteced,scala提供更加精细的权限控制,我们可以在权限声明后面增加一个中括号,里面加一些额外允许访问的包名(很像C++中的友元),例如:

```scala
private [cn.hhh] val f: String
```

那么在cn.hhh包中,即使不在这个类里面,也可以直接访问f属性了.

### 抽象类

和java一样,scala允许定义抽象类,抽象类的方法允许不进行实现,而且抽象类不允许实例化,这些都是OOP的标配了.

```scala
abstract class Fruit {
  def eat()  // 这是一个抽象方法,子类如果不是抽象类,必须实现
  def get(): Unit = println("得到了一个水果!")
}

class Apple extends Fruit {
  override def eat(): Unit = println("吃了一个苹果")
}
```

### 调用父类构造

子类如果继承了父类,需要调用父类的构造,这需要在继承的时候就声明好,可以使用子类的构造去初始化,例如:

```scala
object Test {
  def main(args: Array[String]): Unit = {
    val r = new Teacher("Tang")
    println(r)
  }
}

class Person (name: String)

class Teacher (name:String) extends Person(name) {
  override def toString: String = "teacher's name = " + name
}
```

### 多态

多态和java是一样的,通过向上转型来完成.例如:

```scala
object Test {
  def main(args: Array[String]): Unit = {
    val f1: Fruit = new Apple
    val f2: Fruit = new Orange
    f1.eat()
    f2.eat()
  }
}

abstract class Fruit {
  def eat()
}

class Apple extends Fruit {
  override def eat(): Unit = println("吃苹果")
}

class Orange extends Fruit {
  override def eat(): Unit = println("吃橘子")
}
```

### final

如果在定义类的前面加上final,那么这个类就不能被继承了:

```scala
final class class_name {
    ...
}
```

用在方法上,表示方法不能被覆写,这个和java是一样的.

### 继承结构

我们知道,java中的所有类都继承自Object类.

在Scala中,所有类都继承自Any类.其中包含了如下方法:

- final def ==(that: Any): Boolean
- final def !=(that: Any): Boolean
- def equals(that: Any): Boolean
- def hashCode: Int
- def toString: String

这五个方法在java中都可以找到对应的.

Any有两个直接子类,AnyValue和AnyRef.AnyValue的子类是之前介绍过的除了String的8个基本类型加上一个Unit.这些类型不能new,只能使用直接量;除了这9个类,其它所有类都来自于AnyRef.所以AnyRef才相当于java的Object类.

scala中还有一个scala.Null,所有引用类型都是这个类型的祖先.于是以下的写法:

```scala
val p: Person = null
```

就可以用OOP的向上转型解释通了(因为null必定是Person的祖先).

但是,除了String的8个基本类型不可以使用null赋值:

```scala
val i: Int = null  // 错误!!!
```

因为Null并不是Int的祖先.

scala还有一个scala.Nothing,表示没有任何值.当程序抛出异常后,会返回这个玩意.它是所有类型的子孙.

Scala的继承结构树可以用下图表示:

![外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传](https://img-home.csdnimg.cn/images/20230724024159.png?origin_url=https%3A%2F%2Fwx3.sinaimg.cn%2Fmw690%2F0060lm7Tly1fqkoc5evekj30ja0fnt8y.jpg&pos_id=img-H67ikBJl-1704343520843)

### Trait

特质,可以类比Java的接口,但是和接口的区别是比较大的.在Scala中,类不能说是实现了Trait,而是说类混入了Trait.

特质可以包含抽象方法和非抽象方法.但是和抽象类不一样,子类可以混入多个特质;抽象类子类只能继承一个.

使用with,可以混入特质.特质一般让类符合一些特征.要想混入特质,必须现有extends,否则报错.这是scala让你优先用抽象类,抽象类满足不了你了,才让你用特质.例如:

```scala
trait Eatable {
  def eat()
}

trait Walkable {
  def walk(): Unit = println("走路!")
}

abstract class Person {
  def talk()
}

class Student extends Person with Eatable with Walkable {
  override def talk(): Unit = println("学生说话啦!")
  override def eat(): Unit = println("学生吃饭啦!")
}

object Test {
  def main(args: Array[String]): Unit = {
    val student: Student = new Student
    val eat: Eatable = student
    val walk: Walkable = student
    val person: Person = student

    eat.eat()
    walk.walk()
    person.talk()
  }
}
```

这个特质有点像多继承了.因为特质可以实现具体的方法.

如果混入的多个特质中有同名的具体方法,则在混入后可能冲突,子类就必须覆写这个方法.

## 集合

### 数组

一种长度固定,线性的数据结构.Scala的底层就是由java的数组实现的.

Scala提供两种数组:Array和ArrayBuffer.其中,前者的长度是固定不变的,后者的长度是可以改变的,可以增加,删除一些数据,但是效率较低.

数组的常见操作如下:

```scala
// 定义一个长度为3的数组
val arr1 = new Array[String](3)

// 直接初始化数组
val arr2 = Array[String]("abc", "123", "456")

// 不声明泛型,可以存放不同类型的数据
val arr3 = Array("a", 1, 3, 9.0, "b")

// 访问数组使用小括号
println(arr3(2))

// 遍历数组
for (str <- arr2) {
    println(str)
}

// 修改元素
arr1(1) = "niubi"
println(arr1(1))
```

Array有一些很有用的API,比如,concat可以拼接两个数组,foreach可以对数组的每个元素做一些事情:

```scala
val arr1 = Array[Int](1, 2, 3, 4)
val arr2 = Array[Int](5, 6, 7)

val arr3 = Array.concat(arr1, arr2)
arr3.foreach {
    println(_)
}
```

有很多其它API,大家可以自行查阅文档.

### 列表

List有List和ListBuffer,其中第一个是不可变的,第二个是可变的.

定义List有以下的写法:

```scala
val list1 = List(2, 3, 5)
val list2 = "a" :: "b" :: "c" :: "d" :: "e" :: Nil
```

关心第二种写法,是按照顺序拼接出一个列表,最后一个必须是Nil."::"实际上是在列表的第一个位置插入一个元素.但是操作数是右值.

以下常见方法可以操作List的元素:

```scala
val list1 = List(2, 3, 5)
val list2 = "a" :: "b" :: "c" :: "d" :: "e" :: Nil

println(list2(2))
println(list2.head)
println(list2.tail)
println(list1.sum)

// 连接链表
val list3 = list1 ::: list2
println(list3)

// 向后插入元素
val list4 = list2 :+ "f"
println(list4)

// 向前插入元素
val list5 = "z" +: list2
println(list5)

// 删除元素
val list6 = list2.drop(1)
println(list6)

// 修改元素
val list7 = list2.updated(2, "a")
println(list7)
```

和java不同,对于List的修改是通过返回另外一个List实现的.也就是List本身的元素是不可以改变的.

### 可变列表和数组

以上的Array和List并没有改变本身,而是返回一个新的集合,非常浪费空间.

ArrayBuffer和ListBuffer可以改变,并且可以转换为Array和List.

```scala
val buffer = ArrayBuffer(4, 1, 4)
buffer.append(10, 20)

val arr = buffer.toArray
println(arr)
```

ListBuffer的用法是一样的.

### Range

Range表示一个数据的范围.使用to运算符可以构造,range可以转换为array或list:

```scala
val r = 1 to 10
val list = r.toList

println(list)
```

### Vector

Scala为了提高List的随机读取效率,引入的新的集合.

Vector的用法和List几乎一模一样.

### Set

Set的特点是储存的元素不允许重复并且是无序的.用法和List差不多.

### Map

Map用于保存键值对.Map的元素不允许改变,常见操作如下:

```scala
// 创建Map
val m1 = Map(1 -> "zhang", 2 -> "Li", 3 -> "Wang")

// 追加元素
val m2 = m1 + (4 -> "Zhao")
println(m2)

// 删除元素
val m3 = m1 - 2
println(m3)

// 获取元素
println(m1.getOrElse(1, null))

// 遍历Map
for (key <- m1.keySet) {
    println(key + "," + m1.getOrElse(key, null))
}

// 或
val ite = m1.iterator
for (t <- ite) {
    println(t._1 + "," + t._2)
}

// 或
m1.foreach(t => println(t._1 + "," + t._2))
```

通过zip可以生成Map:

```scala
val l1 = List(1, 2, 3)
val l2 = List("a", "b", "c")

val map = l1.zip(l2).toMap

println(map)
```

这可以把其它List组合成一个Map.

### Tuple

代表键值对,一个键可以有多个值.通过小括号就可以声明一个Tuple,通过"_#"可以获取值:

```scala
val t = (1, "a", "b", 200)
println(t._1)
println(t._2)
println(t._3)
println(t._4)
```

Tuple最多储存22个字段.

### 集合通用方法

基本集合对象都包含以下方法(以下T表示集合储存元素的类型):

- exist(T => Boolean): 给一个函数,每次接收集合中的值,返回true表示元素存在,false表示元素不存在.所有元素只要有一个返回true,整个函数返回true.
- sorted: 对元素进行排序后返回,按照升序排序.
- sortWith((T, T) => Boolean): 自己指定排序的规则.
- distinct: 去重.
- reverse: 反转整个集合.
- reverseMap(T => Unit): 表示对元素进行一些处理后反转.
- contains(T): 是否包含某个元素.
- startsWith(Seq[T]): 是否以某个集合开头.
- endsWith(Seq[T]): 是否以某个集合结尾.

另外有一些集合中的通用高阶函数:

- partition(T => Boolean): 根据某个特征对集合进行拆分.
- map(T => Unit): 把函数作用于每个元素得到一个新的集合.
- filter(T => Boolean): 按照某个条件对集合进行过滤.
- filterNot(T => Boolean): 过滤不符合条件的元素.
- reduce((T, T) => T): 将集合中的元素两两作用于一个函数,最终返回一个值.
- par: 允许对一个集合中的元素计算时使用多线程.
- groupBy(T => Any): 分组,将每个元素作用于函数,返回一样的分为一组.

## 其它特性

### 泛型

Scala的泛型和Java的一模一样,唯一的区别是Scala使用的是中括号.

### lazy

通常情况下,使用val或var声明的属性,会直接分配空间.但是如果这个变量在很后面才会被使用,那么空间一直被占用着,很是浪费.

我们可以在定义变量的时候在前面加一个lazy,这样知道用到这个变量才会为这个变量分配空间:

```scala
lazy val str = "hello"
...           // 此处没有分配空间
println(str)  // 此处为str分配了空间
```

### Option

Option在scala中可以表示为一个结果,它可以有值也可以没有值.它有两个实现:Some和None.Some表示返回一些值,None表示没有值.

Option需要接收一个泛型,表示返回的值是什么类型的.

例如,实现一个除法,如果被除数为0,返回None:

```scala
def div(x: Double, y: Double): Option[Double] = {
    if (y == 0) {
        None
    }
    else {
        Some(x / y)
    }
}

println(div(20.0, 10.0).getOrElse(0.0))
println(div(10.0, 0).getOrElse(0.0))
```

getOrElse()方法接收一个默认值,如果返回的是None,则返回这个默认值;如果返回的是Some,就返回Some的值.

这样就避免了一些异常处理.

### 样例类

样例类是一种特殊的类.这种类必须有构造参数,并且默认实现了序列化接口,默认覆盖了toString,equals,hashCode方法,并且不需要new就可以直接生成对象.

例如:

```scala
object Test {
  def main(args: Array[String]): Unit = {
    val p: Person = Person("zhang")
    println(p.getName)
  }
}

case class Person (name: String) {
  def getName: String = name
}
```

不使用new,则使用Scala内部的工厂来实例化对象.这个类的作用是加快java bean类的编写.
