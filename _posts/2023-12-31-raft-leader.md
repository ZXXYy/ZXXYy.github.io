---
layout: post
title:      "MIT6.824 Lab2A Raft Leader Election"
date:       2023-12-31 09:00:00
author:     zxy
math: true
categories: ["Coding", "Courses", "MIT6.824"]
tags: ["MIT6.824"]
post: true
---

> 阅读本篇blog前需要理解raft leader election的基本概念，阅读Raft论文Section1、2、5，弄明白Figure2，以及尝试实现Lab2A
>
> Lab Description: [https://pdos.csail.mit.edu/6.824/labs/lab-raft.html](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html)
>
> Raft Lecture: [https://youtu.be/R2-9bsKmEbo](https://youtu.be/R2-9bsKmEbo)
>
> Raft Lecture Notes: [https://pdos.csail.mit.edu/6.824/notes/l-raft.txt](https://pdos.csail.mit.edu/6.824/notes/l-raft.txt)
>
> Raft paper: [https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)
>
> Raft official website: [https://raft.github.io/](https://raft.github.io/)
>
> Raft可视化动画: [https://thesecretlivesofdata.com/raft/](https://thesecretlivesofdata.com/raft/)

这篇博客中记录我实现Raft Leader Election这个实验的过程，以及实现过程中遇到的问题。这可能比仅仅看最后的源码实现，可以带给你更多的帮助。这篇博客对应的源码也已开源[https://github.com/ZXXYy/mit6.824-2023](https://github.com/ZXXYy/mit6.824-2023)

## Prelimary Setup

由于Raft算法的随机性，各个peer的超时时间、投票给谁都不确定。运行一次 `go test -run 2A`通过所有测试，并不代表2A的实现没有bug。因此，我们需要多次测试程序，要保证代码的正确性。MIT6.824的助教用Python写了一个批量测试的脚本，我们可以直接拿来用，链接在 [Utility for running MIT 6.824 lab test in parallel and saving failed logs](https://gist.github.com/JJGO/0d73540ef7cc2f066cb535156b7cbdab)。

把该脚本直接拷贝下来，存在 `raft` 目录下，执行 `python3 dstest.py 2A -n 10`   就可以批量测试代码。 `-n 10` 表示测试多少次，默认是 10。如下是批量测试运行中的一个截图。

![截屏2024-01-03 10.00.26](/assets/img/in-post/2023-12-31-raft-leader/test.png)

测试100次后，成功的截图如下：

![截屏2024-01-03 10.08.40](/assets/img/in-post/2023-12-31-raft-leader/result.png)

关于高效的进行日志打印相关内容可以参考这篇blog [Debugging by Pretty Printing](https://blog.josejg.com/debugging-pretty/)。

## Raft Leader Election Keypoint

- **核心思想**：在任意一个时间点，至多只有一个leader，由这个leader来接收和分发用户的请求，来避免split-brain的问题。所有的信息都从leader流向follower。
- **目标**: 基于majority rule，选举出一个leader。

## My Implementation

在通读Lab2A的实验说明之后，可以理解本次实验需要改动的代码只有一个文件`src/raft/raft.go`，在`raft.go`内主要需要实现四个内容(raft数据结构，ticker()，requestVote RPC，Append Entries RPC)：

- raft数据结构及`make()`初始化raft数据结构
  - raft数据结构按照paper Figure2的内容设计，除此之外还需要添加3个额外的信息
    - heartbeat：用来记录心跳，记录leader上一次发送心跳包（即AppendEntries）的时间
    - electionTimer: 用来记录每一个raft服务心跳超时的时常，在make中以random的方式初始化，在这里我设置成150-350ms不等
    - isLeader: 用来记录当前raft服务器是否是leader

- `ticker()`

  - 主要任务是查看心跳包是否超时，如果超时了，把follower转变成为candidate，向其他rafts发送requestVote RPC

- `requestVote` RPC

  - requestVote的数据结构按照Figure2中的Request RPC设计，没有额外的要求。

    ```go
    type RequestVoteArgs struct {
    	// Your data here (2A, 2B).
    	Term         int // the candidate's term
    	CandidateId  int // the candidate requesting the vote
    	LastLogIndex int // the index of the candidate's last log entry
    	LastLogTerm  int // the term of the candidate's last log entry
    }
    
    // example RequestVote RPC reply structure.
    // field names must start with capital letters!
    type RequestVoteReply struct {
    	// Your data here (2A).
    	Term        int  // the current term of the peer
    	VoteGranted bool // true if the peer granted the vote
    }
    
    ```

  - RequestVote RPC主要做的事情是:
    - 发送requestVote请求，根据返回的请求内容，判断投票数是否大于整体raft数量的1/2，进而判断是否要转变成为leader
  - RequestVote RPC handler主要做的事情是：判断要不要投票给发起该请求的candidate
    - 如果当前args.Term < rf.currentTerm，说明candidate的信息未完全同步，不能当leader，投反对票
    - 如果当前args.Term > rf.currentTerm，表明raft在这个term当中未投过票; 如果args.Term == rf.currentTerm，根据rf.votedFor的值来判断是否投过票
    - 如果raft未投过票，并且candidate的最后一个log term大于等于当前raft的log term，log index也大于等于当前raft的log index，grant vote

- `AppendEntries` RPC

  - AppendEntries的数据结构在2A中只需用到Term

    ```go
    type AppendEntriesArgs struct {
    	Term     int // the leader's term
    	LeaderId int // the leader's id
    }
    
    type AppendEntriesReply struct{}
    ```

  - AppendEntries RPC主要做的事情是：2A中如果当前raft是leader，就发送心跳包
  - AppendEntries RPC handler主要做的事情是：更新heartbeat信息，如果当前term小于请求raft的term，将isLeader置为false

## Problems I met

1. 在网络发生问题时，有信息被阻塞，导致在限定时间内没有leader被选举出来，最后发现是在发送RPC call时应该新开一个现场，即用`go rf.sendRequestVote(i, &args, &reply)`， `go rf.sendAppendEntries(i)`。 不然，当网络发生异常时，程序会直接被阻塞住。



## 一些好的blogs

[journey-to-mit-6-824-lab-2a-raft-leader-election](https://medium.com/codex/journey-to-mit-6-824-lab-2a-raft-leader-election-974087a55740)

[mit-6.824-lab2-raft](https://blog.rayzhang.top/2022/11/09/mit-6.824-lab2-raft/index.html#%E6%89%B9%E9%87%8F%E6%B5%8B%E8%AF%95)
