---
title:      "KVM主要结构分析--使用Rust实现KVM userspace"
date:       2023-07-03 09:00:00
author:     zxy
math: true
categories: ["Coding", "System", "Virtualization"]
tags: ["vm", "kvm", "Virtualization"]
post: false
---

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



kvm_bindings API documentation

https://docs.rs/kvm-bindings/latest/kvm_bindings/index.html

https://docs.rs/kvm-ioctls/latest/kvm_ioctls/

kvm create VM principle & overflow

x86 cpu long mode

kvm basic structures

https://www.cnblogs.com/dream397/p/14294065.html

https://web.stanford.edu/class/cs107/guide/x86-64.html

https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt
