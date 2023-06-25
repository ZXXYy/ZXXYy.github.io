---
title:      "VM中的网络连接--以qemu为例(TBC)"
date:       2023-06-24 09:00:00
author:     zxy
math: true
categories: ["Coding", "Network"]
tags: ["vm", "network"]
post: true
---
[libvirt](https://libvirt.org/)--virtualization API:Exposes a consistent API atop many virtualization technologies. APIs are consumed by client tools for provisioning and managing VMs.
virt-manager--GUI to manage KVM, qemu/kvm, xen, and lxc.
virtio

## Prerequist & Important terms to know

1. NAT协议
2. NIC (Network interface card)

## basic network command

1. `hostname -I` 查看本机IP地址
2. `brctl show`
3. `arping`

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


一般来说，虚拟机-同一host上的其他虚拟机-host-host所在局域网-外部网络进行网络连接，从而进行通讯。



- virtual NICs--virbr0-nic/vnet1/vnet2: dummy interface, connected to the virtual ports on the virtual switch
- virtual switches--virbr0: 在host中运行的switch，VM可以直接接入这个switch，layer2设备，(virtual bridge)passes frames between nodes. 
- virtual dhcp server





There are four ways that QEMU guests can be connected: user mode, socket redirection, Tap and VDE networking.

## user networking / NAT (slirp)
虚拟机在外部网络上共享host的IP地址，host为虚拟机发出的网络流量进行网络地址转换（NAT）。
由qemu虚拟出一台DHCP服务器和网关，IP默认是10.0.2.2
再给guest分配一个IP，默认是10.0.2.15
每一台guest都有同样的私有地址
Use case:

- You want a simple way for your virtual machine to access to the host, to the internet or to resources available on your local network.
- You don't need to access your guest from the network or from another guest.
- You are ready to take a huge performance hit.
- Warning: User networking does not support a number of networking features like ICMP. Certain applications (like ping) may not function properly.

`qemu-system-x86_64 -hda /path/to/hda.img -netdev user,id=user.0 -device e1000,netdev=user.0`

## 桥接模式--Bridge Connection

### Private Virtual Bridge

是一个隔离的网络
Use case:

- You want to set up a private network between 2 or more virtual machines. This network won't be seen from the other virtual machines nor from the real network.

### Public Bridge

虚拟机显示为局域网上的另外一台电脑，会消耗宿主所在局域网的IP地址
Use case:

- You want to assign IP addresses to your virtual machines and make them accessible from your local network
- You also want performance out of your virtual machine

## 仅主机 (host-only)

## QEMU中的实现


## Reference
1. [QEMU Documentation/Networking](https://wiki.qemu.org/Documentation/Networking#Network_Basics)
2. [Configuring Guest Networking](https://www.linux-kvm.org/page/Networking)
3. [Archlinux QEMU networking](https://wiki.archlinux.org/title/QEMU#Networking)
4. [Virtualbox virtual networking](https://www.virtualbox.org/manual/ch06.html)
5. [VM Networking](https://joshrosso.com/docs/2020/2020-11-13-vm-networks/)
6. [QEMU's new -nic command line option](https://www.qemu.org/2018/05/31/nic-parameter/)