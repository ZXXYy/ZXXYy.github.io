---
layout: post
title:      "Solana Core Concepts: Accounts, Programs, and PDAs"
date:       2024-09-29 09:00:00-0400
author:     zxy
math: true
categories: Solana Rust
tags: Blockchain
---
> 这篇blog主要整理了Solana中各种概念间的关系。在阅读前最好先阅读Solana文档。

Solana的account model 说白了就是一个key-value store，通过地址访问account数据。总的来说，在这个account model中：
- key = address = public key，用来索引数据。
    - 其中有一种特殊类型的key，叫做PDA(Program Derived Address)，这个地址通过pre-defined string生成，没有对应的private key，not on Ed25519 curve （后续详细介绍，这里有个印象就可以）。
- value = account， account通过其中的`executable`字段，具体分成两种类型
    - stateless account = program = smart contract，program又有两种
        - native programs (built-in with Solana Runtime)
            - System Program (`11111111111111111111111111111111`):
                - 用于创建account, allocate account data space, 改变account的ownership
                - 一个account刚刚创建时的owner是system program，system program可以把ownership转移给另外的program 
            - BPF loader：所有customized programs的owner
                - 用于deploy, upgrade, execute program.
        - customized programs
            - program中存了可执行代码的地址信息以及address authorized to make changes to the program的信息
    ![program_account](/assets/img/in-post/2024-09-29-Solana/program.png)
            - 和以太坊不同，Solana上的程序可以被具有upgrade authority的account修改，当upgrade authority被设置为`None`时，program变成immutable
            - verifiable program：源码公开的program，可以通过SolanaFM查询
    - stateful account = data account
    - 每一个account都有一个owner，只有owner可以修改account中的数据。data account的owner是program，program的owner也另一个program。

### PDA
有了上面的概念之后我们来解释PDA。为了充分理解PDA的概念，我们来看一个一般的场景。
如下图所示，当我们要修改一个data account中的数据时，我们需要校验两个点
1. 修改account的程序是不是这个account的owner
2. 这个account的所有者是否用它的private key authoriz了这个交易，即是否用account private key对transaction进行签名。
<img src="/assets/img/in-post/2024-09-29-Solana/pda1.png" alt="pda" width="800" >
<!-- ![pda]() -->

然而，很多时候我们不希望必须要由client使用private签名后，我们才可以修改account data，一个program应该可以自己操控自己的数据，即第二条条件不满足。比如，在一个program中调用另外一个program修改数据的操作应该是可行的。
<img src="/assets/img/in-post/2024-09-29-Solana/pda_client.png" alt="pda_client" width="800" >

<!-- 
![pda_client](/assets/img/in-post/2024-09-29-Solana/pda_client.png) -->
PDA就是用来解决这个场景下的问题的，即通过pre-defined strings来生成一个地址索引，用于transaction signing。这个时候，我们不需要知道account private key，只需要知道预先定义好的string，就用program可以发起一个transaction，修改account info.
具体来说，生成PDA的过程包括：
1. Seeds：用户提供的pre-defined输入，一般包括account addres和一些'_'下划线等字符+program id
2. Bump seed：用来保证生成的PDA不在Ed25519曲线上
![pda_derivation](/assets/img/in-post/2024-09-29-Solana/pda-derivation.svg)

> Deriving a PDA does not automatically create an on-chain account.

总的来说，PDA可以用来Sign for CPIs并且只需要通过pre-defined strings就可以访问到account的数据，对编程来说更加方便，不需要去查询account的地址。

### Token Program & Token Account & Associated Token Account
有了program, data account, pda的概念之后，理解token Program, mint account,token account,  associated token account就变得非常容易。其实就是这三个概念在token场景下的应用。
- token program(`TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`): contains all the instruction logic for interacting with tokens on the network
    - 可以create mint account/token account, 可以transfer or mint tokens
- mint account: 表示一种特定的token类型，数据中存了token supply,mint authority (address authorized to create new units of a token)等等
- token account: 表示某一个用户拥有多少某一种token
    - token account data中包括了持有的token的mint address(`Mint`),持有者的wallet address(`owner`)以及持有的金额数量(`amount`)
    - For a wallet to own units of a certain token, it needs to create a token account for a specific type of token (mint) that designates the wallet as the owner of the token account. A wallet can create multiple token accounts for the same type of token, but each token account can only be owned by one wallet and hold units of one type of token.
    ![token](/assets/img/in-post/2024-09-29-Solana/token.png)
- Associated Token Account 是通过mint token address+wallet address生成的PDA
    ![associated-token-account](/assets/img/in-post/2024-09-29-Solana/associated-token-account.svg)


### Reference
1. [Solana Documents](https://solana.com/docs/core/pda)
2. [StackExchange What is a Program Derived Address (PDA) exactly?](https://solana.stackexchange.com/questions/26/what-is-a-program-derived-address-pda-exactly)
3. [Solana Tutorial Video](https://www.youtube.com/watch?v=SCS6jt8sye0&list=PLilwLeBwGuK51Ji870apdb88dnBr1Xqhm&index=7)