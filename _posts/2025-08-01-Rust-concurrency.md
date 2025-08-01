---
layout: post
title:      "Rust Concurrency "
date:       2025-08-01 09:00:00-0400
author:     zxy
math: true
categories: ["Coding", "Rust"]
tags: ["Rust"]
---
> 这篇blog主要整理了Rust在多线程编程中会用到的相关概念。


## 1. 引言

随着多核处理器的普及，并发编程已成为现代软件开发的重要组成部分。Rust凭借其内存安全和零成本抽象的特性，为并发编程提供了强大而安全的工具。本文将全面介绍Rust中的并发编程概念，包括线程、进程、共享状态、锁机制、原子操作和通信通道(Channel)等。

## 2. 线程、进程与协程

### 2.1 线程、进程与异步任务的对比

| 特性 | 进程 | 线程 | 异步任务 |
| --- | --- | --- | --- |
| 资源隔离 | 完全隔离 | 共享内存 | 共享内存 |
| 创建开销 | 最大 | 中等 | 最小 |
| 切换开销 | 最大 | 中等 | 最小 |
| 并行能力 | 真并行 | 真并行 | 并发(非并行) |
| 适用场景 | 需要隔离、安全性 | CPU密集型 | I/O密集型 |
| 通信方式 | IPC | 共享内存、锁 | 共享内存、通道 |

**关键差异：**

- **资源消耗**：thread::spawn创建的线程消耗更多内存和CPU资源

- **并发数量**：tokio::spawn可以轻松创建成千上万个任务，而OS线程数量有限

- **调度**：tokio任务由Rust运行时调度，thread由操作系统调度

- **阻塞行为**：在tokio::spawn中使用阻塞操作会影响整个运行时性能

### 2.2 如何选择？

- **异步任务tokio::spawn**：I/O密集型，需要高并发，共享状态，任务主要是等待外部资源

  - 创建轻量级的异步任务（green thread/协程），任务在tokio运行时的线程池上执行（默认线程数等于CPU核心数）

  - 任务切换在用户空间进行，开销很小

  - 只能运行async函数，**不应包含阻塞操作**

- **线程thread::spawn**：CPU密集型，需要并行计算，可接受共享内存的复杂性

  - 创建真正的操作系统线程，每个线程都有独立的栈空间（通常2MB）

  - 线程切换由操作系统内核调度，开销较大

  - 可以运行任何代码，包括阻塞操作

- **进程**：需要隔离、调用外部程序、或者要求极高的稳定性

### 2.3 线程、异步任务和进程的代码示例

下面展示如何创建线程、异步任务和进程的代码示例：

#### **使用thread::spawn创建线程**

```rust
use std::thread;
use std::time::Duration;

fn main() {
	// 创建多个线程
    let mut handles = vec![];
	for id in 0..5 {
    	let handle = thread::spawn(|| {
        	println!("线程开始执行");
        	// 模拟耗时操作
        	thread::sleep(Duration::from_millis(500));
        	println!("线程执行完毕");
        	// 返回值会被join()获取
        	42+i
    	});
		handles.push(handle);
	}
    
    // 主线程继续执行
    println!("主线程继续执行其他工作");
    // 等待线程完成并获取返回值
	for handle in handles {
        let result = handle.join().unwrap();
    	println!("线程返回值: {}", result);
    }
}
```

#### 使用异步任务tokio::spawn

```rust
use tokio::time::{sleep, Duration};
#[tokio::main]
async fn main() {
    // 创建一个异步任务
    let handle = tokio::spawn(async {
        println!("异步任务开始执行");
        // 模拟异步操作
        sleep(Duration::from_secs(1)).await;
        println!("异步任务完成");
		// 返回值会被await获取
        "完成"
    });
    // 等待异步任务完成并获取返回值
    let result = handle.await.unwrap();
    println!("异步任务返回值: {}", result);
}
```

#### **创建进程**

```rust
use std::process::Command;
use std::io;

fn main() -> io::Result<()> {
    // 简单的进程创建并等待完成
    let status = Command::new("ls")
        .arg("-l")
        .status()?;
    println!("进程退出状态: {}", status);
    
    // 捕获进程输出
    let output = Command::new("echo")
        .arg("Hello from another process!")
        .output()?;
        
    if output.status.success() {
        let stdout = String::from_utf8_lossy(&output.stdout);
        println!("进程输出: {}", stdout);
    }
    
    // 创建进程但不等待完成
    let mut child = Command::new("sleep")
        .arg("2")
        .spawn()?;
        
    println!("子进程已启动，PID: {}", child.id());
    
    // 可以选择等待完成
    let status = child.wait()?;
    println!("子进程已完成，退出状态: {}", status);
    
    // 使用管道与进程通信
    let mut child = Command::new("cat")
        .stdin(std::process::Stdio::piped())
        .stdout(std::process::Stdio::piped())
        .spawn()?;
        
    // 向进程写入数据
    if let Some(mut stdin) = child.stdin.take() {
        use std::io::Write;
        stdin.write_all(b"Hello from Rust!")?;
        // 关闭stdin，否则cat会一直等待输入
        drop(stdin);
    }
    
    // 读取进程输出
    let output = child.wait_with_output()?;
    println!("cat输出: {}", String::from_utf8_lossy(&output.stdout));
    
    Ok(())
}
```

## 3. 共享状态和所有权

### 3.1 Arc - 原子引用计数

在Rust中，锁（如`Mutex<T>`、`RwLock<T>`）外面套`Arc`是为了**在多个线程之间共享同一个锁**。Rust的所有权系统规定每个值只能有一个所有者，但多线程编程需要多个线程访问同一份数据，`Arc`（Atomically Reference Counted）提供了**共享所有权**。

`Arc`的工作原理：

1. **引用计数**：跟踪有多少个"所有者"

2. **原子操作**：引用计数的增减是线程安全的

3. **自动清理**：当引用计数降到0时自动释放内存

4. **共享访问**：多个线程可以同时持有指向同一数据的Arc

```rust
use std::sync::Arc;
use std::thread;
fn main() {
    let data = Arc::new(vec![1, 2, 3]);
    let mut handles = vec![];
    
    for i in 0..3 {
        let data_clone = Arc::clone(&data);
        let handle = thread::spawn(move || {
            println!("Thread {}: {:?}", i, data_clone);
        });
        handles.push(handle);
    }
    for handle in handles {
        handle.join().unwrap();
    }
}
```

## 4. 锁机制

锁是并发编程中保护共享数据的重要工具。Rust提供了多种锁机制，每种都有其特定的用途和性能特点。

### 4.1 锁类型对比

| 锁类型 | 并发访问模式 | 阻塞行为 | 适用场景 | 性能特点 | API复杂度 |
| --- | --- | --- | --- | --- | --- |
| `Mutex<T>` | 一次只允许一个线程访问 | 阻塞等待 | 写操作频繁 | 中等 | 简单 |
| `RwLock<T>` | 多读单写 | 阻塞等待 | 读多写少 | 读多时高 | 中等 |
| `tokio::sync::Mutex<T>` | 一次只允许一个任务访问 | 异步等待 | 异步环境中的互斥访问 | 高（不阻塞线程） | 简单 |
| `tokio::sync::RwLock<T>` | 多读单写 | 异步等待 | 异步环境中读多写少 | 高（不阻塞线程） | 中等 |
| `parking_lot::Mutex<T>` | 一次只允许一个线程访问 | 阻塞等待 | 高性能要求的互斥访问 | 高（比标准库快） | 简单（无需处理Result） |
| `parking_lot::RwLock<T>` | 多读单写 | 阻塞等待 | 高性能要求的读多写少 | 高（比标准库快） | 简单（无需处理Result） |
| `Semaphore` | 限制并发数量 | 阻塞/异步等待 | 资源池、限流 | 中等 | 简单 |
| `Barrier` | 同步多个线程 | 阻塞等待 | 多线程同步点 | 低（设计如此） | 简单 |
| `Once` | 确保代码只执行一次 | 阻塞等待 | 一次性初始化 | 高 | 非常简单 |

### 4.2 锁的选择指南

#### 基于访问模式选择

1. **写操作频繁场景**，选择: `Mutex` 或 `parking_lot::Mutex`避免读写锁的复杂管理开销，互斥锁在频繁写入时性能更好

2. **读操作远多于写操作场景**，选择: `RwLock` 或 `parking_lot::RwLock` 允许多个读取者并发访问，提高并发度

3. **异步环境中的锁**，选择: `tokio::sync::Mutex` 或 `tokio::sync::RwLock`， 不会阻塞线程，允许异步等待锁释放

4. **高性能要求场景，** 选择: `parking_lot` 库中的锁，更小的内存占用，更快的锁操作，更简洁的API

#### 基于特殊需求选择

1. **需要限制并发数量**，选择: `Semaphore`

   - 用途: 限制同时访问资源的线程数，实现资源池或限流

2. **需要线程同步点**，选择: `Barrier`，让多个线程在某个点同步等待，然后一起继续执行


3) **一次性初始化需求**，选择: `Once` ，确保某段代码只执行一次，常用于静态变量初始化


4. **避免死锁需求**，选择: 尝试加锁API，如`try_lock()`或带超时的锁，避免无限等待，实现更健壮的错误处理

### 4.3 代码示例

#### Mutex示例 - 适合写操作频繁场景

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // 创建一个共享计数器
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    // 创建10个线程，每个线程递增计数器10次
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..10 {
                // 获取锁
                let mut count = counter.lock().unwrap();
                // 修改共享数据
                *count += 1;
                // 锁在作用域结束时自动释放
            }
        });
        handles.push(handle);
    }
    
    // 等待所有线程完成
    for handle in handles {
        handle.join().unwrap();
    }
    
    // 输出最终结果
    println!("计数器最终值: {}", *counter.lock().unwrap());
}
```

#### RwLock示例 - 适合读多写少场景

```rust
use std::sync::{Arc, RwLock};
use std::thread;
use std::time::Duration;

struct Database {
    data: Vec,
}

fn main() {
    // 创建共享数据库
    let db = Arc::new(RwLock::new(Database {
        data: vec!["初始数据".to_string()],
    }));
    
    // 创建一个写入线程
    let db_writer = Arc::clone(&db);
    let writer = thread::spawn(move || {
        for i in 0..5 {
            // 获取写锁
            let mut database = db_writer.write().unwrap();
            // 修改数据
            database.data.push(format!("新数据 {}", i));
            println!("写入: 新数据 {}", i);
            // 写锁在作用域结束时释放
            drop(database); // 显式释放锁，让读取者有机会获取锁
            thread::sleep(Duration::from_millis(100));
        }
    });
    
    // 创建多个读取线程
    let mut readers = vec![];
    for id in 0..3 {
        let db_reader = Arc::clone(&db);
        let reader = thread::spawn(move || {
            for _ in 0..8 {
                // 获取读锁 - 多个读取者可以同时持有读锁
                let database = db_reader.read().unwrap();
                println!("读取者 {}: 数据量 = {}", id, database.data.len());
                // 读锁在作用域结束时释放
                thread::sleep(Duration::from_millis(50));
            }
        });
        readers.push(reader);
    }
    
    // 等待所有线程完成
    writer.join().unwrap();
    for reader in readers {
        reader.join().unwrap();
    }
}
```

#### 异步Mutex示例 - 适合异步环境

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    // 创建异步互斥锁
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    // 创建多个异步任务
    for id in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = tokio::spawn(async move {
            // 异步获取锁
            let mut count = counter.lock().await;
            *count += 1;
            println!("任务 {} 将计数器增加到 {}", id, *count);
            // 锁在作用域结束时释放
        });
        handles.push(handle);
    }
    
    // 等待所有任务完成
    for handle in handles {
        handle.await.unwrap();
    }
    
    // 输出最终结果
    println!("计数器最终值: {}", *counter.lock().await);
}
```

#### Semaphore示例 - 限制并发数量

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // 创建信号量，最多允许3个并发任务
    let semaphore = Arc::new(Semaphore::new(3));
    let mut handles = vec![];
    
    // 创建8个任务，但最多只有3个能同时执行
    for id in 0..8 {
        let semaphore = Arc::clone(&semaphore);
        let handle = tokio::spawn(async move {
            // 获取许可
            let permit = semaphore.acquire().await.unwrap();
            println!("任务 {} 获得许可，开始执行", id);
            
            // 模拟工作
            sleep(Duration::from_millis(500)).await;
            
            println!("任务 {} 完成执行", id);
            // 许可在permit被drop时自动释放
            drop(permit);
        });
        handles.push(handle);
    }
    
    // 等待所有任务完成
    for handle in handles {
        handle.await.unwrap();
    }
}
```

#### Barrier示例 - 线程同步点

```rust
use std::sync::{Arc, Barrier};
use std::thread;
use std::time::Duration;

fn main() {
    // 创建屏障，等待3个线程
    let barrier = Arc::new(Barrier::new(3));
    let mut handles = vec![];
    
    for id in 0..3 {
        let barrier = Arc::clone(&barrier);
        let handle = thread::spawn(move || {
            println!("线程 {} 开始执行第一阶段", id);
            thread::sleep(Duration::from_millis(100 * id)); // 模拟不同的工作时间
            println!("线程 {} 完成第一阶段", id);
            
            // 等待所有线程完成第一阶段
            let wait_result = barrier.wait();
            
            // 只有一个线程的wait()会返回BarrierWaitResult::Leader
            if wait_result.is_leader() {
                println!("所有线程完成第一阶段，继续执行");
            }
            
            println!("线程 {} 开始执行第二阶段", id);
            // 继续执行第二阶段...
        });
        handles.push(handle);
    }
    
    // 等待所有线程完成
    for handle in handles {
        handle.join().unwrap();
    }
}
```

#### Once示例 - 一次性初始化

```rust
use std::sync::Once;
use std::thread;

// 全局静态变量
static INIT: Once = Once::new();
static mut CONFIG: Option = None;

fn get_config() -> &'static str {
    // 确保初始化代码只执行一次
    INIT.call_once(|| {
        // 这段代码只会执行一次
        println!("正在初始化配置...");
        // 模拟耗时的初始化过程
        thread::sleep(std::time::Duration::from_millis(100));
        
        // 安全地初始化静态变量
        unsafe {
            CONFIG = Some(String::from("数据库配置信息"));
        }
        println!("配置初始化完成");
    });
    
    // 安全地访问已初始化的静态变量
    unsafe {
        CONFIG.as_ref().unwrap()
    }
}

fn main() {
    // 创建多个线程，都尝试初始化配置
    let mut handles = vec![];
    
    for id in 0..5 {
        let handle = thread::spawn(move || {
            println!("线程 {} 请求配置", id);
            let config = get_config();
            println!("线程 {} 获得配置: {}", id, config);
        });
        handles.push(handle);
    }
    
    // 等待所有线程完成
    for handle in handles {
        handle.join().unwrap();
    }
}
```

#### parking_lot库示例 - 高性能锁

```rust
use parking_lot::{Mutex, RwLock};
use std::sync::Arc;
use std::thread;
use std::time::Duration;

fn main() {
    // parking_lot的Mutex不返回Result，API更简洁
    let counter = Arc::new(Mutex::new(0));
    
    // 创建线程递增计数器
    let counter_clone = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut count = counter_clone.lock(); // 不需要unwrap()
        *count += 1;
    });
    
    // parking_lot的RwLock也更简洁
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));
    
    // 读取线程
    let data_clone = Arc::clone(&data);
    let read_handle = thread::spawn(move || {
        let values = data_clone.read(); // 不需要unwrap()
        println!("读取数据: {:?}", *values);
    });
    
    // 写入线程
    let data_clone = Arc::clone(&data);
    let write_handle = thread::spawn(move || {
        let mut values = data_clone.write(); // 不需要unwrap()
        values.push(4);
    });
    
    // 等待所有线程完成
    handle.join().unwrap();
    read_handle.join().unwrap();
    write_handle.join().unwrap();
    
    // 输出最终结果
    println!("计数器最终值: {}", *counter.lock());
    println!("数据最终值: {:?}", *data.read());
    
    // parking_lot还支持尝试加锁，不会阻塞
    if let Some(mut count) = counter.try_lock() {
        *count += 1;
        println!("成功获取锁并递增: {}", *count);
    } else {
        println!("锁被占用，无法获取");
    }
    
    // parking_lot支持带超时的锁
    let timeout = Duration::from_millis(100);
    if let Some(mut count) = counter.try_lock_for(timeout) {
        *count += 1;
        println!("在超时前成功获取锁: {}", *count);
    } else {
        println!("等待超时，无法获取锁");
    }
}
```

### 4.4 锁的最佳实践

1. **关于选择库**

   - 尽量避免对标准库 std::sync 模块中锁同步原语的使用，建议使用 parking_lot 的实现。

   - 尽量避免直接使用标准库 std::sync::mpsc 模块中的 channel，替换为 crossbeam

2. **尽量减小锁的范围，但要避免过细粒度的锁，锁的获取和释放有开销**

   - 只锁定真正需要保护的代码，尽早释放锁，可以使用代码块或显式调用`drop()`

   - 可以在函数级别加锁，在函数开头加锁，加锁的函数在命名的时候需要用动词说明对数据有增删改查的操作（Rust）；也可以在数据结构的维度加锁，用的时候加锁，用完就释放。

3. **避免死锁**：保持一致的锁获取顺序，使用`try_lock()`或带超时的锁，避免在持有锁时调用可能获取其他锁的代码


4) **处理锁中的panic**：Rust的锁在panic时会被毒化(poisoned)，确保锁在panic时释放。考虑使用`parking_lot`库，它的锁不会被毒化


5. **考虑无锁替代方案**

   - 对于简单数据，考虑使用原子类型

   - 使用消息传递代替共享状态

   - 设计无共享的数据结构

## 5. 原子式加锁

原子操作利用CPU的原子指令实现，是不可分割的操作，要么完全执行，要么完全不执行，没有中间状态。线程不会因为等待锁而被阻塞，在并发编程中，原子操作可以在不使用锁的情况下实现线程安全的数据访问，具有更低的开销。

### 5.1 原子类型

Rust中的原子操作非常丰富，主要通过`std::sync::atomic`模块提供：

```rust
use std::sync::atomic::*;

// 整数类型
AtomicBool, AtomicI8, AtomicI16, AtomicI32, AtomicI64, AtomicIsize
AtomicU8, AtomicU16, AtomicU32, AtomicU64, AtomicUsize

// 指针类型
AtomicPtr
```

与锁相比，原子操作的优势在于：

```rust
// Mutex - 有锁
let counter = Arc::new(Mutex::new(0u64));
let mut guard = counter.lock().unwrap(); // 获取锁，可能阻塞
*guard += 1; // 修改数据
// guard 被 drop 时释放锁

// AtomicU64 - 无锁
let counter = Arc::new(AtomicU64::new(0));
counter.fetch_add(1, Ordering::Relaxed); // 直接原子操作，不会阻塞等锁
```

### 5.2 原子操作中的内存顺序

内存顺序（Memory Ordering）控制原子操作在多线程环境中如何与其他操作交互。这是因为现代CPU和编译器会对指令进行重排序以提高性能。

编译器和CPU为了性能优化，会重新排列指令的执行顺序，只要不影响**单线程**的程序逻辑：

```rust
// 原始代码
let mut x = 0;
let mut y = 0;
x = 1;           // 操作A
y = 2;           // 操作B
println!("{}", x);

// 编译器/CPU 可能重排为：
let mut x = 0;
let mut y = 0;
y = 2;           // 操作B 先执行
x = 1;           // 操作A 后执行  
println!("{}", x);
// 单线程下结果一样，但多线程下可能出问题
```

Rust提供了不同的内存顺序级别来控制这种重排：

```rust
use std::sync::atomic::Ordering;

Ordering::Relaxed    // 只保证原子性，没有同步或顺序约束
Ordering::Acquire    // 获取屏障：阻止后续读写重排到此操作之前
Ordering::Release    // 释放屏障：阻止之前读写重排到此操作之后  
Ordering::AcqRel     // 同时具有获取和释放语义
Ordering::SeqCst     // 顺序一致性：全局统一的操作顺序
```

#### Acquire-Release模式

这是最常用的同步模式，类似于锁的获取和释放：

**Release（释放）**：像一个"单向屏障"，阻止之前的读写操作重排到这个store操作之后。

```rust
// 生产者线程
fn producer() {
    DATA.store(42, Ordering::Relaxed);      // 操作1：设置数据
    READY.store(true, Ordering::Release);   // 操作2：Release屏障
}
// Release 确保：
// 1. DATA.store(42, ...) 必须先执行完
// 2. READY.store(true, Release) 之前的操作不能重排到这之后
// 结果：数据总是在标志位之前被设置
```

**Acquire（获取）**：像一个"单向屏障"，阻止后续的读写操作重排到这个load操作之前。

```rust
// 错误示例：使用 Relaxed
fn consumer_wrong() {
    if READY.load(Ordering::Relaxed) {  // 操作1
        let value = DATA.load(Ordering::Relaxed);  // 操作2
        println!("Data: {}", value);
    }
}

// 正确示例：使用 Acquire  
fn consumer_correct() {
    if READY.load(Ordering::Acquire) {  // 操作1：Acquire屏障
        let value = DATA.load(Ordering::Relaxed);  // 操作2：不能重排到操作1之前
        println!("Data: {}", value);
    }
}

// 如果没有Acquire，CPU 可能的重排序：
// 1. let value = DATA.load(...) // 先读数据
// 2. if READY.load(...)         // 后检查就绪标志
// 结果：可能读到未初始化的数据！
```

#### 内存顺序选择指南

```rust
// 决策树
fn choose_ordering() {
    // 1. 问：这个操作需要与其他操作同步吗？
    if needs_synchronization() {
        // 2. 问：是否需要全局一致的顺序？
        if needs_global_order() {
            // 使用 SeqCst
        } else {
            // 3. 问：是发布数据还是消费数据？
            if publishing_data() {
                // 使用 Release
            } else if consuming_data() {
                // 使用 Acquire  
            } else {
                // 既发布又消费，使用 AcqRel
            }
        }
    } else {
        // 只需要原子性，使用 Relaxed
        // 不需要同步：不关心与其他操作的顺序关系
    }
}
```

### 5.3 原子操作代码示例

```rust
use std::sync::atomic::{AtomicU64, Ordering};

let atomic = AtomicU64::new(0);

// 读取
let value = atomic.load(Ordering::Relaxed);

// 存储
atomic.store(42, Ordering::Relaxed);

// 原子加法
let old_value = atomic.fetch_add(10, Ordering::Relaxed);

// 原子最大值更新
atomic.fetch_max(100, Ordering::Relaxed);

// 比较并交换
// atomic.compare_exchange(期望值, 新值, 成功时的内存顺序, 失败时的内存顺序)
// 如果当前值等于期望值，就替换为新值，否则不做任何操作。
// 多个线程可能同时尝试修改同一个值，我们需要确保只有一个线程成功
let result = atomic.compare_exchange(42, 99, Ordering::Relaxed, Ordering::Relaxed);
```

```rust
// 实现一个原子计数器，只在值小于100时递增
fn increment_if_less_than_100(counter: &AtomicU64) -> bool {
    loop {
        let current = counter.load(Ordering::Relaxed);
        if current >= 100 {
            return false; // 已经达到上限
        }
        // 尝试将 current 替换为 current + 1
        match counter.compare_exchange(current, current + 1, Ordering::Relaxed, Ordering::Relaxed) {
            Ok(_) => return true,  // 成功递增
            Err(_) => {
                // 失败说明值被其他线程改了，重试
                continue;
            }
        }
    }
}
```

## 6. Channel (通道)

Channel是线程间通信的强大工具，它允许线程之间安全地传递消息。

### 6.1 Channel类型对比

| 类型 | 特点 | 适用场景 |
| --- | --- | --- |
| std::sync::mpsc | 多生产者，单消费者 | 简单的工作队列 |
| crossbeam_channel | 多生产者，多消费者 | 高性能工作分发 |
| tokio::sync::mpsc | 异步多生产者，单消费者 | 异步任务间通信 |
| tokio::sync::broadcast | 广播，多接收者都收到消息 | 事件通知、配置更新 |
| tokio::sync::oneshot | 一次性，单发送单接收 | 异步任务结果返回 |
| tokio::sync::watch | 状态监听，接收者看到最新状态 | 配置变更监听 |

### 6.2 Channel的选择指南

- **crossbeam_channel**：多对多的**工作队列**（消息被消费）

  - **任务分发**：多个worker处理任务队列

  - **负载均衡**：分散工作负载

  - **生产者-消费者模式**

- **tokio::sync::broadcast**：一对多的**事件广播**（消息被复制）

  - **事件通知**：状态变化通知所有监听者

  - **配置更新**：新配置广播给所有组件

  - **关闭信号**：优雅关闭所有任务

- **标准库中的channel**：

  - `std::sync::mpsc::channel()` - 异步无界channel

  - `std::sync::mpsc::sync_channel(bound)` - 同步有界channel

- **async/await生态中的channel**：

  - `tokio::sync::mpsc` - Tokio的异步channel

  - `tokio::sync::oneshot` - 一次性channel，只能发送一个值

  - `tokio::sync::watch` - 状态监听channel，接收者能看到最新状态变化

- **第三方crate中的高性能channel**：

  - `crossbeam-channel` - 功能更丰富，性能更好的channel实现

  - `flume` - 另一个高性能的channel库

  - `async-channel` - 支持async/await的channel

### 6.3 Crossbeam Channel

Crossbeam提供了功能丰富的通道实现，支持多生产者多消费者模式，crossbeam的基本操作：

```rust
use crossbeam_channel as channel;
use std::thread;
use std::time::Duration;

fn crossbeam_example() {
    // 1. 无界 channel
    let (tx, rx) = channel::unbounded::();
    
    // 2. 有界 channel  
    let (tx_bounded, rx_bounded) = channel::bounded::(10);
    
    // 3. 零容量 channel（同步交换）
    let (tx_sync, rx_sync) = channel::bounded::(0);
    
    // 发送端
    thread::spawn(move || {
        tx.send(42).unwrap();
        
        // 非阻塞发送
        match tx_bounded.try_send(100) {
            Ok(_) => println!("Sent successfully"),
            Err(channel::TrySendError::Full(_)) => println!("Channel full"),
            Err(channel::TrySendError::Disconnected(_)) => println!("Disconnected"),
        }
        
        // 带超时的发送
        if let Err(_) = tx_bounded.send_timeout(200, Duration::from_millis(100)) {
            println!("Send timeout");
        }
    });
    
    // 接收端 - 多种接收方式
    thread::spawn(move || {
        // 1. 阻塞接收
        let value = rx.recv().unwrap();
        
        // 2. 非阻塞接收
        match rx_bounded.try_recv() {
            Ok(val) => println!("Received: {}", val),
            Err(channel::TryRecvError::Empty) => println!("Channel empty"),
            Err(channel::TryRecvError::Disconnected) => println!("Disconnected"),
        }
        
        // 3. 带超时的接收
        match rx_bounded.recv_timeout(Duration::from_millis(100)) {
            Ok(val) => println!("Received: {}", val),
            Err(channel::RecvTimeoutError::Timeout) => println!("Receive timeout"),
            Err(channel::RecvTimeoutError::Disconnected) => println!("Disconnected"),
        }
        
        // 4. 选择操作（类似 Go 的 select）
        channel::select! {
            recv(rx) -> msg => {
                if let Ok(val) = msg {
                    println!("Got from rx: {}", val);
                }
            },
            recv(rx_bounded) -> msg => {
                if let Ok(val) = msg {
                    println!("Got from rx_bounded: {}", val);
                }
            },
            default => {
                println!("No message available");
            },
        }
    });
}
```

**多生产者多消费者示例**

```rust
use crossbeam_channel as channel;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = channel::bounded::(10);
    
    // 创建多个生产者
    for producer_id in 0..3 {
        let tx_clone = tx.clone();
        
        thread::spawn(move || {
            for msg_num in 0..5 {
                let message = format!("生产者{}-消息{}", producer_id, msg_num);
                tx_clone.send(message).unwrap();
                thread::sleep(Duration::from_millis(100));
            }
            println!("生产者{} 完成发送", producer_id);
        });
    }
    
    // 创建多个消费者 (关键：rx.clone() 在 crossbeam 中是可以的！)
    for consumer_id in 0..2 {
        let rx_clone = rx.clone();
        
        thread::spawn(move || {
            while let Ok(message) = rx_clone.recv() {
                println!("消费者{} 收到: {}", consumer_id, message);
                thread::sleep(Duration::from_millis(200)); // 模拟处理时间
            }
            println!("消费者{} 退出", consumer_id);
        });
    }
    
    // 丢弃原始的 tx 和 rx，让消费者知道没有更多的消息了
    drop(tx);
    drop(rx);
    
    // 等待所有线程完成
    thread::sleep(Duration::from_secs(3));
}
```

### 6.4 标准库Channel

仅支持单个消费者

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel::();
    
    // 用循环创建多个生产者
    for producer_id in 0..5 {
        let tx_clone = tx.clone();
        
        thread::spawn(move || {
            for msg_num in 0..3 {
                let message = format!("生产者{}-消息{}", producer_id, msg_num);
                tx_clone.send(message).unwrap();
                thread::sleep(Duration::from_millis(100 * (producer_id + 1)));
            }
            println!("生产者{} 完成发送", producer_id);
        });
    }
    
    // 丢弃原始 tx，这样当所有生产者完成后 channel 会自动关闭
    drop(tx);
    
    // 单个消费者接收所有消息
    while let Ok(message) = rx.recv() {
        println!("消费者收到: {}", message);
    }
    
    println!("所有消息处理完毕");
}
```

### 6.5 Tokio广播Channel

```rust
// 典型的 broadcast 用法：关闭信号
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (shutdown_tx, _) = broadcast::channel(1);
    
    // 启动多个任务
    for i in 0..3 {
        let mut shutdown_rx = shutdown_tx.subscribe();
        tokio::spawn(async move {
            tokio::select! {
                _ = some_work() => println!("任务{}正常完成", i),
                _ = shutdown_rx.recv() => println!("任务{}收到关闭信号", i),
            }
        });
    }
    
    // 发送关闭信号给所有任务
    shutdown_tx.send(()).unwrap();
}

async fn some_work() {
    tokio::time::sleep(tokio::time::Duration::from_secs(10)).await;
}
```

### 

## 7. 参考资源

- [Rust官方文档 - 并发章节](https://doc.rust-lang.org/book/ch16-00-concurrency.html)

- [Tokio文档](https://tokio.rs/tokio/tutorial)

- [Crossbeam项目](https://github.com/crossbeam-rs/crossbeam)

- [《Rust原子和锁》](https://marabos.nl/atomics/)