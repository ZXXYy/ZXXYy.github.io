---
title:      "KVM主要结构分析--使用Rust实现KVM userspace"
date:       2023-07-03 09:00:00
author:     zxy
math: true
categories: ["Coding", "System", "Virtualization"]
tags: ["vm", "kvm", "Virtualization"]
post: false
---

通过`lsmod | grep kvm`可以看到和kvm相关的有两个模块`kvm`和`kvm_intel`

这两个模块位于`/lib/modules/5.19.0-46-generic/kernel/arch/x86/kvm`

```shell
cd /lib/modules/5.19.0-46-generic/kernel/arch/x86/kvm
ls
```

kvm.ko初始化代码不做任何事情，相当于只是把代码加到了内存中，KVM的开启和关闭都是由kvm-intel.ko完成的，当KVM 初始化完成之后，会向用户空间呈现KVM的接口，这些接口是由kvm.ko 导出的，用户程序调用这些接口时，kvm.ko中的通用代码反过来会调 用kvm-intel.ko架构中的相关代码。

因此，在内核的kvm相关代码中应该会有`module_init`函数来初始化内核模块，`module_exit`函数来注销内核模块。

> [`module_init(x)`](https://www.kernel.org/doc/html/v4.14/driver-api/basics.html#c.module_init) will either be called during `do_initcalls()` (if builtin) or at module insertion time (if a module). There can only be one per module. The parameter x is the function to be run at kernel boot time or module insertion.
>
> [`module_exit(x)`](https://www.kernel.org/doc/html/v4.14/driver-api/basics.html#c.module_exit) will wrap the driver clean-up code with `cleanup_module()` when used with rmmod when the driver is a module. If the driver is statically compiled into the kernel, [`module_exit()`](https://www.kernel.org/doc/html/v4.14/driver-api/basics.html#c.module_exit) has no effect. There can only be one per module. The parameter x is the function to be run when driver is removed

 在`linux-4.2.8/arch/x86/kvm/vmx.c`这个路径下，我们可以看到，`module_init`调用`vmx_init`来初始化内核模块

在`linux-4.2.8/arch/x86/kvm/makefile`中，我们可以看到和kvm相关的文件

```makefile
KVM := ../../../virt/kvm
kvm-y			+= $(KVM)/kvm_main.o $(KVM)/coalesced_mmio.o \
				$(KVM)/eventfd.o $(KVM)/irqchip.o $(KVM)/vfio.o
kvm-$(CONFIG_KVM_ASYNC_PF)	+= $(KVM)/async_pf.o

kvm-y			+= x86.o mmu.o emulate.o i8259.o irq.o lapic.o \
			   i8254.o ioapic.o irq_comm.o cpuid.o pmu.o mtrr.o
kvm-$(CONFIG_KVM_DEVICE_ASSIGNMENT)	+= assigned-dev.o iommu.o
kvm-intel-y		+= vmx.o pmu_intel.o
kvm-amd-y		+= svm.o pmu_amd.o

obj-$(CONFIG_KVM)	+= kvm.o
obj-$(CONFIG_KVM_INTEL)	+= kvm-intel.o
obj-$(CONFIG_KVM_AMD)	+= kvm-amd.o
```

vmx.o, pmu_intel.o链接生成`kvm-intel.ko`

kvm_main.o,coalesced_mmio.o, eventfd.o, irqchip.o, vfio.o, x86.o等链接生成 `kvm.ko`

- ioapic.c是中断控制器代码

>  在Linux的makefile中`m` means module, `y` means built-in (stands for yes in the kernel config process)
>
> Kernel Makefiles are part of the `kbuild` system
>
> https://lwn.net/Articles/21835/

从`vmx_init`这个函数开始看，`vmx_init`把相关架构相关操作当作参数传递给 `kvm_init`

```c
// linux-4.2.8/arch/x86/kvm/vmx.c
static int __init vmx_init(void)
{
	int r = kvm_init(&vmx_x86_ops, sizeof(struct vcpu_vmx),
                     __alignof__(struct vcpu_vmx), THIS_MODULE);
	if (r)
		return r;
	...
	return 0;
}

// linux-4.2.8/virt/kvm/kvm_main.c
int kvm_init(void *opaque, unsigned vcpu_size, unsigned vcpu_align,
		  struct module *module){...}
```

我们一一来看其中的参数

- `vmx_x86_ops`一个`kvm_x86_ops`结构体。KVM的所有虚拟化实现（Intel和AMD）都会向KVM模块注册一个 kvm_x86_ops结构体，这样，KVM中的一些函数就是一个外壳，它可能首先会调用kvm_arch_xxx函数，表示的是调用CPU架构相关的函数，而如果kvm_arch_xxx函数需要调用到实现相关的代码，则会调用 kvm_x86_ops结构中的相关回调函数。此处的分析以x86架构的Intel实 现为例。

- `vcpu_size` VMX实现的VCPU结构体的大小
- `vcpu_align`
- `module`

在`kvm_init`这个函数中开启VMX模式。，...

```c
kvm_chardev_ops.owner = module; // "/dev/kvm"这个设备对应的fd的file_operations
kvm_vm_fops.owner = module;     // 创建的虚拟机对应的fd的file_operations
kvm_vcpu_fops.owner = module;   // 创建的VCPU对应的fd的file_operations
// 调用misc_register创建kvm_dev这个misc设备，将kvm_dev这个设备的 file_operations设置为kvm_chardev_ops。
r = misc_register(&kvm_dev);
if (r) {
  pr_err("kvm: misc device register failed\n");
  goto out_unreg;
}

static struct miscdevice kvm_dev = {
	KVM_MINOR,
	"kvm",
	&kvm_chardev_ops,
};
static struct file_operations kvm_chardev_ops = {
	.unlocked_ioctl = kvm_dev_ioctl,
	.compat_ioctl   = kvm_dev_ioctl,
	.llseek		= noop_llseek,
};

```



在`kvm_init`首先调用`kvm_arch_init(opaque)`

```c
// linux-4.2.8/arch/x86/kvm/x86.c
int kvm_arch_init(void *opaque)
{
	int r;
	struct kvm_x86_ops *ops = opaque;
  ...
    kvm_mmu_module_init //完成内存虚拟化的初始化工作
    kvm_set_mmio_spte_mask // 设置MMIO内存的标识符
    kvm_timer_init
    kvm_lapic_init
}
```

是`kvm_arch_hardware_setup` 调用`r = kvm_x86_ops->hardware_setup();`

```c
// linux-4.2.8/arch/x86/kvm/vmx.c
static __init int hardware_setup(void)
{
	int r = -ENOMEM, i, msr;
  ...
  // 首先分配了多个bitmap， 这些bitmap都是VMCS中可能需要用到的
  // 典型的如两个IO位图 vmx_io_bitmap_a/b，并且都设置为1，所以一开始会拦截所有的端口 读写。
  vmx_io_bitmap_a = (unsigned long *)__get_free_page(GFP_KERNEL);
	if (!vmx_io_bitmap_a)
		return r;
  vmx_io_bitmap_b = (unsigned long *)__get_free_page(GFP_KERNEL);
	if (!vmx_io_bitmap_b)
		goto out;
	...
  // 设置一个全局变量 vmcs_config，该函数根据查看CS的特性支持填写vmcs_config
  if (setup_vmcs_config(&vmcs_config) < 0) {
		r = -EIO;
		goto out8;
	}
```







virtual machine (VM), which represents everything associated with one emulated system, including memory and one or more CPUs.

A KVM virtual CPU represents the state of one emulated CPU, including processor registers and other execution state

- Each virtual CPU has an associated `struct kvm_run` data structure, used to communicate information about the CPU between the kernel and user space.

set memory: bind the host memory to guest

- /dev/kvm layer, the one used to control the whole KVM subsystem and to create new VMs,
- VM layer, the one used to control an individual virtual machine,
- VCPU layer, the one used to control operation of a single virtual CPU (one VM can run on a multiple VCPUs)

https://zserge.com/posts/kvm/

linux-4.2.8/include/uapi/linux/kvm.h

linux-4.2.8/Documentation/s390/kvm.txt

/home/zxy/summerCode/DragonOS/kernel/src/filesystem/devfs/mod.rs

source drivers/vhost/Kconfig

source drivers/lguest/Kconfig

https://www.notion.so/July-c89649dd7cb6449f86c02b1f6f93e2b5?pvs=4#1f0c7891fff2427b98a4b4fb707b651d

linux-4.2.8/drivers/char/misc.c

```c
// linux-4.2.8/virt/kvm/kvm_main.c
static struct file_operations kvm_chardev_ops = {
	.unlocked_ioctl = kvm_dev_ioctl,
	.compat_ioctl   = kvm_dev_ioctl,
	.llseek		= noop_llseek,
};
kvm_chardev_ops.owner = module;
kvm_vm_fops.owner = module;
kvm_vcpu_fops.owner = module;
r = misc_register(&kvm_dev);
	if (r) {
		pr_err("kvm: misc device register failed\n");
		goto out_unreg;
	}
```

```
dragonOS
DragonOS/kernel/src/filesystem/devfs/mod.rs
```

Arch/x86/kvm/vmx.c

linux-4.2.8/arch/x86/kvm/vmx.c

```c
static struct kvm_x86_ops vmx_x86_ops = {
	.cpu_has_kvm_support = cpu_has_kvm_support,
	.disabled_by_bios = vmx_disabled_by_bios,
	.hardware_setup = hardware_setup,
	...
	
	static int __init vmx_init(void)
{
//第二个参数表示VMX实现的VCPU结构体的大小
	int r = kvm_init(&vmx_x86_ops, sizeof(struct vcpu_vmx),
                     __alignof__(struct vcpu_vmx), THIS_MODULE);
```

❯ pwd
/lib/modules/5.19.0-45-generic/kernel/arch/x86/kvm
❯ ls
kvm-amd.ko  kvm-intel.ko  kvm.ko

https://www.linux-kvm.org/page/Small_look_inside

kvm_bindings API documentation

https://docs.rs/kvm-bindings/latest/kvm_bindings/index.html

https://docs.rs/kvm-ioctls/latest/kvm_ioctls/

kvm create VM principle & overflow

x86 cpu long mode

kvm basic structures

https://www.cnblogs.com/dream397/p/14294065.html

https://web.stanford.edu/class/cs107/guide/x86-64.html

https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt

https://www.cnblogs.com/popsuper1982/p/3815398.html

https://www.twblogs.net/a/5b7d57e92b71770a43deb6c5

https://www.youtube.com/watch?v=6n-ANRWqll8

https://www.cnblogs.com/sammyliu/p/4543597.html

https://www.youtube.com/watch?v=mAZNlyXVoT4

https://developers.redhat.com/blog/2015/03/24/live-migrating-qemu-kvm-virtual-machines#vmstate_example__before

https://www.cnblogs.com/popsuper1982/p/3815398.html

https://www.twblogs.net/a/5b7d57e92b71770a43deb6c5

