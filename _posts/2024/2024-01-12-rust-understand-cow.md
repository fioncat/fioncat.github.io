---
layout:       post
title:        "Rust: 理解Cow智能指针"
subtitle:     "Rust: Understand the Cow smart pointer"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Rust
---

`Cow`并没有像`Box`, `Rc`这样出现在[The Rust Book](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html)中，但它是个同样重要的智能指针。阅读源码，你会发现很多追求高效的Rust项目都大量使用了`Cow`。今天我们就来简单看一下Cow的用法以及应用场景。

## 为什么需要Cow

Cow的全称是`Copy On Write`，即写时复制，它即能够持有引用也可以持有对象本身，并且仅在需要的时候进行`clone`，可以减少很多没有必要的内存分配和`clone`操作。

那么我们为什么需要它呢？来看个简单的场景：

- 判断某个字符串是否有`hello_`前缀。
- 如果有，则直接创建一个新的`String`。
- 如果没有，则将前缀追加到字符串前面，返回一个新的`String`。

这个简单的函数相信你一下就能写出来：

```rust
fn append_prefix(prefix: &str, s: &str) -> String {
    match s.starts_with(prefix) {
        true => s.to_string(),  // 没有必要的堆内存分配
        false => format!("{prefix}{s}"),
    }
}
```

观察这个函数，实际上不管`s`有没有前缀，函数都会创建一个新的字符串，这样的效率显然不是最优的。事实上如果`s`已经包含了前缀，我们完全可以复用`s`，没必要额外创建一个`String`。

也就是说，一个更加理想的`append_prefix`应该是：

- 如果字符串已经有了`hello_`前缀，直接返回这个字符串引用。
- 如果字符串没有前缀，创建新的字符串。

我们可以在尝试没有`Cow`的情况下来实现，这可能需要定义一个`enum`来分别表示两种情况：

```rust
#[derive(Debug)]
enum PrefixString<'a> {
    WithPrefix(&'a str),
    NoPrefix(String),
}

fn append_prefix<'a>(prefix: &str, s: &'a str) -> PrefixString<'a> {
    match s.starts_with(prefix) {
        true => PrefixString::WithPrefix(s),
        false => PrefixString::NoPrefix(format!("{prefix}{s}")),
    }
}
```

这样就能减少掉`s`已经包含了前缀的情况下的堆内存分配开销。

现在我们来转向该函数的使用者，注意函数返回的`enum`中包括了对`s`的不可变引用。如果我们想在函数外部修改返回值，则还是需要使用`s`创建`String`：

```rust
let s = append_prefix("hello", "hello_test");
println!("s is: {:?}", s); // s is: WithPrefix("hello_test")

// 如果我们想对s进行更改，还是需要创建String对象。
let mut s_mut = match s {
    PrefixString::WithPrefix(s) => String::from(s),
    PrefixString::NoPrefix(s) => s,
};
// 现在可以对s进行更改了
s_mut.push_str("_suffix");

println!("after s is: {s_mut}"); // after s is: hello_test_suffix
```

这实际上就是一个手动挡的`Cow`，还好，Rust已经帮我们把这些细节隐藏了，我们可以直接用Cow来实现。

## Cow介绍

我们来看Cow的定义：

```rust
pub enum Cow<'a, B: ?Sized + 'a>
where
    B: ToOwned,
{
    /// Borrowed data.
    Borrowed(&'a B),

    /// Owned data.
    Owned(<B as  ToOwned>::Owned),
}
```

其实它跟我们上面自己定义的`enum`有点类似，但是是种更加通用的、自动的实现。Cow可以保存一个对象本身或不可变引用，并且在发生写入行为时，自动`clone`引用以防止对原始变量的修改。

具体来说，Cow的两个成员分别表示：

- `Cow::Borrowed`：表示对某个对象的不可变引用。注意它的生命周期需要跟原始对象一致。
- `Cow::Owned`：表示直接持有某个对象。

在对`Cow`进行写入行为时：

- `Cow::Borrowed`：对引用进行`clone`，创建一个新的对象，并将类型改为`Owned`，在这个新的对象上面进行写入操作。
- `Cow::Owned`：直接在持有对象上面进行写入操作。

注意`Cow`的`trait`是`ToOwned`而不是`Clone`（`ToOwned`比`Clone`更加通用，可以实现不同类型的clone，例如`&str`到`String`）。在写入时，`Borrowd`到`Owned`的转变是通过`to_owned`来完成的。

我们可以用`Vec`来简单验证一下`Cow`的特性：

```rust
let vec = vec!["hello", "cow"];
let mut vec_cow = Cow::Borrowed(&vec);

// 在这里，因为`vec_cow`还没有发生写入行为，因此`vec_cow`持有的
// 还是对`vec`的不可变引用。我们可以验证一下
println!("addr for source: {:p}", &vec[0]); // addr for source: 0x55ea8b0ceae0
println!("addr for cow: {:p}", &vec_cow[0]); // addr for cow: 0x55ea8b0ceae0

// 对cow进行更改，此时会发生clone
vec_cow.to_mut().push("new_string");

// 此时Cow的类型变为了Owned
if let Cow::Borrowed(_) = vec_cow {
    panic!("unexpect cow type");
}

// 地址也发生了改变
println!("addr for source: {:p}", &vec[0]); // addr for source: 0x55ea8b0ceae0
println!("addr for cow: {:p}", &vec_cow[0]); // addr for cow: 0x55709e97dba0
```

## Cow的使用

Cow的使用还是比较简单的。我们可以直接用Cow来实现[第一节](#为什么需要cow)的`append_prefix`函数：

```rust
use std::borrow::Cow;

// 注意，Cow需要持有的是s的引用而不是prefix的。因此需要手动指定'a。
fn append_prefix<'a>(prefix: &str, s: &'a str) -> Cow<'a, str> {
    match s.starts_with(prefix) {
        true => Cow::Borrowed(s),
        false => Cow::Owned(format!("{prefix}{s}")),
    }
}
```

使用时，可以直接用`to_owned`来将Cow转换为String，效果跟我们自己实现是一样的，Rust内部会自动判断是否需要进行clone操作：

```rust
let s = append_prefix("hello_", "test");
println!("s is: {s}"); // s is: hello_test

let mut str = s.into_owned();
str.push_str("_suffix");

println!("after s is: {s_mut}"); // after s is: hello_test_suffix
```

## 应用场景

Cow在很多场景下都不是必须的，你完全可以使用`clone`来规避掉所有的Cow场景。Cow更多是对内存的更进一步精细化控制。

以我自己的经验来说，如果你的变量仅可能持有对象或引用，那么就没必要使用Cow，例如：

```rust
let mut vec = Vec::new();  // vec只可能持有引用
vec.push("hello");
vec.push("i love rust");

let mut vec2 = Vec::new(); // vec只可能持有对象
vec.push(String::from("hello"));
vec.push(String::from("i love rust"));
```

但是，一旦你的变量可能会持有对象或引用，就可以使用Cow来优化掉没有必要的内存分配：

```rust
let mut vec = Vec::new();  // vec可能持有引用或对象
vec.push(Cow::Borrowed("hello"));  // 持有引用
vec.push(Cow::Owned(String::from("i love rust")));  // 持有对象
```

或者是，你的函数可能会创建新的对象，或返回引用，类似我们上面的`append_prefix`例子。

结构体字段也可以用Cow来优化，原则跟上面类似，如果你的某个字段即可能持有引用，也可能持有对象，就应该上Cow，例如：

```rust
use std::borrow::Cow;

struct MyStruct<'a> {
    name: Cow<'a, str>,
}

fn main() {
    let s1 = MyStruct {
        name: Cow::Borrowed("s1"), // 持有引用
    };
    let s2 = MyStruct {
        name: Cow::Owned(String::from("s2")), // 持有对象
    };
}
```

> 总结：如果你的变量既可能持有对象，也可能持有引用，就应该考虑使用`Cow`。