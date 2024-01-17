---
layout:       post
title:        "Rust错误处理: anyhow和thiserror的使用"
subtitle:     "Rust error handling: The usage of anyhow and thiserror"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Rust
---

Rust的错误处理是我目前用过的所有语言中最优雅的，它提供了很多语法糖能让我们写出高效的错误处理代码。

我不喜欢Java、Python等语言的`try catch`方式，不仅因为它会对代码造成很多侵入，而且容易导致用户忽略很多重要的错误（无脑throw），增加对错误处理精细化管控的成本。

同样不喜欢Go的`if err != nil`，因为你的Go代码中到处都是它的身影。

Rust的错误处理基本逻辑跟Go类似，都是直接将函数的错误直接返回。但是Rust有强大的enum以及语法糖、宏定义、元组做支撑，语言表达能力比Go强了不是一个量级。因此Rust的错误处理完美解决了Go的代码冗余问题。

Rust错误处理有两个明星级别的库，[anyhow](https://github.com/dtolnay/anyhow)和[thiserror](https://github.com/dtolnay/thiserror)。它们分别用于应用中的错误处理以及库的自定义错误。本文将分别介绍它们的使用。

## anyhow

如果你是一位Rust应用程序开发者，anyhow是错误处理的很好选择。这个库的特点正如它的名字这样，「无所谓」。我们不关心错误类型到底是什么，只希望错误信息能够完整地展示给用户或开发者，以方便地进行debug。

很多Rust的经典项目，例如[Cargo](https://github.com/rust-lang/cargo)，都使用了anyhow。

默认情况下，Rust的`Result`需要提供两个泛型，分别表示成功的返回值和失败的返回值。anyhow定义了另一种`Result`，即`anyhow::Result`：

```rust
pub type Result<T, E = Error> = core::result::Result<T, E>;
```

可以看到，它重新定义了标准Result，将错误类型指定为`anyhow::Error`，这是anyhow为我们定义的错误类型，这个错误能展示丰富的错误信息。并且anyhow为我们提供了很多方法来生成`anyhow::Error`，或将别的错误转换为`anyhow::Error`。

这样，我们的任何函数都可以返回这个简化版的Result，就不需要每次都指定错误类型了：

```rust
use anyhow::Result;

fn foo() -> Result<()> {
    ...
}
```

对比没有anyhow的情况还是要简单了很多：

```rust
fn foo() -> Result<(), &'static str> {
    ...
}
```

那么我们该如何构建错误呢？anyhow为我们提供了一些宏来快速实现：

- `anyhow!`：将别的类型转换为`anyhow::Error`。支持format string。
- `bail!`：将别的类型转换为`anyhow::Error`并返回函数，是`return Err(anyhow!(...))`的语法糖。

用起来非常简单，例如：

```rust
use anyhow::{bail, Result};

fn divide(a: i32, b: i32) -> Result<i32> {
    if b == 0 {
        bail!("The b value {b} cannot be zero");
        // 或：return Err(anyhow!(...));
    }
    Ok(a / b)
}
```

可以看到，这样我们就能快速让函数返回错误信息。

另外一种情况是，在函数中可以调用其他函数，这些函数返回了别的类型的错误，需要将它们转换为`anyhow::Error`：

```rust
use std::fs;
use std::path::Path;

use anyhow::{bail, Result};

fn read_file(path: impl AsRef<Path>) -> Result<Vec<u8>> {
    let data = match fs::read(path) {
        Ok(data) => data,
        Err(err) => bail!(err),
    };
    // 或者：let data = fs::read(path)?;

    // 处理data...
    Ok(data)
}
```

在返回错误时，我们一般希望加上一些上下文信息，这时可以利用`anyhow::Context`来实现：

```rust
use std::fs;
use std::path::Path;

use anyhow::{Context, Result};

fn read(path: impl AsRef<Path>) -> Result<Vec<u8>> {
    let data = fs::read(path.as_ref())
        .with_context(|| format!("read file '{}' failed", path.as_ref().to_str().unwrap()))?;
    // 处理data...
    Ok(data)
}
```

`Context`可以自动将别的类型的错误转换为`anyhow::Error`，并且在原来的基础上增加上下文信息，我们可以调用这个函数并把错误打印出来：

```rust
fn main() -> Result<(), anyhow::Error> {
    read("hello.txt")?;
    Ok(())
}
```

输出：

```
Error: read file 'hello.txt' failed

Caused by:
    No such file or directory (os error 2)
```

输出的`Caused by`会包含错误的上下文层级，以更好地追踪错误。这里可以很清晰地看到读取`hello.txt`文件失败的原因是这个文件不存在。

我们在使用anyhow时，如果调用别的函数返回了错误，使用`Context`为错误增加上下文信息是一个很好的习惯。

另外，上面我们用的是`with_context`，传入一个闭包函数以支持format字符串。如果仅仅使用一个static字符串作为上下文信息，可以使用`context`函数：

```rust
use std::fs;
use std::path::Path;

use anyhow::{Context, Result};

fn read(path: impl AsRef<Path>) -> Result<Vec<u8>> {
    let data = fs::read(path.as_ref()).context("read file failed")?;
    // 处理data...
    Ok(data)
}
```

以上就是anyhow的基本使用了。如果你希望开发一个包含`main.rs`的Rust应用程序，用anyhow来做错误处理是一个很好的选择，这样你就不需要费尽心思来定义自己的错误以及编写各种代码来生成以及展示错误了。

## thiserror

如果你是Rust库的开发者，希望自定义错误，那么thiserror就会是你的好伙伴。这个库很简单，提供了一些宏定义来帮助你快速定义自己的错误类型。

在Rust中，自定义错误一般是一个enum，表示不同的错误类型。thiserror提供了很多宏定义，能让我们方便地根据字符串描述来生成错误，并为错误自动实现`Display`以方便进行展示。

一个简单的例子：

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DataStoreError {
    #[error("data store disconnected")]
    Disconnect(#[from] io::Error),
    #[error("the data for key `{0}` is not available")]
    Redaction(String),
    #[error("invalid header (expected {expected:?}, found {found:?})")]
    InvalidHeader {
        expected: String,
        found: String,
    },
    #[error("unknown data store error")]
    Unknown,
}
```

可以看到，thiserror支持：

- 使用`#[from]`从其他错误类型继承。
- 通过`#[error]`定义错误描述，支持format string。format value来自于enum成员字段，可以是`{0}`这样的匿名字段或`{field:?}`这样的具名字段。

`#[error]`支持一些高级用法，例如，在enum成员中指定结构体：

```rust
#[derive(Debug)]
struct Limit {
    lo: usize,
    hi: usize,
}

#[derive(Debug, Error)]
enum Error {
    #[error("out of bound, expect index in [{}, {}], found {idx:?}", .limit.lo, .limit.hi)]
    OutOfBound { idx: usize, limit: Limit },
}
```

在使用的时候，可以用`.{enum_field}.{field}`的方式来访问结构体中的字段。

甚至可以调用函数：

```rust
fn first_char(s: &String) -> char {
    s.chars().next().unwrap_or(' ')
}

#[derive(Debug, Error)]
enum Error {
    #[error("first letter must be lowercase but was {:?}", first_char(.0))]
    WrongCase(String),
}
```

这样，通过thiserror，我们就不用手动去实现自定义错误的`Display`了。外部可以很方便地展示错误，下面是一个简单的使用例子：

```rust
use thiserror::Error;

fn first_char(s: &str) -> char {
    s.chars().next().unwrap_or(' ')
}

#[derive(Debug)]
struct Limit {
    lo: usize,
    hi: usize,
}

#[derive(Debug, Error)]
enum Error {
    #[error("the first letter of name must be lowercase but was {:?}", first_char(.0))]
    WrongCase(String),

    #[error("index {0} out of bound, expect in [{}, {}]", .1.lo, .1.hi)]
    OutOfBound(usize, Limit),

    #[error("the name is empty")]
    Empty,
}

fn get_name<'a>(names: &[&'a str], idx: usize) -> Result<&'a str, Error> {
    if idx > names.len() - 1 {
        let limit = Limit {
            lo: 0,
            hi: names.len() - 1,
        };
        return Err(Error::OutOfBound(idx, limit));
    }

    let name = names[idx];
    if name.is_empty() {
        return Err(Error::Empty);
    }

    let first = first_char(name);
    if first.is_uppercase() {
        return Err(Error::WrongCase(name.to_string()));
    }

    Ok(name)
}
```

这样就实现了一个简单的自定义错误的场景，使用上来说也非常简单，可以用到我们上面的anyhow：

```rust
fn main() -> Result<(), anyhow::Error> {
    let names = vec!["jack", "john", "rust", "Peter", ""];

    get_name(&names, 100)?; // Error: index 100 out of bound, expect in [0, 4]
    get_name(&names, 3)?; // Error: the first letter of name must be lowercase but was 'P'
    get_name(&names, 4)?; // Error: the name is empty

    Ok(())
}
```

可以看到，有了anyhow和thiserror为我们预定义的一些宏，Rust的错误处理变得非常优雅！