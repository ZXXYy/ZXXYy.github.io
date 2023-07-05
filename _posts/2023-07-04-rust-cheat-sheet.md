---
title:      "Rust Cheat Sheet in Practice"
date:       2023-07-04 09:00:00
author:     zxy
math: true
categories: ["Coding", "Rust"]
tags: ["Rust"]
post: true
---
这篇博客主要记录一下，在实际的Rust编程过程中，会涉及到的一些Rust语言相关知识。

### 1. `()` [unit type](https://doc.rust-lang.org/std/primitive.unit.html#)

`()`类似C语言中的void
`fn func(){}`和`fn func()->(){}`是一样的
分号`;`可以丢弃块末尾的表达式结果，使表达式（以及块）计算为 ()

```rust
fn returns_i64() -> i64 {
    1i64
}
let is_i64 = {
    returns_i64()
};
let is_unit = {
    returns_i64();
};
```

### 2.  [Option type](https://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/std/option/enum.Option.html) vs. [Result type](https://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/std/result/index.html)

```rust
// Rust中使用Option类型来处理缺失值
enum Option<T> {
    None,
    Some(T),
}
```

```rust
//Result是加强版Option，可以用来处理错误值--unexpected value（缺失值可以理解为特殊的一种错误值）
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Option`和`Result`的关系相当于`type Option<T> = Result<T, ()>;`
例子:

```rust
fn double_number(number_str: &str) -> Result<i32, ParseIntError> {
    match number_str.parse::<i32>() {
        Ok(n) => Ok(2 * n),
        Err(err) => Err(err),
    }
}
```

### 3. `unwrap` vs. `expect` vs. `?`

这个知识点涉及Rust中的error handling，要明白unwrap，expect和“?”操作符号，需要知道Rust中的两个类型`Option`和`Result`（见2）

| Method | Behaviour                                                    |
| ------ | :----------------------------------------------------------- |
| unwrap | 返回计算结果，如果出现error，就通过`panic`终止程序           |
| expect | 返回计算结果，如果出现error，就通过`panic`终止程序，且可以自定义panic message |
| ?      | 返回计算结果，如果出现error，返回/传递error而不是终止程序    |

通过一个例子来说明

```rust
// File::open returns Result: Ok(file) or Err(error)
// Unwrap means:
// - "if result is Ok: store value inside enum in `file`
// - "if result is Err (opening file failed): panic (crash program)"
// Panic if opening a file fails:
let mut file = File::open(filename).unwrap();
// expect is the same as unwrap but allows you to print a
// more descriptive error message when panicking.
let mut file = File: :open(filename).expect ("Failed to open file");
// ? means:
// - "if result is Ok: store value inside enum in `file`
// - "if result is Err (opening file failed): return Err(error) to caller"
let mut file = File::open(filename)?;
```

对于`Option`类型来说，默认的`unwrap`相当于

```rust
impl<T> Option<T> {
    fn unwrap(self) -> T {
        match self {
            Option::Some(val) => val,
            Option::None =>
            panic!("called `Option::unwrap()` on a `None` value"),
        }
    }
}
```
`unwrap_or`在Rust标准库中表示，出现缺失值时，把缺失值设置为默认值

对于`Result`类型来说，默认的`unwrap`相当于

```rust
impl<T, E: ::std::fmt::Debug> Result<T, E> {
    fn unwrap(self) -> T {
        match self {
            Result::Ok(val) => val,
            Result::Err(err) =>
            panic!("called `Result::unwrap()` on an `Err` value: {:?}", err),
        }
    }
}
```

### 4. `as_ref()` vs. `&` reference

|          | 类型                                           | 功能                                                         |
| -------- | ---------------------------------------------- | ------------------------------------------------------------ |
| as_ref() | `Option::as_ref` method                        | 把`&Option<T>` 转换为 `Option<&T>`                           |
|          | `Result::as_ref` method                        | 把`&Result<T,E>`转化为` Result<&T, &E>`                      |
|          | 对`Box`                                        | 把`Box<T>`转化为`&T`                                         |
|          | `AsRef::<Target>::as_ref` trait implementation | 如果类型`U`实现了`AsRef<T>`，则`as_ref`可以实现`&U`到`&T`的转换 |
| &        | 对`Option`                                     | 把`Option<T>` 转换为 `&Option<T>`                            |
|          | 对`Result`                                     | 把`Result<T,E>`转化为` &Result<T,E>`                         |
|          | 对`Box`                                        | 把`Box<T>`转化为`&T`                                         |
|          | 取地址运算符                                   | 只能将类型`T`转换为其自身的引用类型`&T`                      |

> Further reading:
>
> 1. [Familiarize yourself with reference and pointer types](https://www.lurklurk.org/effective-rust/references.html)
>
> 2. [Borrow and AsRef](https://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/first-edition/borrow-and-asref.html#borrow-and-asref)

### 5. `&str` vs. `String`, `&[]` vs. `Vec`, `Path` vs. `PathBuf`

### 6. as vs. from/Into

