---
layout: post
title:      "QEMU/KVM中的网络虚拟化--Part1 Overview"
date:       2023-06-24 09:00:00
author:     zxy
math: true
categories: ["Coding", "Network"]
tags: ["vm", "network"]
post: true
---
这是个网络虚拟化系列博客从最基础的QEMU网络虚拟化的配置，逐步介绍网络虚拟化技术类型、原理和源码实现。这篇博客是网络虚拟化系列博客的第一篇，主要介绍QEMU中对网络虚拟化相关的high-level ideas、QEMU中网络参数的配置说明以及概括了不同的网络连接方式。

## Prerequist & Important terms to know

1. NAT协议
2. NIC (Network interface card)

## basic network command

1. `hostname -I` 查看本机IP地址
2. `ip a`
3. `ip route`
4. `brctl show`
5. `arping`

## High-level ideas

我们拿到一台新的主机，配置主机的网络，让电脑可以联网，至少需要满足以下几个条件：

1. 主机上有物理网卡/无限网卡
2. 主机通过网线/Wi-Fi连接到路由器上
3. 配置主机的IP、网关IP地址、DNS服务器IP

同样的，对于一台虚拟机来说，要让虚拟机可以上网，虚拟机也需要满足以下几个条件：

1. 虚拟机上有虚拟网卡（vNIC）
    - 从逻辑上来说，这属于虚拟机（guest），叫做前端设备，比如虚拟网卡，前端主要负责与虚拟机打交道，管理虚拟机的 I/O
2. 虚拟机通过一些设置连接到路由器上
    - 从逻辑上来说物理机（host）上有对应的接口来处理虚拟机请求，叫做后端设备，比如网卡对应的tap后端设备，后端主要与宿主机打交道，将虚拟机的I/O请求转换为宿主机上的I/O请求。
    - 一些后端设备有：
        1. "user" back-end:通过 NAT 提供对主机网络的访问
        2. "tap" back-end:允许guest直接访问host网络
        3. "socket" back-end: 用于连接多个 QEMU 实例为guest模拟共享网络

从QEMU的角度来看，对网络设备的虚拟化就由前后端两个部分组成：

- the virtual network device that is provided to the guest (e.g. a PCI network card).
- the network backend that interacts with the emulated NIC (e.g. puts packets onto the host's network).

## QEMU network configuration in details
现在我们来看一个qemu启动虚拟机时的网络参数设置，来理解虚拟机中的网络。

```shell
qemu-system-x86_64 -hda /path/to/hda.img -net nic,model=e1000,netdev=foo -netdev tap,ifname=tap0,script=no,downscript=no,id=foo
```

其中，`-net`是一个比较大类的参数项，用于为guest设置网络连接，其既可以用于设置前端设备也可以用于设置后端设备

- `-net nic`表示在虚拟机中创建虚拟网卡（前端设备）
    - `model`表示创建网卡类型，e1000,rtl8139,virtio等等
    - `netdev`表示所使用的后端设备id
- `-net user`表示设置虚拟NAT子网（后端设备），通过 NAT 提供对主机网络的访问

`-netdev`仅用于设置网络后端设备

- `tap`表示使用tap设备作为后端
- `ifname`表示 tap设备的名字
- `script`表示虚拟机在打开tap网卡的时候需要执行的脚本，这里设置为no
- `downscript`表示虚拟机在关闭的时候需要执行的脚本，这里设置为no
- `id`表示该后端设备的id

`-device`用于说明虚拟设备，可以直接用来制定对应的网卡，如`-netdev <backend>,id=<id> -device <dev>,netdev=<id>`

在qemu v2.12引入了`-nic`选项用于配置网卡，该选项可以“快速创建网络的前端（虚拟网卡）和宿主中的后端设备”。
如`-nic tap,model=e1000`，tap表示后端设备，e1000表示前端的网卡类型。

> By default QEMU will create a SLiRP user network backend and an appropriate virtual network device for the guest (eg an E1000 PCI card for most x86 PC guests), as if you had typed `-net nic -net user` on your command line.

## VM网络连接方式
根据QEMU对后端设备的不同实现，guests总共有四种方式和外部网络进行通讯：user mode, socket redirection, Tap and VDE networking.

一般来说，对虚拟机而言，外部网络可以分为host, 其他虚拟机, 局域网(LAN)以及互联网，这四种网络连接方式可以访问的网络范围不同，所需host系统权限也不一样，这四种方式可以访问的网络概括及所需host权限总结如下：

| Mode         | Privilege | VM→Host | **VM←Host** | **VM1↔VM2** | **VM→Net/LAN** | **VM←Net/LAN** |
| ------------ | --------- | ------- | ----------- | ----------- | -------------- | -------------- |
| User (Slirp) | user      | +       | 端口转发    | -           | +              | 端口转发       |
| Socket       |           |         |             |             |                |                |
| Tap          |           |         |             |             |                |                |
| VDE          |           |         |             |             |                |                |

后续的blog会通过一些简单的实验来理解这四种网络连接方式、原理以及QEMU相关源码分析。

1. [User mode netwroking](https://zxxyy.github.io/posts/qemu-network-slirp/)
2. Tap networking
3. socket networking
4. Virtio networking



## Reference
1. [QEMU Documentation/Networking](https://wiki.qemu.org/Documentation/Networking#Network_Basics)
2. [Configuring Guest Networking](https://www.linux-kvm.org/page/Networking)
3. [Archlinux QEMU networking](https://wiki.archlinux.org/title/QEMU#Networking)
4. [Virtualbox virtual networking](https://www.virtualbox.org/manual/ch06.html)
5. [VM Networking](https://joshrosso.com/docs/2020/2020-11-13-vm-networks/)
6. [QEMU's new -nic command line option](https://www.qemu.org/2018/05/31/nic-parameter/)
7. [QEMU cheetsheet for full emulation](https://wangziqi2013.github.io/article/2022/03/09/qemu-cheat-sheet.html)
8. [Deep dive into Virtio-networking and vhost-net](https://www.redhat.com/en/blog/deep-dive-virtio-networking-and-vhost-net)
9. [KVM Virtual Networking Concepts](https://kb.novaordis.com/index.php/KVM_Virtual_Networking_Concepts)