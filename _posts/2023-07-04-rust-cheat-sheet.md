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

https://www.lurklurk.org/effective-rust/casts.html

### 7. pub extern "C" & \#[no_mangle]

通过 `extern`来说明跨语言的函数调用

**提供Rust函数给其他语言调用**

```rust
#[no_mangle]
pub extern "C" fn call_from_c() {
    println!("Just called a Rust function from C!");
}
```

`#[no_mangle]` 告诉Rust编译器不要混淆函数名，以便在C语言中调用时可以找到对应的函数。

mangle是指Rust编译器会改变函数的名字，在其中加入更多的信息变得不可读。

比如`dragonos_kernel::syscall::Syscall::handle ()`这样一个函数名会被编译器编译为`_ZN15dragonos_kernel7syscall7Syscall6handle17hd25633fe74647baeE`

**在Rust中调用其他语言函数**

```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```

### 8. core.rs vs. lib.rs vs. mod.rs vs. main.rs

If a package contains `src/main.rs` and `src/lib.rs`, it has two crates: a library and a binary, both with the same name as the package.

### 9. 智能指针--Rc, Arc, Weak

| Rust 规则                            | 智能指针带来的额外规则                  |
| ------------------------------------ | --------------------------------------- |
| 一个数据只有一个所有者               | `Rc/Arc`让一个数据可以拥有多个所有者    |
| 要么多个不可变借用，要么一个可变借用 | `RefCell`实现编译期可变、不可变引用共存 |
| 违背规则导致**编译错误**             | 违背规则导致**运行时`panic`**           |

智能指针往往都实现了 `Deref` 和 `Drop` 特征

- `Deref`可以让智能指针像引用那样工作，这样你就可以写出同时支持智能指针和引用的代码，例如 `*T`
- `Drop` 允许你指定智能指针超出作用域后自动执行的代码，例如做一些数据清除等收尾工作

#### **Box\<T>**

 将一个值分配到堆上

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}
fn main() {
    let a = Box::new(3);
    println!("a = {}", a); // a = 3
}
```

#### **Rc\<T>** 

reference counting，即统计指向堆上变量的指针数量。允许一个数据资源在同一时刻拥有多个所有者，适用于单线程，指向底层数据的不可变的引用，因此你无法通过它来修改数据

Rc::clone就是克隆一个Rc指针，指向的同一个底层数据

```rust
use std::rc::Rc;
fn main() {
        let a = Rc::new(String::from("test ref counting"));
        println!("count after creating a = {}", Rc::strong_count(&a));
        let b =  Rc::clone(&a);
        println!("count after creating b = {}", Rc::strong_count(&a));
        {
            let c =  Rc::clone(&a);
            println!("count after creating c = {}", Rc::strong_count(&c));
        }
        println!("count after c goes out of scope = {}", Rc::strong_count(&a));
}
```

- `Rc<T>` 是一个智能指针，实现了 `Deref` 特征，因此你无需先解开 `Rc` 指针，再使用里面的 `T`，而是可以直接使用 `T`，例如上例中的 `gadget1.owner.name`

#### **Arc\<T>**

atomic reference counting 允许一个数据资源在同一时刻拥有多个所有者，适用于多线程，同样Arc也是指向底层数据的不可变的引用

在多线程编程中，`Arc` 跟 `Mutex` 锁的组合使用非常常见，它们既可以让我们在不同的线程中共享数据，又允许在各个线程中对其进行修改。

```rust
use std::sync::{Arc, Mutex};

fn main() {
    let my_number = Arc::new(Mutex::new(0));
    let mut handle_vec = vec![]; // JoinHandles will go in here

    for _ in 0..2 { // do this twice
      // Make the clone before starting the thread
        let my_number_clone = Arc::clone(&my_number); 
        let handle = std::thread::spawn(move || { // Put the clone in
            for _ in 0..10 {
                *my_number_clone.lock().unwrap() += 1;
            }
        });
      // save the handle so we can call join on it outside of the loop
       // If we don't push it in the vec, it will just die here
        handle_vec.push(handle); 
                                
    }

    handle_vec.into_iter().for_each(|handle| handle.join().unwrap()); // call join on all handles
    println!("{:?}", my_number);
}
```

This looks complicated but `Arc<Mutex<SomeType>>>` is used very often in Rust, so it becomes natural.

#### **Weak\<T>**

`Weak`相当于一种不持有所有权的`Rc`，它仅仅保存一份指向数据的弱引用，用来解决循环引用(reference cycle)的问题。比如，如果a中包含一个Rc指向b，b中也包含一个Rc指向a，这样引用计数就不可能变成0，最终就会导致内存泄漏的发生。如果使用Weak Rc，那么当一个内存只有弱引用指向时，改内存会被释放。

- `Rc<T>` 调用 `downgrade` 方法转换成 `Weak<T>`
- `Weak<T>` 可使用 `upgrade` 方法转换成 `Option<Rc<T>>`，如果资源已经被释放，则 `Option` 的值是 `None`

**对于父子引用关系，可以让父节点通过 `Rc` 来引用子节点，然后让子节点通过 `Weak` 来引用父节点**。

```rust
use std::rc::Rc;
fn main(){
  let five = Rc::new(5);
  
  // 通过Rc，创建一个Weak指针
  let weak_five = Rc::downgrade(&five);
  // 通过Weak引用访问数据，需要通过upgrade实现,返回一个类型为 Option<Rc<T>>
  let strong_five: Option<Rc<_>> = weak_five.upgrade();
  assert_eq!(*strong_five.unwrap(), 5);
  
  // 手动释放资源`five`
   drop(five);
  // Weak引用的资源已不存在，因此返回None
    let strong_five: Option<Rc<_>> = weak_five.upgrade();
    assert_eq!(strong_five, None);
}
```

Also, you use `Rc::weak_count(&item)` to see the weak count.

#### 对比Weak, Rc, Arc

| `Weak`                                          | `Rc`                                      | `Arc`                                     |
| ----------------------------------------------- | ----------------------------------------- | ----------------------------------------- |
| 不计数                                          | 引用计数                                  | 引用计数                                  |
| 不拥有所有权                                    | 拥有值的所有权                            | 拥有值的所有权                            |
| 不阻止值被释放(drop)                            | 所有权计数归零，才能 drop                 | 所有权计数归零，才能 drop                 |
| 引用的值存在返回 `Some`，不存在返回 `None`      | 引用的值必定存在                          | 引用的值必定存在                          |
| 通过 `upgrade` 取到 `Option<Rc<T>>`，然后再取值 | 通过 `Deref` 自动解引用，取值无需任何操作 | 通过 `Deref` 自动解引用，取值无需任何操作 |
| /                                               | 单线程                                    | 多线程                                    |

### 10. 智能指针--Cell, RefCell, Ref, RefMutW

Cell\<T>

RefCell\<T>

Rust 提供了 `Cell` 和 `RefCell` 用于内部可变性

内部可变性的实现是因为 Rust 使用了 `unsafe` 来做到这一点，但是对于使用者来说，这些都是透明的，因为这些不安全代码都被封装到了安全的 API 中

Ref\<T>

Wraps a borrowed reference to a value in a `RefCell` box. A wrapper type for an immutably borrowed value from a `RefCell<T>`.

RefMut\<T>

Mutex\<T>

### 11. clone vs. copy

### 10. dyn

`dyn` is a prefix of a [trait object](https://doc.rust-lang.org/book/ch17-02-trait-objects.html)’s type. The `dyn` keyword is used to highlight that calls to methods on the associated `Trait` are [dynamically dispatched](https://en.wikipedia.org/wiki/Dynamic_dispatch). To use the trait this way, it must be ‘object safe’. 相当于动态绑定

### 12. Box as_ref(), into_raw()





## Reference

1. [Easy Rust](https://dhghomon.github.io/easy_rust/Chapter_49.html)
2. [Rust圣经](https://course.rs/advance/smart-pointer/rc-arc.html)
