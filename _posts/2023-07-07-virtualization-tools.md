---
title:      "Virtualization Tools Overwiew for Layman"
date:       2023-07-07 09:00:00
author:     zxy
math: true
categories: ["Coding", "System", "Virtualization"]
tags: ["vm"]
post: true
---

作为想入门云服务、虚拟化的初学者，去搜索相关的信息、学习资料，就会看到各种各样的工具、概念，完全迷失方向。qemu，kvm，然后又出来一个qemu/kvm。好不容易弄明白之后，又蹦出来一个libvirt，和libvirt相关的又有virsh, virt-manager等等。下载一些镜像的时候又有cloud-init 。看一些云厂商AWS，Azure等文章突然又出现SasS, PaaS, IaaS。还有openstack要来凑热闹，openstack里面又有n个名词，比如Horizon，Nova，Cinder等等。这个时候，只有“What the fuck?"可以形容内心的心情。

所以这篇博客想要从high-level以及这些技术的演化路线的角度来总结一下主流的开源虚拟化概念/工具间的相关性，并且给出在一台服务器上通过libvirt/qemu/kvm配置一台虚拟机的具体过程来加深理解。

## What is cloud service

云服务（Cloud Service）可以分为三种基础的模型--IaaS, PaaS, SaaS，Albert Barron在2014年时提出的Pizza-as-a-Service比喻很好的解释了这三种基础模型的区别，这里我们采用饺子的比喻。如果要在家做饺子，我们需要剁肉馅，擀饺子皮，包饺子，煮饺子，最后把饺子上上餐桌，这里面的每一个步骤都需要我们亲力亲为。在计算的场景下，也就是我们需要自己去买CPU、硬盘，搭建服务器，配置网络，搭建操作系统，配置系统环境，下载必要的应用，我们才可以真正使用服务器。

- SaaS (Software as a Service)：SaaS就相当于我们去餐厅吃了份饺子，有服务员会把饺子直接端到我们面前。也就是说，厂商直接把开发好的应用提供给我们了，我们直接就可以拿来用了。比如我们访问google docs就可以直接对文档进行多人同步编辑，数据也在云存着。

- PaaS (Platform as a Service)：PaaS就相当于我们点了个饺子的外卖，我们需要做的事情只是上饺子。也就是说，厂商把一些软件开发的环境也配置好了，比如操作系统、中间件、运行时环境等等。也就说可以直接在这一层上根据自己的需求开发/测试自己的应用。
- IaaS (Infrastructure as a Service)：IaaS就相当于我们去买了一袋速冻饺子，我们只需要去煮饺子、上饺子就好了。也就是说，有厂商给我们搭建好了一组服务器，我们只要向他们租一台服务器（虚拟机），直接就可以使用了，不需要再自己去买硬盘这些设备了。在这一层，最重要的技术就是虚拟化技术。

![SCR-20230709-oqkh](/Users/zhengxiaoye/Library/Application Support/typora-user-images/SCR-20230709-oqkh.png)

## Virtualization Framework Generalized

有了上面的big picture后，我们现在更加关注IaaS层面，主要是从high-level的角度分析虚拟化相关的概念和开源工具。以下是这些工具/概念发展的时间线：





https://dl.acm.org/doi/fullHtml/10.1145/3365199

http://cs312.osuosl.org/slides/21_virtualization.html#5

https://www.socallinuxexpo.org/sites/default/files/presentations/KVM%2C%20OpenStack%2C%20and%20the%20Open%20Cloud%20-%20SCaLE%20-%20ANJ%2021Feb15.pdf

## Tools



![img](https://linux-blog.anracom.com/wp-content/uploads/2021/02/Spice_basic_800.gif)

![The libvirt & virtualization tools software development platform](http://berrange.com/wp-content/uploads/2012/01/virt-platform-2.png)



https://www.berrange.com/posts/2012/01/13/the-libvirt-virtualization-tools-software-development-platform/

System tools

- kvm

  - Verify. the kvm kernel modules are properly loaded

    `lsmod | grep kvm`

    `usermod --append --groups=kvm,libvirt ${USER}`

- Xen

- qemu

- libvirt

  - Hypervisor 比如 qemu-kvm 的命令行虚拟机管理工具参数众多，难于使用。

  - Hypervisor 种类众多，没有统一的编程接口来管理它们，这对云环境来说非常重要。

  - 没有统一的方式来方便地定义虚拟机相关的各种可管理对象。

  - is a toolkit to manage [virtualization platforms](https://libvirt.org/platforms.html)

  - ![img](https://upload.wikimedia.org/wikipedia/commons/thumb/d/d0/Libvirt_support.svg/300px-Libvirt_support.svg.png)

  - the goal of the libvirt library is to provide a common and stable layer to manage VMs running on a hypervisor.

  - In Linux, you will have noticed some of the processes are deamonized. The libvirt process is also deamonized, and it is called libvirtd.

  - what exactly happens when a libvirt client such as virsh or virt-manager requests a service from `libvirtd`. 

    a server side daemon and driver required to manage the virtualization capabilities of KVM hypervisor.

    ```shell
    sudo apt install libvirt-daemon-system
    systemctl start libvirtd
    systemctl enable libvirtd
    ```

    

  - some common virtual machine management tools (e.g. virsh, virt-install, virt-manager, etc.) and cloud computing framework platforms (e.g. OpenStack, OpenNebula, Eucalyptus, etc.) are available

  - https://subscription.packtpub.com/book/cloud-&-networking/9781784399054/2/ch02lvl1sec17/getting-acquainted-with-libvirt-and-its-implementation

  - https://www.vyomtech.com/2013/12/17/libvirt_the_unsung_hero_of_cloud_computing.html

  - https://libvirt.org/api.html

  - https://www.cnblogs.com/sammyliu/p/4558638.html https://www.sobyte.net/post/2022-05/libvirt/

  - https://www.youtube.com/watch?v=gI66ZlnLs0Y

  - 

- Openstack https://www.youtube.com/watch?v=IseEhw-Dxrc

  - 开源云操作系统

  - OpenStack is a cloud operating system that controls large pools of compute, storage, and networking resources throughout a datacenter, all managed and provisioned through APIs with common authentication mechanisms.

  - Nova: a component in the openstack

    

- Cloud-init
  - `cloud-init` is a tool used in cloud computing environments to customize virtual machines at boot time
    - It allows for the automatic configuration of network settings, user accounts, and other system settings that are required for the virtual machine to function properly.
    -  `cloud-init` works by reading configuration data from a variety of sources, including user data, metadata, and vendor data.
  - A cloud backing image is an image that has been created with `cloud-init` support built-in.  Many cloud providers offer a selection of pre-configured cloud backing images for common operating systems and applications, which can be easily launched from their cloud dashboard or API.

user/client tools

- virsh
  - The command line client interface of libvirt is the binary called `virsh`.

- Virt-manager
  - Virt-install
  - Virt-viewer

http://www.pixelbeat.org/docs/openstack_libvirt_images/



>  后续如果有时间，会对kvm, libvirt, openstack做更加详细的介绍。

## Reference

1. [What is cloud computing](https://dev.to/inesattia/openstack-11pd)
