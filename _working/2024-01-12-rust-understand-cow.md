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
