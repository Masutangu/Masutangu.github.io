---
layout: post
date: 2017-01-25T10:21:16+08:00
title: Linux 内核系列－内存管理
category: 读书笔记
---

本系列文章为阅读《现代操作系统》和《Linux 内核设计与实现》所整理的读书笔记，源代码取自 Linux-kernel 2.6.34 版本并有做简化。

# 概念

操作系统存储管理方案的演进：

## 无存储抽象

早期计算机并没有存储抽象，程序直接访问物理内存地址。使用这种模型，想要同时运行多个程序非常困难。

## 存储抽象：地址空间

### 地址空间的概念

要保证多个应用程序同时处于内存并且互不影响，则需要解决两个问题：**保护**和**重定位**。内存块加上保护键并通过装载时重定位程序虽然可以做到，但是是个缓慢和复杂的解决方案。一个更好的方法是创造一个新的内存抽象：地址空间。地址空间是一个进程可用于寻址内存的一套地址集合。进程的地址空间独立于其他进程的地址空间。

### 交换技术

有两种处理内存超载的通用方法，最简单的策略是交接（swapping）技术，即把一个进程完整调入内存，运行一段时间后，存回磁盘。另一种策略是虚拟内存。

### 空闲内存管理

在动态分配内存时，操作系统通常有两种方式跟踪内存使用情况：**位图**和**空闲链表**。

## 虚拟内存

虚拟内存的基本思想是：每个程序拥有自己的地址空间，这个空间被分割为许多块，每一块被称为页面（page）。这些页被映射到物理内存，但不是所有页都在内存才能运行程序。

### 分页

大部分虚拟内存系统都使用了**分页**技术。程序产生的地址称为虚拟地址，访问时不是直接由内存总线处理，而是通过内存管理单元（MMU)，MMU 把虚拟地址映射到物理内存。

虚拟地址空间按照固定大小划分页面，物理内存对应的单元称为页框（page frame）。页面和页框的大小通常是一样的。RAM 和磁盘之间的交换总是以整个页面为单元进行的。

当进程访问了一个未映射的页面，MMU 注意到该页面没有被映射，于是使 CPU 陷入到操作系统（缺页中断），操作系统找到一个很少使用的页框，将其内容写入磁盘，并把需要访问的页面的内容读到该页框中，修改映射关系后，重新启动引起陷进的指令。

### 页表

页表的简单实现：虚拟地址被分成**虚拟页号**（高位部分）和**偏移量**（低位部分）。虚拟页号可用作页表的索引，以找到相应的页表项。由页表项可以找到页框号。页框号加上偏移量就是实际的内存物理地址。

页表项的结构同城包含页框号、“在／不在”位、“保护”位（读写执行权限）、“修改”位、“访问”位和“高速缓存禁止”位。

若页面不在内存时，该页面对应的磁盘地址不是页表的一部分。这部分信息保存在操作系统内部的软件表格中，硬件不需要。

### 加速分页过程

大多数程序总是对少量页面进行多次访问，因此优化方案是为计算机配置一个小型硬件设备 TLB，将虚拟地址直接映射成物理地址，而不必访问页表。

### 针对大内存的页表

64位机器上，多级页表不是个好主意。解决方案之一是使用倒排页表。在这种设计中，每一个页框有一个表项，而不是每个页面有一个表项。倒排页表的不足在于从虚拟地址到物理地址的转换变得很困难。可以通过 TLB 和散列表来提升效率。

## 分页系统中的设计问题

### 分离的指令空间和数据空间

大多数计算机只有一个地址空间，即存放程序也存放数据。如果地址空间太小的话，PDP-11的解决方案是为指令和数据设置分离的地址空间，分别称为 I 空间和 D 空间。此时链接器必须将数据重定位到虚拟地址 0，而不是指令段后。

### 共享库

传统的链接，会将被调用的外部库的函数加载到二进制文件，当程序载入内存开始执行时，它所需的所有函数都已经准备就绪。

为了节省磁盘和内存空间，引入了共享库。当程序和共享库链接时，链接器没有加载被调用的函数，而是加载了一小段能够在运行时绑定所调用函数的存根例程（stbu routine）。共享库或和程序一起加载，或在第一次被调用时加载。当其他程序已经加载过，就没有必要再次加载了。共享库不是一次性读入内存，而是以页面为单位装载的。

### 内存映射文件 

共享库其实是一种更通用的机制：内存映射文件的一个特例。其思想是：进程可以通过系统调用，将一个文件映射到其虚拟地址空间的一部分。在映射共享的页面时不会实际读入页面的内容，而是访问页面时才会被读入，磁盘文件当作后备存储，当进程退出或显式解除文件映射时，所有改动才会写回文件。

如果两个或以上的进程同时映射了一个文件，它们就可以通过共享内存来通信。

# Linux 中的实现

## 内存管理

### 页

内核把物理页作为内存管理的基本单位。尽管处理器最小可寻址单位通常为字，但内存管理单元（MMU)通常以页为单位进行处理。体系结构不同，支持的页大小也不同。大多数 32 位体系结构支持 4 KB 的页，而 64 位体系结构一般支持 8 KB 的页。

内核用 ```struct page``` 结构表示系统中的每个物理页，该结构位于 <linux/mm_types.h> 中：

```c 
struct page {
	unsigned long flags;	/* Atomic flags, some possibly updated asynchronously */
	atomic_t _count;		/* Usage count, see below. */
	
    atomic_t _mapcount;	    /* Count of ptes mapped in mms,
					         * to show when page is mapped
					         * & limit reverse map searches.
	    			         */
	};
	union {
	    struct {
		unsigned long private;	/* Mapping-private opaque data:
					 	         * usually used for buffer_heads
						         * if PagePrivate set; used for
						         * swp_entry_t if PageSwapCache;
						         * indicates order in the buddy
						         * system if PG_buddy is set.
						         */
		struct address_space *mapping;	/* If low bit clear, points to
						                 * inode address_space, or NULL.
						                 * If page mapped as anonymous
						                 * memory, low bit is set, and
						                 * it points to anon_vma object:
						                 * see PAGE_MAPPING_ANON below.
						                 */
	    };
	union {
		pgoff_t index;		/* Our offset within mapping. */
		void *freelist;		/* SLUB: freelist req. slab lock */
	};
	struct list_head lru;	/* Pageout list, eg. active_list
					         * protected by zone->lru_lock !
					         */
	void *virtual;			/* Kernel virtual address (NULL if not kmapped, ie. highmem) */
};
```
flage 域用来存放页的状态。这些状态包括页是不是脏的，是不是被锁定在内存中等等。_count 域存放页的引用计数，变成 0 说明当前内核没有引用这一页。页可以由页缓存使用（此时 mapping 域指向和这个页关联的 address\_space 对象)，或者作为私有数据（由 private 指向），或者作为进程页表中的映射。

virtual 域是页的虚拟地址，通常情况下，就是页在虚拟内存中的地址。高端内存并不会永久映射到内核地址空间上，这种情况下，virtual 域的值为 NULL。

page 结构与物理页相关，并非与虚拟页相关。

### 区

由于硬件的限制，内核把页划分为不同的区（zone）。内核使用区对具有相似特性的页进行分组。Linux 必须处理以下两种因硬件存在的缺陷而引起的内存寻址问题：

* 一些硬件只能用特定的内存地址来执行 DMA（直接内存访问）
* 一些体系结构其内存的物理寻址范围比虚拟寻址范围大得多，这样就有些内存不能永久地映射到内核空间上

因为存在这些制约条件，Linux 使用三种区：

* ZONE_DMA： 这个区包含的页能用来执行 DMA 操作
* ZONE_NORMAL： 这个区包含的都是能正常映射的页
* ZONE_HIGHMEM： 这个区包含“高端内存”，其中的页并不能永久地映射到内核地址空间。这些区定义于 <linux/mmzone.h>

Linux 把系统的页划分为区，形成不同的内存池。每个区都用 ```struct zone``` 表示，定义在 <linux/mmzone.h> 中。

### 获得页

内核提供一种请求内存的底层机制，并提供了对它进行访问的几个接口。所有这些接口都以页为单位分配内存，定义于 <linux/gfp.h>。最核心的函数是：

```c
struc page * alloc_pages(unsigned int gfp_mask, unsigned int order)
```

该函数分配 2^order 个连续的物理页，并返回一个指针，该指针指向第一个页的 page 结构体；如果出错就返回 NULL。

下面的函数：

```c
void *page_address(struct page *page)
``` 

把给定的物理页转换成逻辑地址。如果你不需要使用 struct page，可以直接调用

```c
unsigned long__get_free_pages(unsigned int gfp_mask, unsigned int order)
```

该函数与 ```alloc_page()``` 作用相同，不过它直接返回请求的第一个页的逻辑地址，因为页是连续的，其他页也会紧随其后。

## kmalloc()

```kmalloc()``` 函数与用户空间的 ```malloc()``` 一族非常类似，只不过它多了一个 flags 参数。```kmalloc()``` 函数是一个简单的接口，用它可以获得以字节为单位的一块内核内存。```kmalloc()``` 在 <linux/slab.h> 中声明：

```c
void *kmalloc(size_t size, int flags)
```

这个函数返回一个指向内存块的指针，其内存块至少要有 size 大小。所分配的内存区在物理上是连续的。在出错时返回 NULL。

### vmalloc()

```vmalloc()``` 函数的工作方式类似于 ```kmalloc()```，只不过前者分配的内存虚拟地址是连续的，而物理地址无需连续。这也是用户空间分配函数的工作方式：由 ```malloc()``` 返回的页在进程的虚拟地址空间内是连续的，但并不保证它们在物理 RAM 中也连续。```kmalloc()``` 函数确保页在物理地址上是连续的。

大多数情况下，只有硬件设备需要得到物理地址连续的内存，硬件设备存在于内存管理单元之外，它根本不理解什么是虚拟内存。因此硬件设备用到的任何内存区都必须是物理上连续的块，而不仅仅是虚拟地址连续的块。而仅供软件使用的内存块（例如进程相关的缓冲区）就可以使用只有虚拟地址连续的内存块。

尽管在某些情况下才需要物理上连续的内存块，但很多内核代码都用 ```kmalloc()``` 来获得内存，而不是 ```vmalloc()```。这主要是出于性能的考虑。```vmalloc()``` 为了把物理上不连续的页转换为虚拟空间上连续的页，必须专门建立页表项，通过 ```vmalloc()``` 获得的页必须一个个进行映射，这会导致比直接内存映射大得多的 TLB 抖动。因此 ```vmalloc()``` 仅在不得已的时候才使用，一般是在为了获得大块内存，例如当模块被动态插入到内核中时，就把模块装载到由 ```vmalloc()``` 分配的内存上。

### slab 层

Linux 内核提供了 slab 层（即 slab 分配器），slab 分配器扮演了通用数据结构缓存层的角色。

#### slab 层的设计

slab 层把不同的对象划分为所谓的**高速缓存组**，其中每个高速缓存都存放不同类型的对象，每种对象类型对应一个高速缓存。例如一个高速缓存用于存放进程描述符，另一个高速缓存存放索引节点对象（struct inode）。```kmalloc()``` 接口建立在 slab 层之上，使用了一组通用高速缓存。

这些高速缓存又被划分为 slab。slab 由一个或多个物理上连续的页组成。每个高速缓存由多个 slab 组成。每个 slab 都包含一些对象成员，即被缓存的数据结构。每个 slab 处于三种状态之一：满、部分满或空。当内核需要一个新对象时，先从部分满的 slab 中进行分配，如果没有部分满的 slab，就从空的 slab 进行分配。如果没有空的 slab，就要创建一个 slab。

## 进程地址空间

内核除了管理自身的内存外，还必须管理进程的地址空间。Linux 操作系统采用虚拟内存技术，对每个进程来说，它们好像都可以访问整个系统的所有物理内存，即使单独一个进程，它拥有的地址空间也可远大于系统物理内存。

每个进程都有一个 32 或 64 位的平坦地址空间，空间的具体大小取决于体系结构。每个进程有唯一的平坦地址空间，而且进程地址空间之间彼此互不相干。两个不同的进程可以在它们各自地址空间中国年相同的地址内存存放不同的数据，进程之间也可以选择共享地址空间（线程）。

进程地址空间中，可被访问的合法地址区间称为内存区域（memory area）。进程只能访问有效范围内的内存地址。每个内存区域也具有进程必须遵循的特定访问属性，如只读、只写、可执行等属性。如果进程访问了不在有效范围中的地址，或以不正确的方式访问有效地址，那么内核就会终止该进程，并返回“段错误”。

内存区域可以包含各种内存对象：

* 可执行文件代码的内存映射，称为代码段（text section）
* 可执行文件的已初始化全局变量的内存映射，称为数据段（data section）
* 包含未初始化全局变量，即 bss 段的零页的内存映射
* 用于进程用户空间栈的零页的内存映射
* 每个共享库的代码段、数据段和 bss 也会载入进程的地址空间
* 任何内存映射文件
* 任何共享内存段
* 任何匿名的内存映射，比如由 malloc() 分配的内存

### 内存描述符

内核使用内存描述符结构体表示进程的地址空间，该结构包含了和进程地址空间有关的全部心血。内存描述法由 mm_struct 结构体表示，定义在文件 <linux/sched.h> 中。

```c
struct mm_struct {
	struct vm_area_struct * mmap;		/* list of VMAs */
	struct rb_root mm_rb;
	struct vm_area_struct * mmap_cache;	/* last find_vma result */

	unsigned long free_area_cache;		/* first hole of size cached_hole_size or larger */
	pgd_t * pgd;
	atomic_t mm_users;			/* How many users with user space? */
	atomic_t mm_count;			/* How many references to "struct mm_struct" (users count as 1) */
	int map_count;				/* number of VMAs */
	struct rw_semaphore mmap_sem;
	spinlock_t page_table_lock;		/* Protects page tables and some counters */

	struct list_head mmlist;		/* List of maybe swapped mm's.	These are globally strung
						 			 * together off init_mm.mmlist, and are protected
						 			 * by mmlist_lock
									 */
	unsigned long hiwater_rss;	/* High-watermark of RSS usage */
	unsigned long hiwater_vm;	/* High-water virtual memory usage */

	unsigned long total_vm, locked_vm, shared_vm, exec_vm;
	unsigned long stack_vm, reserved_vm, def_flags, nr_ptes;
	unsigned long start_code, end_code, start_data, end_data;
	unsigned long start_brk, brk, start_stack;
	unsigned long arg_start, arg_end, env_start, env_end;	
	
	cpumask_t cpu_vm_mask;
};
```

mm\_users 域纪录正在使用该地址的进程数目。mm\_count 域是 mm_struct 结构体的主引用数。只有当 mm\_users 为 0，mm\_count 值才会变成 0，表示已经没有任何指向该 mm\_struct 结构体的引用，这时该结构体就会被销毁。

mmap 和 mm\_rb 这两个不同数据结构体描述的对象是相同的：该地址空间中的全部内存区域。前者以链表形式存放而后者以红黑树的形式存放。mmap 结构体作为链表，方便简单、高效地遍历所有元素；而 mm_rb 结构体作为红黑树，方便搜索指定元素。

所有 mm\_struct 结构体都通过自身的 mmlist 域链接在一个双向链表中。该链表的首元素是 init_mm 内存描述符，他代表 init 进程的地址空间。

### 分配内存描述符

在进程的进程描述符中，mm 域存放着该进程使用的内存描述符。```fork()``` 函数利用 ```copy_mm()``` 函数复制父进程的内存描述符，也就是 current->mm 域给其子进程。而子进程的 mm\_struct 结构体是通过文件 kernel/fork.c 中的 ```allcote_mm()``` 宏从 mm\_cachep slab 缓存中分配得到的。通常每个进程都有唯一的 mm\_struct 结构体，即唯一的进程地址空间。

如果父子进程共享地址空间，可以调用 ```clone()``` 时，设置 CLONE\_VM 标志。我们把这样的进程称为线程：

```c
if (clone_flags & CLONE_VM) {
	atomic_inc(&current->mm->mm_users);
	tsk->mm = current->mm;
}
```

### 销毁内存描述符

进程退出时，内核会调用 ```exit_mm()``` 函数，该函数会调用 ```mmput()``` 函数来减少内存描述符中的 mm\_users 用户计数，如果为 0 则调用 ```mmdrop()``` 函数，减少 mm\_count 使用计数。如果使用计数也等于 0 了，则调用 ```free_mm()``` 宏通过 ```kmem_cache_free()``` 函数将 mm\_struct 结构体归还到 mm\_cachep slab 缓存中。

### mm_struct 与内核线程

内核线程没有进程地址空间，也没有相关的内存描述符。所以内核线程对应的进程描述符中的 mm 域为空。

当进程被调度时，该进程的 mm 域指向的地址空间被装载到内存，进程描述符中的 active_mm 域会被更新，指向新的地址空间。当内核线程被调度时，内核发现他的 mm 域为 NULL，就会保留前一个进程的地址空间，随后更新内核线程对应进程描述符中的 active\_mm 域，使其指向前一个进程的内存描述符。因此在需要时，内核线程可以使用前一个进程的页表。因为内核线程不访问用户空间的内存，仅仅使用地址空间中和内核内存相关的信息。

## 内存区域

内存区域由 vm\_area\_struct 结构体描述，定义在文件 <linux/mm.h> 中，内存区域在内核中也经常被称做虚拟内存区域或 VMA。

vm\_area\_struct 结构体描述了指定地址空间内连续区间上的一个独立内存范围。内核将每个内存区域作为一个单独的内存对象管理，每个内存区域都拥有一致的属性，比如访问权限等。

```c
struct vm_area_struct {
	struct mm_struct * vm_mm;	/* The address space we belong to. */
	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address within vm_mm. */

	/* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next;

	pgprot_t vm_page_prot;		/* Access permissions of this VMA. */
	unsigned long vm_flags;		/* Flags, see mm.h. */

	struct rb_node vm_rb;

	/*
	 * For areas with an address space and backing store,
	 * linkage into the address_space->i_mmap prio tree, or
	 * linkage to the list of like vmas hanging off its node, or
	 * linkage of vma in the address_space->i_mmap_nonlinear list.
	 */
	union {
		struct {  
			struct list_head list;
			void *parent;	/* aligns with prio_tree_node parent */
			struct vm_area_struct *head;
		} vm_set;

		struct raw_prio_tree_node prio_tree_node;
	} shared;

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.	A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 */
	struct list_head anon_vma_chain; /* Serialized by mmap_sem & page_table_lock */
	struct anon_vma *anon_vma;		 /* Serialized by page_table_lock */

	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops;

	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE units, *not* PAGE_CACHE_SIZE */
	struct file * vm_file;		/* File we map to (can be NULL). */
	void * vm_private_data;		/* was vm_pte (shared mem) */
	unsigned long vm_truncate_count;/* truncate_count or restart_addr */
};
```

vm\_mm 域指向和 VMA 相关的 mm\_struct 结构体，注意每个 VMA 对其相关的 mm\_struct 结构体来说都是唯一的。即使两个独立的进程将同一个文件映射到各自的地址空间，它们分别都有一个 vm\_area\_struct 结构体来标志自己的内存区域。但如果两个线程共享一个地址空间，那么它们也同时共享其中的所有 vm\_area\_struct 结构体。

### VMA 标志

VMA 标志是一种位标志，包含在 vm_flags 域内，标志了内存区域所包含的页面的行为和信息：

|  	  标志	   |  			对 VMA 及其页面的影响	 	  |
|---------------|----------------------------------------|
|   VM_READ 	|   			页面可读取				 |
|   VM_WRITE    |   			页面可写 				  |
|   VM_EXEC     |   			页面可执行  				 |
|  VM_SHARED    |   			页面可共享 		         |

当访问 VMA 时，需要查看其访问权限。比如进程的对象代码映射区域可能会标志为 VM\_READ 和 VM\_EXEC，而不会标志为 VM\_WRITE。另一方面，可执行对象数据段的映射区域被标志为 VM\_READ 和 VM\_WRITE，而 VM\_EXEC 标志对它毫无意义。只读文件数据段的映射区域仅可被标志为 VM\_READ。VM\_SHARED 指明了内存区域包含的映射是否可以在多进程间共享，如果该标志被设置，则称其为共享映射。如果未被设置，则只有一个进程可以使用该映射的内容，称其为私有映射。

VM\_IO 标志内存区域中的包含对设备 I/O 空间的映射。该标志通常在设备驱动程序执行 ```mmap()``` 函数进行 I/O 空间映射时才被设置。同时该标志也表示该内存区域不能被包含在任何进程的 coredump 中。

### 实际使用中的内存区域

可以使用 /proc 文件系统和 pmap 工具查看给定进程的内存空间和其中所含的内存区域。

## 页表

当应用程序访问一个虚拟地址时，必须将虚拟地址转化为物理地址，然后处理器才能解析地址访问请求。

Linux 中使用三级页表完成地址转换。利用多级页表能够节约地址转换需占用的存放空间。

顶级页表是页全局目录(PGD)，PGD 包含了一个 pgd_t 类型数组，PGD 中的表项指向二级页目录中的表项：PMD。

二级页表是中间页目录（PMD)，PMD 是一个 pmd_t 类型数组，其中表项指向 PTE 中的表项。

最后一级的页表简称页表，其中包含 pte_t 类型的页表项，该页表项指向物理页面。

每个进程都有自己的页表，内存描述符的 pgd 域指向的就是进程的页全局目录。操作和检索页表时必须使用 page\_table\_lock 锁。

由于每次对虚拟内存中的页面访问都必须先解析，所以页表操作的性能非常关键。为了加快搜索，多数体系结构都实现了一个翻译后缓冲器（TLB)。TLB 是一个将虚拟地址映射到物理地址的硬件缓存。

# 相关资料

[《How the Kernel Manages Your Memory》](http://duartes.org/gustavo/blog/post/how-the-kernel-manages-your-memory/)