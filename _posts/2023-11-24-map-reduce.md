---
layout: post
title:      "MIT6.824 Lab1 Map Reduce"
date:       2023-11-24 09:00:00
author:     zxy
math: true
categories: ["Coding", "Courses", "MIT6.824"]
tags: ["MIT6.824"]
post: true
---

> 阅读本篇blog前需要理解map reduce的基本概念，以及尝试实现Lab1
>
> Lab Description: [https://pdos.csail.mit.edu/6.824/labs/lab-mr.html](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)
>
> Map reduce Lecture: [https://youtu.be/WtZ7pcRSkOA](https://youtu.be/WtZ7pcRSkOA)
>
> Map reduce Lecture Notes: [https://pdos.csail.mit.edu/6.824/notes/l01.txt](https://pdos.csail.mit.edu/6.824/notes/l01.txt)
>
> Mao reduce paper: [https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)

这篇博客中记录我实现map reduce这个实验的过程，以及实现过程中遇到的问题。这可能比仅仅看最后的源码实现，可以带给你更多的帮助。这篇博客对应的源码也已开源[https://github.com/ZXXYy/mit6.824-2023](https://github.com/ZXXYy/mit6.824-2023)

## MapReduce Key point

- **核心思想**：将大规模的数据处理任务分解成可以并行处理的小任务，并通过Map 阶段和 Reduce 阶段来实现数据处理。

- **目标**: easy for non-expert to write distributed applications，即程序员只需定义映射和归约函数，通常是相当简单的顺序代码。分布式的所有方面由MapReduce 框架管理并隐藏。
- **Approach**：
  - map + reduce function(stateless)--一般由用户提供
  - map reduce framework/library deals with distribution--框架实现

- **Abstract view:**

  ​A bunch of files: f1, f2, f3…

  ​keys, values = map(f1), word count = map

  ```
  Input1 -> Map -> a,1 b,1
  Input2 -> Map ->     b,1
  Input3 -> Map -> a,1     c,1
                    |   |   |
                    |   |   -> Reduce -> c,1
                    |   -----> Reduce -> b,2
                    ---------> Reduce -> a,2
  1) input is (already) split into M files
  2) MR calls Map() for each input file, produces list of k,v pairs
    "intermediate" data
    each Map() call is a "task"
  3) when Maps are done,
    MR gathers all intermediate v's for each k,
    and passes each key + values to a Reduce call
  4) final output is set of <k,v> pairs from Reduce()s
  ```

**Expensive costs**: The reduce function needs to contact every mapper to extract the output and fetch intermediate results through network.

- Can map  and reduce function run twice? yes, map and reduce both  are stateless—functional & deterministic

- Coordinate cannot fail, coordinate has states.

- What about slow workers? backup/replicate tasks, go for the first finishing task.

## My Implementation

在通读Lab1的实验说明之后，可以理解需要在以下文件中新增代码：

- `src/mr/coordinator.go`：增加状态维护，任务分配逻辑
- `src/mr/worker.go`： 新增map/reduce任务的执行逻辑，即如何调用用户提供的mapf和reducef函数
- `src/mr/rpc.go`: 新增RPC call参数和返回值的结构体

整个MapReduce的流程如下：

1. worker通过RPC向coordinator请求一个任务
2. coordinator将当前未完成的任务分配给这个worker
3. worker根据coordinator的RPC返回值执行相应的Map/Reduce任务
4. worker完成任务后，通过RPC通知coordinator相应任务已经完成

其中，需要注意的细节有：

1. reduce任务必须在所有map任务完成后才可以开始
2. coordinator需要处理crashed worker的情况，在某个worker很长一段时间没响应后，应该把该worker上的任务分配给其他worker

有了上述理解之后，我们就可以开始设计相应的数据结构了。

### Data Structure

Coordinator中需要维护任务的对应状态，设计如下：

```go
// 通过Status来表示对应处理文件的处理状态
type Status int32

const (
	Ready   Status = 0 // 等待worker处理
	Process Status = 1 // 有worker正在处理
	Done    Status = 2 // 已完成处理
)

type FileStatus struct {
	status    Status
	timestamp int64	// 通过timestamp来处理crashed worker的情况，如果worker处理的时间过长，coordinator再次分配该任务给新worker
}

type Coordinator struct {
  files             map[string]FileStatus // 表示需要处理的原始文件，由map函数分析，如files['pg-grimm.txt']={Ready, 0}
	filesIndex        map[string]int        // 给每一个原始文件一个index，便于索引
	intermediateFiles map[string]FileStatus // 表示map后的中间文件，由reduce函数分析
	intermediateIndex map[string]int        // 给每一个中间文件一个index，便于索引
	nReduce           int
	fileMutex         sync.Mutex            // 整个结构体的锁，防止多个进程对该结构访问造成的数据竞争
}
```

worker需要向coordinator请求任务以及通知coordinator任务已经完成，所以设计了两个RPC call（`AssignTask`和`FinishTask`）。

这两个RPC call对应的数据结构如下：

```go
type TaskArgs struct{}
type TaskReply struct {
	NReduce       int     // 告诉worker总共有几个reduce task
	Nmap          int     // 告诉worker总共有几个map task
	MapTaskNum    int     // 告诉worker当前处理的是第几个map task
	ReduceTaskNum int     // 告诉worker当前处理的是第几个reduce task
	TaskType      string  // 告诉worker需要处理的task类型是map还是reduce
	TaskFile      string  // 告诉worker需要处理的文件是哪个
}
// ⚠️!：MapTaskNum这个参数只会在map task中使用
// ⚠️!：ReduceTaskNum这个参数只会在reduce task中使用
```

```go
type FinishArgs struct {
	TaskType string     // 告诉coordinator当前完成的任务类型
	TaskFile string     // 告诉coordinator当前完成的任务对应的文件
}
type FinishReply struct{}
```

### RPC Call

具体流程如下，源码实现详见GitHub

Coordinator

- AssignTask

  1. 处理crashed worker情况，判断当前是否有正在处理的task过长时间没响应
     - 若有讲该task状态设置为Ready，便于后续分配给新worker

  2. 处理原始文件，如果当前有原始文件未处理，将该map任务分配给worker
  3. 判断所有原始文件已经处理完成，否则告诉worker等待
  4. 若所有原始文件处理完成，处理map生成的中间文件，将给reduce任务分配给worker

- FinishTask

  - 更新对应任务的文件状态

Worker对应的代码如下：

```go
func Worker(mapf func(string, string) []KeyValue, reducef func(string, []string) string) {
  for true{
    // RPC call
    task, taskfile, nreduce, nmap, mapTaskNum, reduceTaskNum := AvailableForTask()
    // 判断task类型
    if task == "map" {
      do_map(taskfile, nreduce, mapTaskNum, mapf)
      // task完成后通知coordinator
      CallFinishTask(task, taskfile)
    } else if task == "reduce" {
      do_reduce(taskfile, nmap, reduceTaskNum, reducef)
      CallFinishTask(task, strconv.Itoa(reduceTaskNum))
    }
    time.Sleep(time.Second)
  }
}
```

`do_map`, `do_reduce`可以参考`mrsequential.go`完成, coordinator中的`Done`逻辑较为简单，此处省略。

## Problems I met

1. 在RPC Call的Reply中worker收到的值均为0，原因是go中的RPC structure内的field首字母必须要大写。

   - 在 Go 中，首字母大写的标识符是导出的，可以被其他包访问。
   - 如果字段的首字母是小写的，则它们只能在定义它们的包内访问，相当于是私有的。

   ```go
   package main
   import "fmt"
   // MyStruct 是一个示例结构体
   type MyStruct struct {
       PublicField  int    // 导出字段
       privateField string // 非导出字段（私有）
   }
   
   func main() {
       // 创建结构体实例
       myInstance := MyStruct{
           PublicField:  42,
           privateField: "secret",
       }
       // 访问导出字段
       fmt.Println("PublicField:", myInstance.PublicField)
       // 访问非导出字段会导致编译错误
       // fmt.Println("privateField:", myInstance.privateField)
   }
   ```

2. 在运行map parallelism/reduce parallelism测试时，报错`fatal error: concurrent map writes`

   由于多个worker的RPC call可能同时访问coordinator中的各种map结构，所以coordinator中需要锁来解决concurrency的问题。

3. 在运行early exit测试时，程序不通过测试

   reduce任务必须在所有map任务完成后才可以开始，即在reduce任务前必须判断原始文件的file Status均为Done，若有文件状态是Process，worker进程也需进入等待状态。
