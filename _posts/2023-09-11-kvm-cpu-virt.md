---
title:      "Linux KVM x86 CPU虚拟化原理及源代码分析"
date:       2023-09-11 09:00:00
author:     zxy
math: true
categories: ["Coding", "System", "Virtualization"]
tags: ["Virtualization"]
post: true
---

## 硬件辅助的CPU虚拟化

虚拟化技术允许在一个物理计算机上创建多个虚拟环境，每个虚拟环境都可以独立运行操作系统和应用程序，就像它们在独立的物理计算机上运行一样。硬件辅助的虚拟化技术，顾名思义，就是在CPU、芯片组及I/O等硬件中加入专门针对虚拟化的 支持，使得系统软件更容易、高效地实现虚拟化功能。本篇blog主要介绍x86 VT-x技术对CPU虚拟化的支持。

VT-x中的CPU虚拟化主要可以分为三个方面：

- 引入两种操作模式，统称为VMX操作模式，该模式与Ring0-Ring3的特权级正交

  - 根操作模式（VMX Root Operation）: hypervisor运行所处的模式
  - 非根操作模式（VMX Non-Root Operation）：guest/VM运行所处的模式

  用QEMU/KVM的例子来说，root和non-root模式以及Ring0-Ring3可以总结为下图：

  QEMU是运行在根模式下的用户态，为用户提供虚拟化接口。QEMU通过调用KVM提供的API，即`ioctl`进入到运行在根模式、内核态下的KVM。KVM通过执行虚拟化相关的特殊指令，将CPU切换到非根模式，运行虚拟机。

  ![kvm_model](/assets/img/in-post/2023-09-11-cpu-virt/Kvm_model.png)

- 引入了VMCS（Virtual-Machine Control Structure），其实就是一块不超过4KB的内存块，用于保存虚拟CPU需要的相关状态。比如CPU在根模式和非根模式下的特权寄存器值，相当于根模式和非根模式的上下文。每个VMCS对应一个虚拟CPU。具体来说，VMCS包括以下五大类别：

  - Guest State Area

    这个状态域的主要作用是为了保存客户机运行时的CPU状态。当VM-exit发生时，CPU把当前状态存入客户机状态域；VM-entry发生时，CPU从客户机状态域恢复状态。

    比如，最简单的情况下，在进入虚拟机前需要设置Guest State Area里的 `RIP`的值为虚拟机代码执行开始的位置，这样在VM-entry时，硬件会自动把真实CPU中的RIP的值设置为VMCS中Guest State Area里的RIP值。

    ![截屏2023-09-11 下午7.43.53](/assets/img/in-post/2023-09-11-cpu-virt/guest_state_area.png)

  - Host State Area

    这个状态域的主要作用是为了CPU在根模式下运行时的CPU状态。因为Host State area的内容通常几乎不变，这个状态域通常只在VM-exit时被恢复，VM-entry时不用保存。

    比如，简单来说，Host State Area里的 `RIP`的值应该设置为VM-exit时VMM的入口地址，也就是虚拟机退出时的异常处理代码的地址。

    ![截屏2023-09-11 下午7.51.34](/assets/img/in-post/2023-09-11-cpu-virt/host_state_area.png)

  - Control Area

    这个区域用于配制一些虚拟机的执行模式。

    通过配置该区域，可以指明在虚拟机中执行哪些指令，会引起VM-exit。比如`HLT exiting`字段就是控制虚拟机执行HLT指令时是否会发生VM-exit。

    通过对`Enable EPT`位的设置，可以在虚拟化时开启EPT页表内存虚拟化功能。

    ![截屏2023-09-11 下午8.00.16](/assets/img/in-post/2023-09-11-cpu-virt/control_area.png)

  - Vm-exit Control Fields

    这个区域规定了VM-exit发生时CPU的行为，比如`Host address space size`用来指定VM-exit后CPU是否处于64位的模式，64位的VMM通常需要打开这一位。

    ![截屏2023-09-11 下午8.12.36](/assets/img/in-post/2023-09-11-cpu-virt/vm-exit-control.png)

  - Vm-exit Information Fields

    这个区域用来获取VM-exit时的相关信息，比如是什么原因引起的VM-exit，是执行了`HLT`指令还是`EPT violation`？通过访问这个字段的信息，我们可以找到VM-exit的原因，并分配不同的处理函数处理这个VM-exit

    ![截屏2023-09-11 下午8.20.37](/assets/img/in-post/2023-09-11-cpu-virt/vm-exit-info.png)

- 引入了一组新指令，如`vmxon`, `vmxoff`打开/关闭VMX 操作模式；`vmlaunch`, `vmresume`发起VM-entry；`vmread`, `vmwrite`用于配制VMCS结构等等。

​		在这里，我们通过一张图来表示，这些指令和操作模式之间的关系。

​		![截屏2023-09-11 下午8.37.17](/assets/img/in-post/2023-09-11-cpu-virt/root-non-root.png)

## QEMU/KVM CPU虚拟化工作流程

OenHan将KVM的整个工作流程概括为下图：

1.  由虚拟化管理软件Qemu开始启动⼀个虚拟机

2. 通过ioctl等系统调⽤向内核中申请指定的资源，搭建好虚拟环境，启动虚拟机内的OS，执⾏ `vmlaunch` 指
   令，即进⼊了guest代码执⾏过程。

   对于`ioctl`，KVM通过3类不同的文件句柄实现，`kvm-fd`,`vm-fd`和`vcpu-fd`。

3. 如果 Guest OS 在⾮根模式下敏感指令引起的 trap，暂停 Guest OS 的执⾏，退出QEMU，即guest VMexit，进⾏⼀些必要的处理，然后重新进⼊客户模式，执⾏guest代码；这个时候如果是io请求，则提交给⽤户态下的qemu处理， qemu模拟处理后再次通过`ioctl`反馈给KVM驱动。

![截屏2023-09-11 下午7.43.53](/assets/img/in-post/2023-09-11-cpu-virt/kvm_process-1.png)



## Linux KVM CPU虚拟化源码分析

### 1. Overview

我们首先用思维导图的方式总览Linux KVM kernel module对CPU虚拟化的实现。对于调度和中断的处理我们暂时先不考虑，可以看到KVM实现CPU虚拟化主要分为三步：创建VM，创建VCPU以及VCPU运行。

![overview](/assets/img/in-post/2023-09-11-cpu-virt/overview.png)

### 2. 主要数据结构

KVM模块中使用结构体`struct kvm`来表示虚拟机，每一个虚拟机实例用一个`struct kvm`结构表示。KVM结构体包含了vCPU、内存、APIC、IRQ、MMU、Event事件管理等信息。该结构体中的信息主要在KVM虚拟机内部使用，用于跟踪虚拟机的状态。

```c
struct kvm {
	spinlock_t mmu_lock;
	struct mutex slots_lock;
	struct mm_struct *mm; /* userspace tied to this vm */
	struct kvm_memslots *memslots[KVM_ADDRESS_SPACE_NUM]; /* KVM虚拟机分配的内存slot，用于记录GPA-HVA的映射，内存虚拟化使用 */
	struct srcu_struct srcu;
	struct srcu_struct irq_srcu;
	struct kvm_vcpu *vcpus[KVM_MAX_VCPUS]; /* KVM虚拟机中包含的vCPU结构体，一个虚拟机CPU对应一个vCPU结构体 */
	atomic_t online_vcpus;
	int last_boosted_vcpu;
	struct list_head vm_list; /* HOST上VM管理链表，multiple "KVM structs"的双向链表 */
	struct mutex lock;
	struct kvm_io_bus *buses[KVM_NR_BUSES]; /* KVM虚拟机中的I/O总线，一条总线对应一个kvm_io_bus结构体，如ISA总线、PCI总线 */
	struct kvm_vm_stat stat; /* KVM虚拟机中的页表、MMU等运行时的状态信息 */
	struct kvm_arch arch; /* 这个是host的arch的一些参数 */
	atomic_t users_count;
	struct mutex irq_lock;
	long tlbs_dirty;
	struct list_head devices;
  ...
};
```

每一个vCPU都对应一个kvm_vcpu结构体，在用户通过KVM_CREATE_VCPU系统调用请求创建vCPU时创建。

```c
struct kvm_vcpu {
	struct kvm *kvm; /* 这个CPU所属的KVM */
	int cpu;
	int vcpu_id; /* 对应的CPU的ID，由用户进程指定*/
	int srcu_idx;
	int mode;
	unsigned long requests;
	unsigned long guest_debug;
	struct mutex mutex;
	struct kvm_run *run;  /* CPU运行时的状态，其中保存了内存信息、虚拟机状态等各种动态信息，如VM-Exit发生的原因等 */
	int fpu_active;
	int guest_fpu_loaded, guest_xcr0_loaded;
	unsigned char fpu_counter;
	wait_queue_head_t wq;
	struct pid *pid;
	int sigset_active;
	sigset_t sigset;
	struct kvm_vcpu_stat stat;
	bool preempted;
	struct kvm_vcpu_arch arch; // 当前VCPU虚拟的架构，存储有KVM虚拟机的运行时参数，如定时器、中断、内存槽等方面的信息
	...
};
```

### 3. 源码分析

有了总体的概念和对主要数据结构的掌握，我们就可以逐一来分析Linux KVM CPU虚拟化的实现了。

#### 3.1 创建VM

创建虚拟机的主体函数是`kvm_create_vm`(linux/virt/kvm/kvm_main.c)，用户态Qemu-kvm通过ioctl `KVM_CREATE_VM`，经过层层调用，最终进入此函数中，由该函数完成虚拟机的创建。

```c
ioctl(KVM_CREATE_VM..)
  -->kvm_dev_ioctl  // /dev/kvm设备的ioctl接口的处理函数，用于分发不同的ioctl
      --> kvm_dev_ioctl_create_vm  // 调用kvm_create_vm，并生成kvm-vm的控制文件，用于vm的ioctl
        -->kvm_create_vm
```

`kvm_dev_ioctl_create_vm`的主要流程如下：

```c
// virt/kvm/kvm_main.c: (有所省略)
static int kvm_dev_ioctl_create_vm(void)
{
	int fd;
	struct kvm *kvm;

  kvm = kvm_create_vm(type); 	/*创建VM*/
  if (IS_ERR(kvm))
         return PTR_ERR(kvm);
  r = kvm_coalesced_mmio_init(kvm);
  r = get_unused_fd_flags(O_CLOEXEC);
  /*生成kvm-vm控制文件，这个文件的作用是提供对vm的io_ctl控制*/
  file = anon_inode_getfile("kvm-vm", &kvm_vm_fops, kvm, O_RDWR);
  ...
  fd_install(fd, file);
	return fd;
}
```

`kvm_create_vm`主要完成KVM虚拟机结构体的创建、KVM的MMU操作接口的安装、KVM的IO总线、事件通道的初始化等操作，主要实现了以下功能：

1. `kvm_arch_alloc_vm()`申请并初始化kvm结构体
2. `hardware_enable_all()`，针对每一个CPU，调用kvm_x86_ops中硬件相关的函数进行硬件使能，主要设置相关寄存器和标记，使CPU进入虚拟化相关模式中(如Intel VMX)。
3. 初始化memslots结构体信息
4. 初始化BUS总线结构体信息
5. 初始化事件通知信息和内存管理相关结构体信息
6. 将新创建的虚拟机加入KVM的虚拟机列表

去掉不重要过程以及错误处理路径等，`kvm_create_vm`的主要源码如下：

```c
// virt/kvm/kvm_main.c
static struct kvm *kvm_create_vm(unsigned long type)
{
	int r, i;
	struct kvm *kvm = kvm_arch_alloc_vm(); // 申请kvm结构体内存
  ...
  // 调用架构相关的初始化函数，初始化虚拟机
  // 这部分主要是初始化KVM中类型为kvm_arch的arch成员，用于存放与架构相关的数据
	r = kvm_arch_init_vm(kvm, type); 
	...
  // 启用硬件虚拟化支持，主要设置相关寄存器和标记，使CPU进入虚拟化相关模式中
	r = hardware_enable_all(); 
	...
	r = -ENOMEM;
  // 初始化memslots结构体信息
	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
		kvm->memslots[i] = kvm_alloc_memslots();
		if (!kvm->memslots[i])
			goto out_err_no_srcu;
	}

	...
  // 初始化BUS总线结构体信息
	for (i = 0; i < KVM_NR_BUSES; i++) {
		kvm->buses[i] = kzalloc(sizeof(struct kvm_io_bus),
					GFP_KERNEL);
		if (!kvm->buses[i])
			goto out_err;
	}
	
  // 由于虚拟机的内存其实也就是QEMU进程的虚拟内存，因此这里需要引用到当前QEMU进程的mm_struct
  // 并且初始化mmu_lock成员来表示操作虚拟机MMU数据的锁
	spin_lock_init(&kvm->mmu_lock);
	kvm->mm = current->mm;
	atomic_inc(&kvm->mm->mm_count);
  
  // 初始化事件通知信息
	kvm_eventfd_init(kvm);
	mutex_init(&kvm->lock);
	mutex_init(&kvm->irq_lock);
	mutex_init(&kvm->slots_lock);
	atomic_set(&kvm->users_count, 1);
	INIT_LIST_HEAD(&kvm->devices);
	
  // 初始化内存管理单元（MMU）通知器，用于跟踪内存的变化
	r = kvm_init_mmu_notifier(kvm);
	...
	
  // 将创建的虚拟机添加到虚拟机列表vm_list中
	spin_lock(&kvm_lock);
	list_add(&kvm->vm_list, &vm_list);
	spin_unlock(&kvm_lock);

	preempt_notifier_inc();
	return kvm;

out_err:
...
	return ERR_PTR(r);
}

```

#### 3.2 创建vCPU

创建vCPU的主体函数是`kvm_vm_ioctl_create_vcpu`(linux/virt/kvm/kvm_main.c)，用户态Qemu通过ioctl `KVM_CREATE_VCPU`，经过调用，进入此函数中，由该函数完成vCPU的创建。

```c
ioctl(KVM_CREATE_VCPU..)
  -->kvm_vm_ioctl  // kvm-vm匿名设备的ioctl接口的处理函数，用于分发不同的ioctl
      --> kvm_vm_ioctl_create_vcpu  // 进行一些错误判断、vcpu_fd创建，调用创建/初始化vCPU的函数
  			--> kvm_arch_vcpu_create & kvm_arch_vcpu_setup // 创建vCPU的主体函数
```

`kvm_vm_ioctl_create_vcpu`主要完成kvm_vcpu结构体的创建，包括初始化VMCS等硬件虚拟化结构。

```c
// virt/kvm/kvm_main.c
static int kvm_vm_ioctl_create_vcpu(struct kvm *kvm, u32 id)
{
	int r;
	struct kvm_vcpu *vcpu, *v;
	/* 判断是否达到最大cpu个数 */
	if (id >= KVM_MAX_VCPUS)
		return -EINVAL;
	/* 调用相关cpu的vcpu_create 通过arch/x86/x86.c 进入vmx.c */
  /* 创建kvm_vcpu结构体，具体实现跟架构相关，直接调用kvm_x86_ops中的create_cpu方法执行，主要完成相关寄存器和CPUID的初始化 */
	vcpu = kvm_arch_vcpu_create(kvm, id);
	if (IS_ERR(vcpu))
		return PTR_ERR(vcpu);

	preempt_notifier_init(&vcpu->preempt_notifier, &kvm_preempt_ops);
	/* 调用相关cpu的vcpu_setup，初始化kvm_vcpu结构体 */
	r = kvm_arch_vcpu_setup(vcpu);
	if (r)
		goto vcpu_destroy;
  

	mutex_lock(&kvm->lock);
	if (!kvm_vcpu_compatible(vcpu)) {
		r = -EINVAL;
		goto unlock_vcpu_destroy;
	}
  /* 判断是否达到最大cpu个数，如果是，则销毁刚创建的实例 */
	if (atomic_read(&kvm->online_vcpus) == KVM_MAX_VCPUS) {
		r = -EINVAL;
		goto unlock_vcpu_destroy;
	}
	/* 判断当前VCPU是否已经加入了某个KVM主机，如果是，则销毁刚创建的实例 */
	kvm_for_each_vcpu(r, v, kvm)
		if (v->vcpu_id == id) {
			r = -EEXIST;
			goto unlock_vcpu_destroy;
		}

	BUG_ON(kvm->vcpus[atomic_read(&kvm->online_vcpus)]);

	/* Now it's all set up, let userspace reach it */
  /* 生成kvm-vcpu控制文件，创建vcpu_fd */
	kvm_get_kvm(kvm);
	r = create_vcpu_fd(vcpu);
	if (r < 0) {
		kvm_put_kvm(kvm);
		goto unlock_vcpu_destroy;
	}
	// 将创建的kvm_vcpu结构体加入kvm的VCPU数组中
	kvm->vcpus[atomic_read(&kvm->online_vcpus)] = vcpu;
	smp_wmb();
	atomic_inc(&kvm->online_vcpus); // 增加online vcpu数量

	mutex_unlock(&kvm->lock);
	kvm_arch_vcpu_postcreate(vcpu);
	return r;
	...
}
```

其中两个主要函数是`kvm_arch_vcpu_create`以及`kvm_arch_vcpu_setup`

- 在vmx的实现中，`kvm_arch_vcpu_create`会调用`vmx_create_vcpu`（arch/x86/kvm/vmx.c）函数来创建vCPU。`vmx_create_vcpu`会申请VMCS的内存空间，设置VMCS中的状态域、控制域等。为启动虚拟化做好准备。
- 在vmx的实现中，`kvm_arch_vcpu_setup`会调用`vcpu_load`来使用当前CPU以及`kvm_mmu_setup`来设置内存虚拟化相关配置。

#### 3.3 运行VCPU

在VM和VCPU创建好并完成初始化后，就可以调度该VCPU运行了。VCPU(虚拟机)的运行主要任务是要进行上下文切换，主要就是相关寄存器、APIC状态、TLB等。

```c
ioctl(KVM_RUN..)
  -->kvm_vcpu_ioctl  // kvm-vm匿名设备的ioctl接口的处理函数，用于分发不同的ioctl
      --> kvm_arch_vcpu_ioctl_run  
  			--> vcpu_run // 运行vCPU的主体函数
  				--> vcpu_enter_guest
  					--> kvm_x86_ops->run / vmx_vcpu_run
```

`vcpu_run`的源码如下，分析在注释中，vcpu的主要流程：

```c
static int vcpu_run(struct kvm_vcpu *vcpu)
{
	int r;
	struct kvm *kvm = vcpu->kvm;

	vcpu->srcu_idx = srcu_read_lock(&kvm->srcu);

	for (;;) {
    /* 通过vcpu_enter_guest进入guest模式 */ 
		if (kvm_vcpu_running(vcpu))
			r = vcpu_enter_guest(vcpu);
		else
			r = vcpu_block(kvm, vcpu);
		if (r <= 0)
			break;
    // 以下是vmexit后的一些处理函数
		/* 检查是否有阻塞的时钟timer */
		clear_bit(KVM_REQ_PENDING_TIMER, &vcpu->requests);
		if (kvm_cpu_has_pending_timer(vcpu))
			kvm_inject_pending_timer_irqs(vcpu);
		/* 检查是否有用户空间的中断注入 */ 
		if (dm_request_for_irq_injection(vcpu)) {
			r = -EINTR;
			vcpu->run->exit_reason = KVM_EXIT_INTR;
			++vcpu->stat.request_irq_exits;
			break;
		}

		kvm_check_async_pf_completion(vcpu);
		/* 是否有阻塞的signal */
		if (signal_pending(current)) {
			r = -EINTR;
			vcpu->run->exit_reason = KVM_EXIT_INTR;
			++vcpu->stat.signal_exits;
			break;
		}
    /* 执行调度 */
		if (need_resched()) {
			srcu_read_unlock(&kvm->srcu, vcpu->srcu_idx);
			cond_resched();
			vcpu->srcu_idx = srcu_read_lock(&kvm->srcu);
		}
	}

	srcu_read_unlock(&kvm->srcu, vcpu->srcu_idx);
	return r;
}
```

- `vcpu_enter_guest`最终调用kvm_x86_ops中的run函数运行。对应于Intel平台，该函数为`vmx_vcpu_run`(设置Guest CR3和其他寄存器、EPT/影子页表相关设置、汇编代码VMLAUNCH切换到非根模式，执行Guest目标代码)。

- Guest代码执行到敏感指令或因其他原因(比如中断/异常)，VM-Exit退出非根模式，返回到`vcpu_enter_guest`函数继续执行。

## Reference

1. [KVM内核模块重要的数据结构](http://liujunming.top/2017/06/27/KVM%E5%86%85%E6%A0%B8%E6%A8%A1%E5%9D%97%E9%87%8D%E8%A6%81%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)

2. [KVM基本原理及架构八-KVM内核模块重要流程分析](http://blog.chinaunix.net/uid-26985337-id-5740797.html)
3. [QEMU 之 CPU 虚拟化（二）：KVM 模块初始化介绍](https://xie.infoq.cn/article/6fa43a9b435048e005845e422)
4. [QEMU之CPU虚拟化（三）：虚拟机的创建](https://juejin.cn/post/7274140856033591348)

5. [KVM源代码分析3:CPU虚拟化](https://oenhan.com/kvm-src-3-cpu)
6. [KVM(Kernel-based Virtual Machine)源码分析](https://www.owalle.com/2019/02/20/kvm-src-analysis/)

7. [KVM思维导图](https://www.owalle.com/2019/02/20/kvm-src-analysis/kvm-src-analysis-mind.svg)
8. 《系统虚拟化：原理与实现》
