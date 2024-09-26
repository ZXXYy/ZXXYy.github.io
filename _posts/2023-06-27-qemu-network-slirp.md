---
layout: post
title:      "QEMU/KVM中的网络虚拟化--Part2 User Networking"
date:       2023-06-24 09:00:00
author:     zxy
math: true
categories: ["Coding", "Network"]
tags: ["vm", "network"]
post: true
---
这篇博客是网络虚拟化系列博客的第二篇，主要介绍QEMU网络虚拟化中的User mode networking，这里只讨论QEMU/KVM基于硬件虚拟化的场景。
这篇blog会通过一个简单的实验，介绍QEMU里的user networking，说明网络包的收发流程，概览QEMU内部的User Networking相关源码。

### Experiment Setup

0. QEMU版本 

   ```shell
   $ qemu-system-x86_64 --version
   QEMU emulator version 6.2.0 (Debian 1:6.2+dfsg-2ubuntu6.11)
   Copyright (c) 2003-2021 Fabrice Bellard and the QEMU Project developers
   ```

1. 下载一个虚拟机云镜像，适用于非图形化界面

   `wget https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img`

2. 配置该镜像登录密码

   > By default, Ubuntu cloud images are configured to use SSH key authentication and disable password authentication for the root user.
   >
   > 所以，我们直接去更改了镜像中的设置

   ```shell
   # install virt-customize tool
   sudo apt install libguestfs-tools
   # By default, the system image will only have one user, root, and the password is not configured yet.
   virt-customize -a bionic-server-cloudimg-amd64.img \
   	--root-password password:<your-password> 
   ```

3. 由于我们下载的是个云镜像，在启动前我们需要通过cloud-init对其进行一些配置

   > cloud-init 是一种用于云环境的开源工具，用于自定义和配置云实例的启动过程。它能够在实例首次启动时执行各种任务，包括设置主机名、配置网络、创建用户账户、安装软件包、挂载存储卷等。

   - 新建一个文件`user-data.yaml`

   - 编辑`user-data.yaml`，配置/激活网络

     ```yaml
     #cloud-config
     network:
       version: 2
       ethernets:
         eth0:
           dhcp4: true
     ```

   - 通过`cloud-localds`创建cloud-init配置image

     ```shell
     sudo apt-get install cloud-image-utils
     cloud-localds cloud.img user-data.yaml
     ```

4. 运行虚拟机

   ```shell
   qemu-system-x86_64 -enable-kvm -cpu host \                                                                                        	-m size=1G  -nographic \
     -hda bionic-server-cloudimg-amd64.img -drive file=cloud.img
   ```

   使用username: root password: \<your-password\>登录

   `-enable-kvm` 表示使用kvm加速虚拟化

   `-cpu host` 确保虚拟机使用与宿主机相同的 CPU 模型，包括所有可用的扩展和功能

   `-m size=1G` 设置虚拟机内存大小为1G

   `-nographic`表示使用非图形化界面

   `-drive file=cloud.img` 是用于指定虚拟机中的磁盘镜像文件，在系统启动时，cloud-init会从该文件中读取用户的配置

   `-hda` 是用于指定要连接到虚拟机的主磁盘镜像文件的路径和名称

### User networking  (slirp)

QEMU中默认的网络连接就是user networking，使用最简单的命令就是以该种方式建立网络连接

以下三种命令是等价的

```shell
1. qemu-system-x86_64 -nographic -hda ubuntu.img 
2. qemu-system-x86_64 -nographic -hda ubuntu.img -net nic -net user
3. qemu-system-x86_64 -nographic -hda ubuntu.img -netdev user,id=user.0 -device e1000,netdev=user.0
```

在VM中查看虚拟机IP为10.0.2.15：![user-vm-ip](/assets/img/in-post/2023-06-26-qemu-net/user-vm-ip.png)

在VM中查看虚拟机的路由表，查看默认网关IP为10.0.2.2

![user-vm-route](/assets/img/in-post/2023-06-26-qemu-net/user-vm-route.png)

在host中查看host的IP为10.162.185.114

![host_ip](/assets/img/in-post/2023-06-26-qemu-net/host_ip.png)

如果我们再开起一个虚拟机VM2，会发现VM2中的IP地址也是10.0.2.15，即如果使用这种方式每一台guest都有同样的私有IP地址。

在这种模式下，虚拟机在外部网络上共享host的IP地址，host为虚拟机发出的网络流量进行网络地址转换（NAT）。
由QEMU虚拟出一台DHCP服务器和网关，IP默认是10.0.2.2

QEMU也会虚拟出一个虚拟网卡并给guest分配一个IP，默认是10.0.2.15

在host中使用`nc -lv 1234`监听端口1234

在guest中`nc -v 10.0.2.2 1234`连接host

可以看到guest中的消息可以发送到host中，使用`nc -v 10.162.185.114 1234`也可以和host进行通讯

![nc_guest](/assets/img/in-post/2023-06-26-qemu-net/nc_guest.jpg)

但是在host中不可以连到guest上

![nc_host](/assets/img/in-post/2023-06-26-qemu-net/nc_host.png)

上述实验，对应的整个网络拓扑结构如下：

![topology](/assets/img/in-post/2023-06-26-qemu-net/topology.png)

## Network Package Flow

在User network模式下，虚拟机中网络包的发送流程（TX）如下图所示

1. guest内核将tx报文信息写入TX ring，通知网卡处理tx报文
2. 触发VM Exit，KVM处理VM Exit，判断为I/O操作
3. KVM返回用户态QEMU进程
4. QEMU处理I/O，根据MMIO写入地址，判断为网络请求
5. QEMU调用虚拟网卡相关函数，更新虚拟网卡状态
6. 虚拟网卡中的函数将请求发送给网络后端，此处为SLiRP
7. SLiRP对网络包进行一系列处理后，通过系统调用，将改网络包发送给host上真正的网卡
8. 调用层层返回，QEMU通过ioctl `KVM_RUN`再次进入KVM
9. KVM通过VM Entry恢复guest执行

![TX Package Flow](/assets/img/in-post/2023-06-26-qemu-net/flow.png)

## QEMU SLiRP Source Code Analysis

通过GDB调试QEMU源码（调试方式见[GDB入门](https://zxxyy.github.io/posts/gdb_command/#qemu%E4%B8%8Egdb%E7%9A%84%E4%BA%A4%E4%BA%92)），在`slirp_input`处设置断点，可以看虚拟机发送网络请求时的整个函数调用栈。

![gdb_qemu_1](/assets/img/in-post/2023-06-26-qemu-net/gdb_qemu_1.png)

![gdb_qemu_2](/assets/img/in-post/2023-06-26-qemu-net/gdb_qemu_2.png)

-  `#29 kvm_cpu_exec` 这个函数中，捕获到VM_Exit，并且发现Exit原因为KVM_EXIT_MMIO，并调用`address_space_rw`处理

  ```c++
  int kvm_cpu_exec(CPUState *cpu){
    ...
    do {
         ...
         // VM entry
         run_ret = kvm_vcpu_ioctl(cpu, KVM_RUN, 0);
       	 trace_kvm_run_exit(cpu->cpu_index, run->exit_reason);
         switch (run->exit_reason) {
          case KVM_EXIT_IO: ...
          case KVM_EXIT_MMIO:
              DPRINTF("handle_mmio\n");
              /* Called outside BQL */
              address_space_rw(&address_space_memory,
                               run->mmio.phys_addr, attrs,
                               run->mmio.data,
                               run->mmio.len,
                               run->mmio.is_write);
              ret = 0;
              break;
           ...
         }
    }while (ret == 0);
  }
  ```

- `#21 e1000_mmio_write` 模拟e1000网卡，把传进来的数据更新到虚拟网卡中，然后一层层调用`#16 e1000_send_packet()`，再通过`#15 qemu_send_packet()`发送给模拟出来的网络后端
- `#15 qemu_send_packet()`转接到slirp 所提供的`NetClientInfo->receive()`API, 即`net_slirp_receive()`，最后通过syscall把请求发送给host kernel

对其中重要的调用流程总结如下图，cited from [QEMU network introduction(architecture)](https://cwshu.github.io/arm_virt_notes/notes/qemu_kvm/qemu_net.html#introduction-architecture)

![../../_images/e1000_slirp.png](https://cwshu.github.io/arm_virt_notes/_images/e1000_slirp.png)

对于每一个函数的细节以及QEMU中的各种网络结构之后再开新的blog，边学边记吧～

## Reference

1. [QEMU虚拟机网络模拟](https://www.owalle.com/2019/12/26/network-in-vm/)

2. [Lab 6: Network Driver (default final project)](https://pdos.csail.mit.edu/6.828/2017/labs/lab6/)

3. [网络虚拟化——QEMU虚拟网卡](https://blog.csdn.net/dillanzhou/article/details/120169734)

4. [Deep dive into Virtio-networking and vhost-net](https://www.redhat.com/en/blog/deep-dive-virtio-networking-and-vhost-net)

