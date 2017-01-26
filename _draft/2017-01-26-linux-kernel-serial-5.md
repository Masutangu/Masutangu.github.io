---
layout: post
date: 2017-01-26T21:37:16+08:00
title: Linux 内核系列－文件系统和 IO
category: 读书笔记
---

本系列文章为阅读《现代操作系统》和《Linux 内核设计与实现》所整理的读书笔记，源代码取自 Linux-kernel 2.6.34 版本并有做简化。


# Linux 中的实现

## 虚拟文件系统

虚拟文件系统（VPS）作为内核子系统，为用户空间程序提供了文件系统相关的接口。通过虚拟文件系统，程序可以利用标准的 UNIX 文件系统调用对不同介质的不同文件系统进行读写操作。

### 通用文件系统接口

VFS 使用户可以直接使用 open()、read() 和 write() 这样的系统调用而无需考虑具体文件系统和实际物理介质。

### 文件系统抽象层

VFS 抽象层定义了所有文件系统都支持的基本的、概念上的接口和数据结构，因此能衔接各种各样的文件系统。

在内核中，除了文件系统本身外，并不需要了解文件系统的内部细节。例如用户空间程序执行如下的操作：

```c 
write(f, &buf, len);
```

该代码将 &buf 指针指向的、长度为 len 字节的数据写入文件描述符 f 对应的文件的当前位置。该用户调用首先被一个通用系统调用 ```sys_write()``` 处理，```sys_write()``` 函数要找到 f 所在的文件系统对应的写操作，然后执行该操作。

### Unix 文件系统

Unix 使用了四种和文件系统相关的传统抽象概念：文件、目录项、索引节点和安装点（mount point）。从本质上讲，文件系统是特殊的数据分层存储结构，它包含文件、目录和相关的控制信息。文件系统的通用操作包含创建、删除和安装等等。在 Unix 中，文件系统被安装在一个特定的安装点上，该安装点在全局层次结构中被称为命名空间，所有的已安装文件系统都作为根文件系统树的枝叶出现在系统中。

Unix 中，目录属于普通文件，它列出包含在其中的所有文件。因此可以对目录执行和文件相同的操作。 

Unix 将文件的相关信息和文件本身着两个概念加以区分，例如访问控制权限、大小、拥有者、创建时间等信息。文件相关信息有时被称为索引节点（inode）。

文件系统的控制信息存储在超级块中，超级块是一种包含文件系统信息的数据结构。我们将文件信息和文件系统的信息统称为文件系统数据元。

Unix 文件系统在他们物理磁盘布局中也是按照上述的概念实现的。文件信息按照索引节点形式存储在单独的块中，控制信息集中存储在磁盘的超级块中。

### VFS 对象及其数据结构

VFS 采用的是面向对象的设计思路，使用一族数据结构来代表通用文件对象。VFS 中有四个主要的对象类型：

* 超级块对象，代表一个已安装文件系统
* 索引节点对象，代表一个文件
* 目录项对象，代表一个目录项，是路径的一个组成部分
* 文件对象，代表进程打开的文件

### 超级块对象

各种文件系统都必须实现超级块，该对象用于存储特定文件系统信息，通常对应存放于磁盘特定扇区中的文件系统超级块或文件系统控制块。对于并非基于磁盘的文件系统，它们会在现场创建超级块并保存到内存中。

超级块对象由 super\_block 结构体表示，定义在文件 <linux/fs.h> 中。

### 索引节点对象

索引节点对象包含了内核在操作文件或目录时需要的全部信息，由 inode 结构体表示，定义在文件 <linux/fs.h> 中。

### 目录项对象

VFS 经常需要执行目录相关的操作，例如路径名查找等。路径名查找需要解析路径中的每一个组成部分。为了方便查找操作，VFS 引入目录项的概念。每个 dentry 代表路径中的一个特定部分。在路径中，包括普通文件在内，每一个部分都是目录项对象。不同于前面的两个对象，目录项对象没有对应的磁盘数据结构，VFS 根据字符串形式的路径名现场创建它。

#### 目录项状态

目录项对象有三种有效状态：被使用、未被使用和负状态。

一个被使用的目录项对应一个有效的索引节点（d\_inode 指向相应的索引节点）并表明该对象存在一个或多个使用者（d\_count 为正值）。

一个未被使用的目录项对应一个有效的索引节点（d\_inode 指向一个索引节点）但 VFS 当前并未使用它（d\_count 为 0)。该目录项对象仍然指向一个有效对象，而且被保留在缓存中以便需要时再使用。

一个负状态的目录项没有对应的有效索引节点（d\_inode 为 NULL），因为索引节点已被删除或路径不再正确。

#### 目录项缓存

内核将目录项对象缓存在目录项缓存（dcache）中。目录项缓存包含三个主要部分：

* “被使用的”目录项链表：该链表通过索引节点对象中的 i\_dentry 项连接相关的索引节点。一个给定的索引节点可能有多个链接，就可能有多个目录项对象，因此用链表来连接。
* “最近被使用的”双向链表：该链表含有未被使用的和负状态的目录项对象。该链表以时间顺序插入，所以链头的节点是最新数据。当内核必须通过删除节点项回收内存时，会从链尾删除节点项。
* 散列表和相应的散列函数用来快速将给定路径解析为相关目录项对象。

#### 文件对象

文件对象表示进程已打开的文件。文件对象通过 f\_dentry 指针指向相关的目录项对象，目录项会指向相关的索引节点，索引节点会记录文件是否为脏。

### 文件系统相关的数据结构

struct file\_system\_type 用来描述每种文件系统的功能和行为：

```c
struct file_system_type {
	const char *name;  				/* 文件系统的名字 */
	int fs_flags;					/* 文件系统类型标志 */

	/* 从磁盘读取超级块 */
	int (*get_sb) (struct file_system_type *, int,
		       const char *, void *, struct vfsmount *);

	/* 终止访问超级块 */
	void (*kill_sb) (struct super_block *);

	struct module *owner;			/* 文件系统模块 */
	struct file_system_type * next; /* 链表中下一个 */
	struct list_head fs_supers;     /* 超级块对象链表 */
	
};
```

当文件系统被实际安装时，将有一个 vfsmount 结构体在安装点被创建。该结构体用来代表文件系统的实例：

```c
struct vfsmount {
	struct list_head mnt_hash;
	struct vfsmount *mnt_parent;	/* fs we are mounted on */
	struct dentry *mnt_mountpoint;	/* dentry of mountpoint */
	struct dentry *mnt_root;	/* root of the mounted tree */
	struct super_block *mnt_sb;	/* pointer to superblock */
	struct list_head mnt_mounts;	/* list of children, anchored here */
	struct list_head mnt_child;	/* and going through their mnt_child */
	int mnt_flags;
	/* 4 bytes hole on 64bits arches */
	const char *mnt_devname;	/* Name of device e.g. /dev/dsk/hda1 */
	struct list_head mnt_list;
	struct mnt_namespace *mnt_ns;	/* containing namespace */
	int mnt_id;			/* mount identifier */
};
```

### 进程相关的数据结构

有三个数据结构将 VFS 层和系统进程紧密联系在一起，分别是：files\_struct、fs\_struct 和 namespace 结构体。

files\_structt 结构体定义在文件 <linux/file.h> 中。该结构体由进程描述符中的 files 域指向。所有与每个进程（per-process）相关的信息如果打开的文件和文件描述符都在其中：

```c
struct fdtable {
	unsigned int max_fds;
	struct file ** fd;      /* current fd array */
	fd_set *close_on_exec;
	fd_set *open_fds;
	struct rcu_head rcu;
	struct fdtable *next;
};

struct files_struct {
  /*
   * read mostly part
   */
	atomic_t count;
	struct fdtable *fdt;
	struct fdtable fdtab;
	int next_fd;
	struct embedded_fd_set close_on_exec_init;
	struct embedded_fd_set open_fds_init;
	struct file * fd_array[NR_OPEN_DEFAULT];
};
```

fd 数组指针指向已打开的文件对象链表，默认情况下指向 fd\_array 数组。因为 NR\_OPEN\_DEFAULT 等于 32，如果一个进程打开的文件对象超过 32 个，内核将分配一个新数组并将 fd 指针指向它。

和进程相关的第二个结构体是 fs\_struct。该结构由进程描述符 fs 域指向。它包含文件系统和进程相关的信息，定义在 <linux/fs_struct.h> 中：

```c
struct path {
	struct vfsmount *mnt;
	struct dentry *dentry;
};

struct fs_struct {
	int users; 					/* 结构的使用计数 */
	rwlock_t lock;   
	int umask;					/* 默认的文件访问权限 */
	int in_exec;			 
	struct path root, pwd;     /* 根目录和当前目录的目录项对象和安装点对象 */  
};
```

最后一个结构体是 mnt_namespace，定义在 <linux/mnt_namespace.h> 中，由进程描述符中的 namespace 指向。

```c
struct mnt_namespace {
	atomic_t		count;  		/* 结构的使用计数 */
	struct vfsmount *	root; 		/* 根目录的安装点对象 */
	struct list_head	list;   	/* 安装点链表 */
	wait_queue_head_t poll; 
	int event;
};
```

对大多数进程来说，它们的描述符会指向唯一的 files\_struct 和 fs\_struct 结构体。对于使用克隆标志 CLONE\_FILES 或 CLONE\_FS 创建的进程，会共享这两个结构体。所以多个进程描述符可能会指向同一个 files\_struct 或 fs\_struct 结构体。因此每个结构体维护一个 count 域（或 users 域）作为引用计数。

namespace 结构体则不同，默认情况下，所有进程共享同样的命名空间。

## 块 I/O 层

系统能随机访问固定大小数据片（chunk）的设备被称作块设备。另一种基本的设备类型是字符设备，字符设备按照字符流的方式被有序访问，例如串口和键盘。

### 解剖一个块设备

块设备中最小的可寻址单元是扇区（sector）。扇区大小一般是 2 的整数倍，最常见的大小是 512 个字节。扇区的大小是设备的物理属性，扇区是所有块设备的基本单元，块设备无法对它还小的单元进行寻址的操作。

最小逻辑可寻址单元是块（block）。块是文件系统的抽象，只能基于块来访问文件系统。虽然物理磁盘寻址是按照扇区级进行的，但内核执行的所有磁盘操作都是按照块进行的。对块大小的要求是：必须是扇区大小的 2 的整数倍，并且要小于页面大小。所以块大小通常为 512 字节、1K 或 4K。

### 缓冲区和缓冲区头

当块被调入内存时，它要存储在一个缓冲区中。缓冲区相当于磁盘块在内存中的表示。由于内核在处理数据时需要一些相关的控制信息（比如块属于哪一个块设备，对于哪个缓冲区等），所以每个缓冲区都有一个对应的描述符，由 buffer\_head 结构体表示，称为缓冲区头，在 <linux/buffer_head.h> 中定义。

```c
struct buffer_head {
	unsigned long b_state;		/* buffer state bitmap (see above) */
	struct buffer_head *b_this_page;/* circular list of page's buffers */
	struct page *b_page;		/* the page this bh is mapped to */

	sector_t b_blocknr;		/* start block number */
	size_t b_size;			/* size of mapping */
	char *b_data;			/* pointer to data within the page */

	struct block_device *b_bdev;
	bh_end_io_t *b_end_io;		/* I/O completion */
 	void *b_private;		/* reserved for b_end_io */
	struct list_head b_assoc_buffers; /* associated with another mapping */
	struct address_space *b_assoc_map;	/* mapping this buffer is
						   associated with */
	atomic_t b_count;		/* users using this buffer_head */
};
```

与缓冲区对应的磁盘物理块由 b\_blocknr 域索引，该值是 b\_bdev 域指明的块设备中的逻辑块号。

与缓冲区对应的内存物理页由 b\_page 域表示，另外 b\_data 域直接指向相应的块（它位于 b\_page 域所指明的页面的某个位置上）。块的大小由 b\_size 域表示，所以块在内存中的起始位置在 b\_data 处，结束位置在 (b\_data + b\_size) 处。

### bio 结构体

内核中块 I/O 操作的基本容器由 bio 结构体表示，定义在文件 <linux/bio.h> 中。

每个块 I/O 请求都通过一个 bio 结构体表示。每个请求包含了一个或多个块，这些块存储在 bio\_vec 结构体数组中。这些结构体描述了每个片段在物理页中的实际位置，并且像 vector 一样被组织在一起。I/O 操作的第一个片段由 b\_io\_vec 结构体所指向，共有 bi\_vcnt 个片段。当块 I/O 层开始执行请求、需要使用各个片段时，bi\_idx 域会不断更新，从而指向当前片段。

buffer\_head 和 bio 结构体之间存在明显差别。bio 结构体代表的是 I/O 操作，它可以包含内存中的一个或多页；而另一方面，buffer\_head 结构体代表的是一个缓冲区，它描述的仅仅是磁盘中的一个块。因为缓冲区头关联的是单独页中的单独磁盘块，所以它可能会引起不必要的分割，将请求按块为单位划分。bio 则不需要连续存储区，也不需要分割 I/O 操作。

### 请求队列

块设备将它们挂起的块 I/O 请求保存在请求队列中，该队列由 reques\_queue 结构体表示，定义在 <linux/blkdev.h> 中，包含一个双向请求链表以及相关控制信息。请求队列表中的每一项都是一个单独的请求，由 request 结构体表示。一个请求可能要操作多个连续的磁盘块，所以每个请求可以由多个 bio 结构体组成。

### I/O 调度程序

磁盘寻址是整个计算机中最慢的操作之一，缩短寻址时间是提高系统性能的关键。为了优化寻址操作，内核会在提交任务前先执行合并与排序的预操作。在内核中负责提交 I/O 请求的子系统称为 I/O 调度程序。

#### I/O 调度程序的工作

I/O 调度程序的工作是管理块设备的请求队列。I/O 调度程序通过**合并**和**排序**来减少磁盘寻址时间。

#### Linus 电梯

第一个 I/O 调度程序被称为 Linus 电梯。当有新的请求加入队列时，会先检查每一个挂起的请求是否可以和新请求合并。Linus 电梯 I/O 调度程序可以执行向前和向后合并。如果合并失败，那就需要寻找可能的插入点。如果找到就插入，如果没合适的位置，那么新请求就被加入到队列尾部。另外如果队列中有驻留时间过长的请求，那么新请求也将被加入到队列尾部，即使插入后还要排序。这是为了避免由于访问相近磁盘位置的请求太多，从而造成访问磁盘其他位置的请求难以得到执行机会。

#### 最终期限 I/O 调度程序

最终期限 I/O 调度程序中，每个请求都有一个超时时间。默认情况下，读请求的超时时间是 500 毫秒，写请求的超时时间是 5 秒。最终期限 I/O 调度请求类似于 Linus 电梯，也以磁盘物理位置次序维护请求队列，这个队列称为排序队列。但同时也会根据请求类型将它们插入到额外队列中。读请求按次序被插入特定的读 FIFO 队列中，写请求被插入到特定的写 FIFO 队列中。一般情况下，调度程序从排序队列头部取出请求，再推入到派发队列中。如果在写／读 FIFO 队列头的请求超时，那么调度程序便从 FIFO 队列中提取请求。

## 页高速缓存和页回写

页高速缓存是 Linux 内核实现的一种主要磁盘缓存。它主要用来减少对磁盘的 I/O 操作。具体来讲，是通过把磁盘中的数据缓存到物理内存，把对磁盘的访问变成对物理内存的访问。

页高速缓存是由 RAM 中的物理页组成，每一页都对应磁盘多个块。每当内核开始执行一个页 I/O 操作时，首先会检查需要的数据是否在高速缓存中，如果在，则直接使用高速缓存中的数据。

也可以通过块 I/O 缓冲区把独立的磁盘块与页高速缓存联系在一起。通过缓存磁盘块以及缓冲块 I/O 操作，页高速缓存同样可以减少块 I/O 操作期间的磁盘访问量。这种缓存通常称为“缓冲区高速缓存”，也是页高速缓存中的一部分。

### 页高速缓存

页高速缓存包含了最近被访问过的文件的全部页面，在执行 I/O 操作前，内核会检查数据是否已经在页高速缓存中了，如果在，则不再需要从磁盘读取数据。

#### address_space 对象

一个物理页可能由多个不连续的物理磁盘块组成，所以在页高速缓存中检测特定数据是否已经被缓存是件非常困难的工作。

Liunx 页高速缓存使用 address\_space 结构体描述页高速缓存中的页面。该结构定义在 <linux/fs.h>：

```c
struct address_space {
	struct inode			*host;		/* owner: inode, block_device */
	struct radix_tree_root	page_tree;	/* radix tree of all pages */
	spinlock_t				tree_lock;	/* and lock protecting it */
	unsigned int			i_mmap_writable;/* count VM_SHARED mappings */
	struct prio_tree_root	i_mmap;		/* tree of private and shared mappings */
	struct list_head		i_mmap_nonlinear;/*list VM_NONLINEAR mappings */
	spinlock_t				i_mmap_lock;	/* protect tree, count, list */
	unsigned int			truncate_count;	/* Cover race condition with truncate */
	unsigned long			nrpages;	/* number of total pages */
	pgoff_t					writeback_index;/* writeback starts here */
	const struct address_space_operations *a_ops;	/* methods */
	unsigned long			flags;		/* error bits/gfp mask */
	struct backing_dev_info *backing_dev_info; /* device readahead, etc */
	spinlock_t				private_lock;	/* for use by the address_space */
	struct list_head		private_list;	/* ditto */
	struct address_space	*assoc_mapping;	/* ditto */
};
```

i\_mmap 字段是个优先搜索树，它的搜索范围包含了在 address\_space 中所有共享和私有的映射页面。address\_space 结构往往会和某些内核对象关联。通常会与一个索引节点（inode）关联，这时 host 域就指向该索引节点，该索引节点的 i\_mapping 域指向到 address_space 对象，方便查找自身文件数据是否已经缓存。

a\_ops 域指向地址空间对象中的操作函数表。

struct page 中有两个字段：mapping 和 index。其中 mapping 指向该页所有者的 address\_space，index 字段表示所有者地址空间中以页大小为单位的偏移量。用这两个字段就能在页高速缓存中查找。


页高速缓存通过两个参数：address\_space 对象和一个偏移量进行搜索。每个 address_space 对象都有唯一一个基树，保存在 page\_tree 结构体中。基树是一个二叉树，只要指定了文件偏移量，就可以在基树中迅速检索到希望的数据。

inode、address space 和 page 三者的关系如下图：

<img src="/assets/images/linux-kernel-serial-5/illustration-1.png" width="800" />

（补充下 apue 中缓存 io 的内容）
