---
layout: post
title:      "Linux KVM x86内存虚拟化EPT源代码分析"
date:       2023-09-12 09:00:00
author:     zxy
math: true
categories: ["Coding", "System", "Virtualization"]
tags: ["Virtualization"]
post: true
---
> 阅读本篇博客前请先阅读
>
> 1. [x86内存虚拟化--影子页表(Shadow Page Table)和拓展页表(EPT)](https://zxxyy.github.io/posts/SPT-EPT/)
> 2. [Linux KVM x86 CPU虚拟化原理及源代码分析](https://zxxyy.github.io/posts/kvm-cpu-virt/)

## 重要数据结构

```c
/* for KVM_SET_USER_MEMORY_REGION */
struct kvm_userspace_memory_region {
	__u32 slot; // 要在哪个slot上注册内存区间
  // flags有两个取值，KVM_MEM_LOG_DIRTY_PAGES和KVM_MEM_READONLY
  // KVM_MEM_LOG_DIRTY_PAGES用来开启内存脏页
  // KVM_MEM_READONLY用来开启内存只读。
	__u32 flags; 
	__u64 guest_phys_addr; // 虚机内存区间起始物理地址
	__u64 memory_size; /* bytes */ // 虚机内存区间大小
  // 虚机内存区间对应的主机虚拟地址
	__u64 userspace_addr; /* start of the userspace allocated memory */
};
```

```c
struct kvm_memslots {
	int nmemslots;
	struct kvm_memory_slot memslots[KVM_MEMORY_SLOTS + KVM_PRIVATE_MEM_SLOTS];
};
```

```c
struct kvm_memory_slot {
	gfn_t base_gfn;       // 该块物理内存块所在guest 物理页帧号
	unsigned long npages; // 该块物理内存块占用的page数
	unsigned long flags;
	unsigned long *rmap;  // 分配该块物理内存对应的host内核虚拟地址（vmalloc分配）
	unsigned long *dirty_bitmap;
	struct {
		unsigned long rmap_pde;
		int write_count;
	} *lpage_info[KVM_NR_PAGE_SIZES - 1];
	unsigned long userspace_addr; // 用户空间地址（QEMU)
	int user_alloc;
};
```

```c
struct kvm_mmu {
	void (*set_cr3)(struct kvm_vcpu *vcpu, unsigned long root);
	unsigned long (*get_cr3)(struct kvm_vcpu *vcpu);
	u64 (*get_pdptr)(struct kvm_vcpu *vcpu, int index);
	int (*page_fault)(struct kvm_vcpu *vcpu, gva_t gva, u32 err,
			  bool prefault);
	void (*inject_page_fault)(struct kvm_vcpu *vcpu,
				  struct x86_exception *fault);
	gpa_t (*gva_to_gpa)(struct kvm_vcpu *vcpu, gva_t gva, u32 access,
			    struct x86_exception *exception);
	gpa_t (*translate_gpa)(struct kvm_vcpu *vcpu, gpa_t gpa, u32 access,
			       struct x86_exception *exception);
	int (*sync_page)(struct kvm_vcpu *vcpu,
			 struct kvm_mmu_page *sp);
	void (*invlpg)(struct kvm_vcpu *vcpu, gva_t gva);
	void (*update_pte)(struct kvm_vcpu *vcpu, struct kvm_mmu_page *sp,
			   u64 *spte, const void *pte);
	hpa_t root_hpa;  // 页表根地址
	int root_level;
	int shadow_root_level;
	union kvm_mmu_page_role base_role;
	bool direct_map;
  ...
}
```

以上数据结构的关系总结为下图：

![structure](/assets/img/in-post/2023-09-12-mem-virt/struct.jpeg)

## EPT内存虚拟化代码流程与分析

![截屏2023-09-12 下午3.15.04](/assets/img/in-post/2023-09-12-mem-virt/overview.png)

我们可以把硬件辅助的内存虚拟化大致分为四个部分：

1. 建立HVA-GPA的映射关系
2. MMU初始化
3. EPT根页表初始化
4. EPT violation处理

### 建立GPA-HVA的映射关系

这个环节中涉及到的数据结构主要是`kvm_userspace_memory_region`和`kvm_memory_slot`。HVA-GPA的具体映射信息就保存在`kvm_memory_slot`这个结构内。

```c
ioctl(KVM_SET_USER_MEMORY_REGION..)
  --> kvm_vm_ioctl // kvm-vm匿名设备的ioctl接口的处理函数，分发不同的ioctl
    -->kvm_vm_ioctl_set_memory_region  
        --> kvm_set_memory_region 
          -->__kvm_set_memory_region   // 根据用户态提供的信息创建slot
            --> kvm_arch_create_memslot
```

### MMU初始化

这个环节主要是设置MMU相关的处理函数，设定内存虚拟化方式（比如是EPT还是SPT），在创建vCPU的时候进行。

```c
ioctl(KVM_CREATE_CPU..)
  --> kvm_vm_ioctl // kvm-vm匿名设备的ioctl接口的处理函数，分发不同的ioctl
    -->kvm_vm_ioctl_create_vcpu  
        --> kvm_arch_vcpu_setup 
          -->kvm_mmu_setup  
            --> init_kvm_mmu
  					// 初始化kvm_mmu结构体
            // 在这个函数中设定与页表操作相关的各种处理函数，比如发生EPT violation时该调用哪个函数等
  						--> init_kvm_tdp_mmu 
```

### EPT根页表初始化

对于硬件辅助的内存虚拟化来说，需要设置一个EPT根页表，并将其地址值写入VMCS结构体中，在vCPU运行的时候进行。这样，在进入guest模式时，硬件会自动把EPT根页表地址写入EPTP寄存器中，用于EPT页表映射。

```c
ioctl(KVM_RUN..)
  --> kvm_vcpu_ioctl // kvm-vm匿名设备的ioctl接口的处理函数，分发不同的ioctl
    -->kvm_arch_vcpu_ioctl_run  
        --> vcpu_run 
          -->kvm_mmu_reload  
            --> kvm_mmu_load
  						--> mmu_alloc_roots // 申请EPT根页表内存
  							--> vcpu->arch.mmu.set_cr3 (vmx_set_cr3) // 在VMCS中设置根页表地址eptp
```

### EPT violation处理

如果在EPT页表中GPA->HPA的映射不存在，将会触发VM-Exit，KVM负责捕捉该异常，并交由KVM的缺页中断机制进行相应的缺页处理。

```c
vmx_handle_exit
  --> handle_ept_violation
  	--> kvm_mmu_page_fault
  		--> tdp_page_fault // EPT的缺页处理函数，在MMU初始化的时候设置好的
  			|--> gfn_to_pfn  // 结合memslot中的信息，以及host内存信息，负责GPA-HPA的转换
  			|--> __direct_map // 建立EPT页表结构
```

## 具体函数分析

To Be Continued...

## Reference

1. [QEMU注册内存到KVM流程](https://blog.csdn.net/huang987246510/article/details/105744738)
2. [KVM源代码分析4:内存虚拟化](https://oenhan.com/kvm-src-4-mem)
3. [KVM之内存虚拟化(KVM MMU Virtualization)](https://royhunter.github.io/2016/03/13/kvm-mmu-virtualization/)
4. [KVM MMU Note](https://gist.github.com/tan-yue/93acf33f46e1f223b68d0235b563fa79)
5. [KVM(Kernel-based Virtual Machine)源码分析](https://www.owalle.com/2019/02/20/kvm-src-analysis/)