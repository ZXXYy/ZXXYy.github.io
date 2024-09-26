---
layout: post
title:      "Rust裸指针,引用,slice,box,vec,string的内存布局及转换"
date:       2023-07-13 09:00:00
author:     zxy
math: true
categories: ["Coding", "Rust"]
tags: ["Rust"]
post: true
---

> 本篇博客适用于有基础编程经验的小白（有一门熟悉的编程语言，了解泛型并且熟悉堆栈概念）

在零基础入门C/C++的时候，指针就是个晦涩难懂的概念，还有数组指针、指针数组什么的，再加上C++中的引用标准库里的vector，简直就是大一C语言考试的大难题。现在到学些Rust了，这些概念还是绕不开，甚至更加晦涩。box, slice更多的概念被引入了，而且有些概念还和C中不同，令人火大。其实，静下心来学上几个小时，弄明白这些类型（不包括智能指针）的内存布局，以及异同，很多问题就能迎刃而解。这篇博客就是分析这些和指针强相关的类型内存的。首先，我们会通过一个表格来介绍各种类型。然后会结合实际代码来说明各种类型的内存分布，最后我们会介绍各种类型间相互转换。Hope you stick with me!

## 指针相关类型介绍

|              | 裸指针                            | 引用                                            | 切片                 | 字符串切片                | Box                  | Vector               | String                  | 数组        |
| ------------ | --------------------------------- | ----------------------------------------------- | -------------------- | ------------------------- | -------------------- | -------------------- | ----------------------- | ----------- |
| in English   | raw pointer                       | reference                                       | slice                | string slice              | box                  | vec                  | String                  | Array       |
| in Rust      | \*const T <br />\*mut T           | &T<br />&mut T                                  | &[T]<br />&mut [T]   | str                       | Box\<T, A = Global\> | Vec\<T, A = Global\> | String                  | [T; N]      |
| 类型         | primitive                         | primitive                                       | primitive            | primitive                 | std                  | std                  | std                     | primitive   |
| 说明         | 内存不安全的裸指针，就是C中的指针 | 指向有效且对齐的内存的指针，表示对该T元素的借用 | 不定长的连续元素序列 | 不定长的连续UTF-8字符序列 | 指向堆上数据的指针   | 连续可增长的数组     | UTF-8编码的可增长字符串 | 定长数组    |
| 栈上类型大小 | usize                             | usize                                           | 2 * usize            | 2 * usize                 | usize<br />2 * usize | 3 * usize            | 3 * usize               | Sizeof(T)*N |

> T在表格中表示element type，右滑查看更多

使用以下代码可以查看这些类型在栈上的大小

```rust
println!("The size of raw pointer: {}", std::mem::size_of::<*const u64>()); // 8  bytes
println!("The size of reference: {}", std::mem::size_of::<&u64>()); // 8  bytes
println!("The size of slice: {}", std::mem::size_of::<&[u8]>()); // 16 bytes
println!("{}", std::mem::size_of::<[u8]>()); // compiler error
println!("The size of string slice: {}", std::mem::size_of::<&str>()); // 16 bytes
println!("The size of box: {}", std::mem::size_of::<Box<u8>>()); // 8  bytes
println!("The size of box: {}", std::mem::size_of::<Box<[u8]>>()); // 16 bytes
println!("The size of vec: {}", std::mem::size_of::<Vec<u8>>()); // 24 bytes
println!("The size of String: {}", std::mem::size_of::<String>()); // 24 bytes
println!("The size of Array[u8;3]: {}", std::mem::size_of::<[u8;3]>()); // 3  bytes
```

## 指针相关类型内存分布

### 裸指针内存分布

```rust
let val: u64 = 5;
let p: *const u64 = &val;
let val2: u64 = 7;
println!("The value of val: {}", val);       // 5
println!("The value of pointer: {:p}", p);   // 0x16dd6e848
println!("The value of val2: {}", val2);     // 7
println!("The address of val: {:p}", &val);  // 0x16dd6e848
println!("The address of pointer : {:p}", &p); // 0x16dd6e850
println!("The address of val2 : {:p}", &val2); // 0x16dd6e858
println!("The value of pointed memory: {:?}", unsafe{*p}); // 5
```

上面的代码对应的内存如下，其实和C中的内存分布是一模一样的。从这张图中，我们也可以看到一个裸指针占8个字节（在64位机中，usize大小为8 bytes）。

如果把val的类型改成u32，也就是4个字节大小，在64位机中可能会发生一些有意思的事情，和内存对齐相关，可以动手去试试，弄明白为什么。

![](/assets/img/in-post/2023-07-27-pointer/raw_pointer.png)

### 引用内存分布

```rust
let val: u64 = 5;
let r: &u64 = &val;
let val2: u64 = 7;
println!("The value of val: {}", val);       // 5
println!("The value of reference: {:p}", r); // 0x16f6d6810
println!("The value of val2: {}", val2);     // 7
println!("The address of val: {:p}", &val);  // 0x16f6d6810
println!("The address of reference : {:p}", &r); // 0x16f6d6818
println!("The address of val2 : {:p}", &val2);   // 0x16f6d6820
println!("The value of referenced memory: {}", *r); // 5
println!("The value of referenced memory: {}", r);  // 5
```

上面的代码对应的内存如下，可以看出引用本质上也就是一个指针，大小为8个字节。但是引用是安全的，比如在上述代码中解引用`*r`不需要在unsafe块内完成，也可以直接通过引用变量`r`来访问引用元素的值（也就是C++中的引用/别名alias）。在大多数情况下，引用可以直接当做原来的元素去使用。在Rust中引用又是借用机制(borrow)的组成，即引用变量并不会转移内存所有权。

![](/assets/img/in-post/2023-07-27-pointer/reference.png)

换句话说，引用（`&T`）其实就是**附加条件的指针**:

- 这个指针一定是对齐的，非空的(not null)，指向包含T类型有效值的内存地址。
- 这个指针有生命周期(lifetime)，引用的生命周期不可以超过被引用的元素。

- 指针需要解引用去访问元素`*p`（在Rust中解引用裸指针是不安全的，也就是说除非使用unsafe关键字，我们在Rust中不能解引用一个裸指针）。然而，在大多数情况下，引用可以直接当做原来的元素去使用。这是因为`&T`实现了`Deref` trait，编译器在大多数情况下会帮我们自动进行解引用操作([Deref coercion](https://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/first-edition/deref-coercions.html))。

### 切片/数组/Vec的内存分布

由于切片、数组和Vec的关系十分紧密，我们把这三个类型的内存分布放在一起说明。

这段代码有点长，说明了三个类型，一鼓作气，看完就会了！

```rust
let arr: [u8; 4] = [1, 2, 3, 4];
let slice: &[u8] = &arr[0..2];
let mut vec: Vec<u8> = vec![5, 6, 7, 8];
vec.push(9);
let slice2: &[u8] = &vec[0..2];
println!("The value of arr: {:?}", arr);
// Output: The value of arr: [1, 2, 3, 4]
println!("The value of slice: {:?}", slice);
// Output: The value of slice: [1, 2]
println!("The value of vec: {:?}", vec);
// Output: The value of vec: [5, 6, 7, 8, 9]
println!("The value of slice2: {:?}", slice2);
// Output: The value of slice2: [5, 6]
println!("The address of arr: {:p}", &arr);
// Output: The address of arr: 0x16d13a544
println!("The address of slice: {:p}", &slice);
// Output: The address of slice: 0x16d13a548
println!("The address of vec: {:p}", &vec);
// Output: The address of vec: 0x16d13a568
println!("The address of slice2: {:p}", &slice2);
// Output: The address of slice2: 0x16d13a580
```

> 切片这个类型其实是`[T]`，上述代码如`arr[0..2]`的类型就是`[u8]`，但是切片表示的是**不定长**的连续元素序列，其大小在编译的时候是不确定的，所以切片通常以`&[T]`的形式出现。

下面这段代码输出了slice和vec在栈上内存里的值。

```rust
println!("The size of slice: {}", std::mem::size_of_val(&slice));
// Output: The size of slice: 16
let p = &slice as *const _ as *const u64;
println!("The value in memory {:p}: 0x{:x}", p, unsafe{*p});
// Output: The value in memory 0x16d13a548: 0x16d13a544
println!("The value in memory {:p}: {}", unsafe{p.offset(1)}, unsafe{*(p.offset(1))});
// Output: The value in memory 0x16d13a550: 2
let p = &vec as *const _ as *const u64;
println!("The value in memory {:p}: {}", p, unsafe{*p});
// Output: The value in memory 0x16d13a568: 8
println!("The value in memory {:p}: 0x{:x}", unsafe{p.offset(1)}, unsafe{*(p.offset(1))});
// Output: The value in memory 0x16d13a570: 0x124e06bc0
println!("The value in memory {:p}: {}", unsafe{p.offset(2)}, unsafe{*(p.offset(2))});
// Output: The value in memory 0x16d13a578: 5
```

从这个内存分布图中我们可以看出：

- 在Rust中数组的所有数据都存在栈上，而C中数组在栈上只存一个指向具体数据的指针。

- 切片中由一个指向具体数据的指针+所指向的内存的长度(length)组成。slice可以指向栈/堆/全局数据区域中的数据。
- vec由一个指向具体数据的指针+所指向的内存的长度(length)+容器的容量大小(capacity)组成。

> 带有额外信息的指针又被称为胖指针(fat point)

![slice](/assets/img/in-post/2023-07-27-pointer/slice.png)

### 字符串切片/string literal/String的内存分布

字符串切片/string literal/String这三者的关系和切片/数组/Vec的关系非常相似，可以一一对应起来理解。

- 字符串切片(&str)，从本质上来说是有类型为`&[u8]`的切片，附加一个额外条件：这个切片中的所有字节都需要是有效的UTF-8字符。
- String literal，如`"Hello"`，是存在全局静态区域的一个大小固定的UTF-8字符串数组。
- String可以理解为一个具体数据是由UTF-8字符组成的vec

### Box的内存分布

从已经分析的类型中我们可以看到，String和vec会把数据保存在堆上，数组的所有数据都放在栈上。在Rust中，一般而言，我们申明一个变量时，数据都会被存在栈上。但是如果我们想把数据保存在堆上，就需要用到Box。换句话说，box就是一个指向堆空间的指针，可以是一半的指针，也可以是胖指针。

```rust
let val: u64 = 5;
let boxed: Box<u64> = Box::new(val);											
let val2: u64 = 7;
println!("The value of val: {}", val);	 // 5
println!("The value of box: {:p}", boxed); // 0x12d606be0

println!("The address of val: {:p}", &val);     // 0x16ee8a6c0
println!("The address of boxed: {:p}", &boxed); // 0x16ee8a6c8
println!("The address of val2 : {:p}", &val2);  // 0x16ee8a6d0 

println!("The value of box pointed memory: {:?}", boxed);  // 5
println!("The value of box pointed memory: {:?}", *boxed); // 5
```

![](/assets/img/in-post/2023-07-27-pointer/box.png)

```rust
let arr = [0;5];
let boxed: Box<[u8]> = Box::new(arr);
println!("The size of box: {}", std::mem::size_of_val(&boxed));  // 16
```

```rust
let vec = vec![0;5];
let boxed = Box::new(vec);	// ownership of vec is moved to boxed
println!("The size of box: {}", std::mem::size_of_val(&boxed));  // 8
```

![](/assets/img/in-post/2023-07-27-pointer/box2.png)

## 指针相关类型的相互转换

![](/assets/img/in-post/2023-07-27-pointer/cast.png)

其实，有了上面的内存layout理解后，这些API就变得很容易理解了。上面的图当做一个总结，需要注意的是把裸指针转化为其他类型的操作都是`unsafe`的。

接下来我们来辨析两组组概念来加深理解：

1. `Box<T>`, `&Box<T>`, `Box<&T>`以及`&*Box<T>`的区别和联系是什么？他们之间是怎么转换的？

   ```rust
   fn print_type_of<T>(_: T) {
       println!("{}", std::any::type_name::<T>())
   }
   let boxed = Box::new(5);
   print_type_of(&boxed);    // &alloc::boxed::Box<i32>
   print_type_of(&*boxed);   // &i32
   print_type_of(boxed.as_ref());    // &i32
   print_type_of(boxed);    // alloc::boxed::Box<i32>
   let boxed = Box::new(&5);				
   print_type_of(boxed);    // alloc::boxed::Box<&i32>
   ```

   从上面代码的输出我们可以看到，`&boxed`会创建一个指向box的引用，而`&*boxed`会创建一个指向boxed指向的堆上具体数据的引用，`box.as_ref()`和`&*boxed`的效果是一样的。`Box<&T>`则是把本应该在栈上申请的引用放到了堆上。

2. `Box<Vec<T>>`, `Vec<Box<T>>`的区别和联系是什么？

   - `Box<Vec<T>>`: 不要使用，只是套了层壳，把`Vec`的指针、长度、capacity移到了堆上。

   - `Vec<Box<T>>`: 在Vec中每一个元素的大小不固定时使用，比如`T`是trait。

欢迎随时留下评论、反馈、建议或你希望详细了解的信息～

阅读以下Reference可以加深理解～

## Reference

1. [What's the difference between references and pointers in Rust?](https://ntietz.com/blog/rust-references-vs-pointers/)
2. [Array, vec, slice](https://www.cs.brandeis.edu/~cs146a/rust/doc-02-21-2015/book/arrays-vectors-and-slices.html)

2. [Treating Smart Pointers Like Regular References with the `Deref` Trait](https://doc.rust-lang.org/book/ch15-02-deref.html)

3. [Exploring Rust fat pointers](https://iandouglasscott.com/2018/05/28/exploring-rust-fat-pointers/)

4. [Item 9: Familiarize yourself with reference and pointer types](https://www.lurklurk.org/effective-rust/references.html)
5. [Type layout](https://doc.rust-lang.org/reference/type-layout.html)
6. [蚂蚁集团 ｜ Rust 数据内存布局](https://rustmagazine.github.io/rust_magazine_2021/chapter_6/ant-rust-data-layout.html#蚂蚁集团--rust-数据内存布局)
7. [Understanding rust slice](https://codecrash.me/understanding-rust-slices)
8. [Visualizing memory layout of Rust's data types](https://www.youtube.com/watch?v=rDoqT-a6UFg)
9. [When to use Box<Vec<..>> or Vec<Box<..>>?](https://stackoverflow.com/questions/29847928/when-to-use-boxvec-or-vecbox)
