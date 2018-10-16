---
layout: post
date: 2016-11-27T13:49:16+08:00
title: Linux 内核系列－进程通信和同步
tags: 读书笔记
---

本系列文章为阅读《现代操作系统》《UNIX 环境高级编程》和《Linux 内核设计与实现》所整理的读书笔记，源代码取自 Linux-kernel 2.6.34 版本并有做简化。

# 概念

## 竞争条件
多个进程读写某些共享数据，而最后的结果取决于进程运行的精确时许，称为竞争条件。


## 忙等待的互斥
几种实现互斥的方案：

* 屏蔽中断

  	在单处理器系统中，最简单的方法是使每个进程在刚刚进入临界区后立即屏蔽所有中断，包括时钟中断。CPU 只有在发生中断的时候才会进行进程切换，这样在中断被屏蔽后 CPU 将不会被切换到其他进程。

* 锁变量

* 严格轮换法

	```c
	while (TRUE) {
		while (turn != 0) 
		critical_region(); 
		turn = 1;
		noncritical_region();
	}


	while (TRUE) {
		while (turn != 1) 
		critical_region(); 
		turn = 0;
		noncritical_region();
	}
	```
	忙等待检查变量。使用忙等待的锁称为自旋锁。

* Peterson 解法

	```c
	#define FALSE 0 
	#define TRUE  1
	#define N 	  2 				   /* number of processes */
	
	int turn;						   /* whose turn is it? */
	int interested[N];				   /* all values initially 0 (FALSE) */

	void enter_region(int process);    /* process is 0 or 1 */
	{
		int other;

		other = 1 - process;
		interested[process] = TRUE;
		turn = process;
		while (turn == process && interested[other] == TRUE); 
	}


	void leave_region(int process) 	   /* process: who is leaving */ 
	{
		interested[process] = FALSE;   /* indicate departure from critical region */ 
	}
	```

* TSL 指令

	TSL（测试并加锁）指令将一个内存字 lock 读到寄存器 RX 中，然后在该内存地址上存一个非零值。读字和写字操作保证是不可分割的，即在该指令结束前其他处理器均不允许访问该内存字。执行 TSL 指令的 CPU 将锁住内存总线，以防止其他 CPU 在本指令结束前访问内存。

	```
	enter region:
		TSL REGISTER, LOCK 		| copy lock to register and set lock to 1
		CMP REGISTER, #0 		| was lock zero?
		JNE enter_region 		| if it was not zero, lock was set, so loop
		RET 					| return to caller; critical region entered

	leave region:
		MOVE LOCK, #0 			| store a 0 in lock
		RET 					| return to caller 	
	```

	一个可替代 TSL 的指令是 XCHG，它原子性的交换了两个位置的内容，例如寄存器和内存字。代码如下：

	```
	enter region:
		MOVE REGISTER, #1  		| put a 1 in the register
		XCHG REGISTER, LOCK 	| swap the contents of the register and lock variable
		CMP REGISTER, #0 		| was lock zero?
		JNE enter_region 		| if it was non zero, lock was set, so loop
		RET 					| return to caller; critical region entered

	leave region:
		MOVE LOCK, #0 			| store a 0 in lock
		RET 					| return to caller
	```


忙等待存在优先级反转的问题。假设存在 H 和 L 两个进程，L 优先级较低，调度规则规定只要 H 处于就绪状态就可以允许。在某一时刻，L 处于临界区中，此时 H 变到就绪态，然后开始忙等待。但由于 H 就绪时 L 不会被调度，也就无法离开临界区，所以 H 将永远忙等待下去。

## 信号量
信号量使用整型变量来累计唤醒次数。信号量的取值可以为 0 或者为正值。信号量有 up 和 down 这两种操作。

使用信号量解决生产者－消费者问题：

```c
#define N 100 					/* number of slots in the buffer */
typedef int semaphore;  		/* semaphores are a special kind of int */
semaphore mutex = 1;  			/* controls access to critical region */
semaphore empty = N;  			/* counts empty buffer slots */
semaphore full = 0; 			/* counts full buffer slots */

void producer(void) {
  int item;
  while (TRUE) { 				/* TRUE is the constant 1 */
    item = produce_item();      /* generate something to put in buffer */
	down(&empty);  				/* decrement empty count */
	down(&mutex);				/* enter critical region */
    insert_item(item);  		/* put new item in buffer */
	up(&mutex); 				/* leavecriticalregion */
	up(&full); 					/* increment count of full slots */
  } 
}

void consumer(void) {
  int item;
  while (TRUE) {   				/* infinite loop */
	down(&full); 				/* decrement full count */
    down(&mutex); 				/* enter critical region */
	item = remove_item(); 		/* take item from buffer */
	up(&mutex);  				/* leavecriticalregion */
	up(&empty); 				/* increment count of empty slots */
	consume item(item);			/* do something with the item */
  } 
}
```

中断可以用信号量来实现，最自然的方法是为每个 I/O 设备设置一个信号量，初始值为 0。在启动 I/O 设备后，管理进程就立即对相关信号量执行一个 down 操作，于是进程被阻塞。当中断到来时，中断处理程序随后对相关信号量执行一个 up 操作，从而使进程变成就绪态。

信号量可以用以实现互斥，也可以用于事件同步。

## 互斥量
互斥量是信号量的简化版本，称为互斥量。常用于用户线程包。互斥量只有两个状态：解锁和加锁。

TSL 或 XCHG 指令可以方便的在用户空间中实现互斥量：

```
mutex_lock:
	TSL REGISTER, MUTEX 				| copy mutex to register and set mutex to 1
	CMP REGISTER, #0 					| was mutex zero?
	JZE ok 								| if it was zero, mutex was unlocked, so return
	CALL thread_yield					| mutex is busy; schedule another thread
	JMP mutex_lock						| try again	
ok: RET									| return to caller; critical region entered
										

mutex_unlock:
	MOVE MUTEX, #0 						| store a 0 in mutex
 	RET 								| return to caller
```

mutex\_lock 和 enter\_region 很类似，但有个关键的区别。当 enter\_region 进入临界区失败时，进程始终在重复测试锁（忙等待），直到时钟超时，重新调度其他进程运行。

在用户线程中，由于没有时钟来停止运行过长的线程，因此需要调用 thread_yield 来让出 CPU。

Pthread 提供多种同步机制，包括互斥量和条件变量。**条件变量总是结合互斥量一起使用。**另外条件变量不会存在内存中，需要避免丢失信号。

```c
#include <stdio.h>
#include <pthread.h>
#define MAX 1000000000  								/* how many numbers to produce */
pthread_mutex_t the mutex; 
pthread_cond_t condc, condp; 							/* used for signaling */
int buffer = 0;											/* buffer used between producer and consumer */

void *producer(void *ptr) {  							/* produce data */
	int i;

	for (i= 1; i <= MAX; i++) {
		pthread_mutex_lock(&the_mutex);					/* get exclusive access to buffer */			
		while (buffer != 0) 
			pthread_cond_wait(&condp, &the_mutex);
		buffer = i;										/* put item in buffer */	
		pthread_cond_signal(&condc);					/* wakeupconsumer */
		pthread_mutex_unlock(&the_mutex); 				/* release access to buffer */
  	}
	pthread exit(0); 
}


void *consumer(void *ptr) { 							/* consume data */
	int i;
	for (i = 1; i <= MAX; i++) {
		pthread_mutex_lock(&the_mutex); 				/* get exclusive access to buffer */ 
		while (buffer ==0 ) 
			pthread_cond_wait(&condc, &the_mutex);		/* wakeupproducer */
		buffer = 0;
		pthread_cond_signal(&condp);  
		pthread_mutex_unlock(&the_mutex); 				/* release access to buffer */
	}
	pthread exit(0); 
}

int main(int argc, char **argv) {
	pthread_t pro, con;
	pthread_mutex_init(&the_mutex, 0); 
	pthread_cond_init(&condc, 0); 
	pthread_cond_init(&condp, 0); 
	pthread_create(&con, 0, consumer, 0); 
	pthread_create(&pro, 0, producer, 0); 
	pthread_join(pro, 0);
	pthread_join(con, 0);
	pthread_cond_destroy(&condc); 
	pthread_cond_destroy(&condp); 
	pthread_mutex_destroy(&the_mutex);
}
```

# 系统调用及库函数

## 线程同步

### 互斥量

可以使用 pthread 的互斥接口来保护数据。

```c
#include <pthread.h>

int pthread_mutex_lock(pthread_mutex_t *mutex);

int pthread_mutex_trylock(pthread_mutex_t *mutex);

int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

### 读写锁

读写锁可以有三个状态：读模式下加锁状态，写模式下加锁状态，不加锁状态。

```c
#include <pthread.h>

int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);

int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);

int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```

### 条件变量

条件变量与互斥量一起使用时，允许线程以无竞争的方式等待特定的条件发生。**条件本身是由互斥量保护的，线程在改变条件状态前必须先锁住互斥量**。

```c

#include <pthread.h>

int pthread_cond_wait(pthread_cond_t *restrict cond,
					 pthread_mutex_t *restrict mutex);

int pthread_cond_signal(pthread_cond_t *cond);

```

传递给 pthread\_cond\_wait 的互斥量对条件进行保护。调用者把锁住的互斥量传给函数，函数自动把调用线程放到等待条件的线程列表上，再对互斥量解锁。这就**关闭了条件检查和线程进入休眠状态等待条件改变这两个操作之间的时间通道，因此线程不会错过条件的任何变化**。pthread\_cond\_wait 返回时，互斥量再次被锁住。

### 自旋锁

自旋锁与互斥量类似，但其不是通过休眠使进程阻塞，而是获得锁之前一直处于忙等待。自锁锁可以用于以下情况：锁被持有时间短，而且线程不希望在重新调度上花费太多成本。

自旋锁通常用于底层原语用于实现其它类型的锁。

```c
#include <pthread.h>

int pthread_spin_lock(pthread_spinlock_t *lock);

int pthread_spin_trylock(pthread_spinlock_t *lock);

int pthread_spin_unlock(pthread_spinlock_t *lock);
```

### 屏障

屏障是用户协调多个线程并行工作的同步机制。屏障允许每个线程等待，直到所有的合作线程都到达某一个点，然后从该点继续执行。

```c
#include <pthread.h>

int pthread_barrier_init(pthread_barrier_t *restrict barrier,
						const pthread_barrierattr_t *restrict attr,
						unsigned int count);

int pthread_barrier_wait(pthread_barrier_t *barrier);
```


## 进程同步与通信 

### 管道

管道是 UNIX 系统 IPC 的最古老方式。管道是半双工的，只能在具有公共祖先的两个进程之间使用。

```c
#include <unistd.h>

int pipe(int fd[2]);
```

常量 PIPE\_BUF 规定了内核的管道缓冲区大小，如果写管道的字节数小于等于 PIPE\_BUF，则此操作不会与其它进程对同一管道的写操作交叉进行。

### FIFO

通过 FIFO 不相关的进程也能交换数据。创建 FIFO 类似创建文件：

```c
#include <sys/stat.h>

int mkfifo(const char *path, mode_t mode);
```

### XSI IPC

消息队列，信号量以及共享存储器称为 XSI IPC。

#### 标识符和键

每个内核中的 IPC 结构都用一个非负整数的标识符加以引用。标识符是 IPC 对象的内部名，每个 IPC 对象都与一个键关联，将这个键作为该对象的外部名。

#### 权限结构

每个 IPC 结构关联了一个 ipc_perm 结构，该结构规定了权限和所有者，其至少包括了以下成员：

```c
struct ipc_perm {
	uid_t 	uid;   /* owner's effective user id */
	gid_t 	gid;   /* owner's effective group id */
	uid_t   cuid;  /* creator's effective user id */
	gid_t 	cgid;  /* creator's effective group id */
	mode_t  mode;  /* access modes */
	...
}
```

#### 优缺点

XSI IPC 是在系统范围内起作用的，没有引用计数。例如，消息队列在引用进程终止后其内容不会被删除。与管道相比，当最后一个引用管道的进程终止时，管道就被完全的删除了。另一个问题是，XSI IPC 结构在文件系统中没有名字，无法使用文件系统的函数来访问或修改它们的属性，因此内核增加了十几个全新的系统调用（msgget、semop、shmat等）来支持这些 IPC 对象。

消息队列的优点是：可靠、流控制的。

#### 信号量、记录锁和互斥量的比较

如果在多个进程间共享一个资源，则可使用信号量、记录锁或互斥量来协调访问。

若使用信号量，则先创建包含一个成员的信号量集合，分配资源调用 sem\_op 为 -1 的 semop。释放资源调用 sem\_op 为 +1 的 semop。对每个操作都指定 SEM_UNDO，以处理未释放资源条件下进程终止的情况。

若使用记录锁，则先创建一个空文件，并使用该文件的第一个字节（无需存在）为锁字节。记录锁的性质确保当锁的持有者终止时，内核会自动释放锁。

若使用互斥量，需要所有进程将相同文件映射到各自的地址空间，并使用 PTHREAD\_PROCESS\_SHARED 互斥量属性在文件的相同偏移处初始化互斥量。如果一个进程没有释放互斥量而终止，要恢复将是非常困难的。

下图显示在 Linux 上，使用这三种不同技术进行锁操作所需要的时间。在每一种情况下，资源都被分配、释放 1 000 000 次：

|  		 操作	  	 | 用户时间 | 系统时间 |  时钟  |
|-----------------|---------|---------|-------|
| 	带undo的信号量  |  0.50   |  6.08  |  7.55  |
|    建议性记录锁   |  0.51   |  9.06  | 4.38   | 
| 共享存储中的互斥量 |  0.21   |  0.40  |  0.25  |

如果我们仅需要对单一资源加锁，不需要 XSI 信号量的所有花哨功能，记录锁将比信号量更好，使用起来更简单、速度更快，当进程终止时系统会管理遗留下来的锁。除非特别考虑性能，否则不会使用共享存储中的互斥量。首先，在多个进程间共享内存中使用互斥量来恢复一个终止的进程更困难，其次，进程共享的互斥量属性还没得到普遍的支持。
 

### POSIX 信号量

POSIX 信号量接口意在解决 XSI 信号量接口的几个缺陷：

* 更高性能的实现
* 接口使用更简单，没有信号量集
* 删除时表现更完美

**尽可能避免使用消息队列和信号量，应当考虑全双工管道和记录锁，它们使用起来更简单。**

# Linux 中的实现

内核提供了一系列实现同步的方法，包括原子操作、自旋锁、信号量、序列锁等。

## 自旋锁

自旋锁可以在中断上下文中使用，而信号量不能，因为信号量可能会休眠。在中断处理中使用了锁的话，需要禁止本地中断，否则可能会因为重复申请锁而导致死锁。

内核提供了一个方便的方法以禁止中断并申请锁：

```c
spin_lock_irqsave(&mr_lock, flags);
/* critical region ... */ 
spin_unlock_irqrestore(&mr_lock, flags);
```

spin_lock_irqsave 保存当前的中断状态，禁止本地中断并获取自旋锁。

在单处理器系统，自旋锁的实现仅仅是禁止本地中断，并禁止抢占。

## 完成变量（Completion Variables）

Completion Variables 类似信号量，可以认为是信号量的简化版本。在内核中 Completion Variables 用于 vfork 系统调用：当子进程终止时通过 Completion Variables 通知父进程。

## 顺序锁（Sequential Locks）

顺序锁提供了一个简单的机制用以读写共享的数据，其基于一个顺序计数器。当数据被写入的时候，就会获得一把锁，并且序列值会增加。在读取数据之前和之后，读取序列值。如果序列值相同的话，那么在读数据的时候，并没有写操作发生。更进一步将，如果序列值是偶数的话，说明当前没有写操作正在进行（由于初始值为0，写操作获得和释放锁时均会让序列值加1）。

顺序锁非常轻量级，适用于以下场景：

* 数据拥有非常多的读取者。
* 数据有很少的写入者。
* 数据更倾向于写，并且不允许读造成写饥饿。
* 数据非常的简单，但是因为某些原因，不能使用atomic变量。

内核中存储 uptime 的变量 jiffies 的读写就使用了顺序锁。在某些不能原子读取 64 位 jiffies_64 变量的系统上，使用顺序锁来读取：

```c
u64 get_jiffies_64(void) {
	unsigned long seq; 
	u64 ret;

	do {
		seq = read_seqbegin(&xtime_lock);
		ret = jiffies_64;
	} while (read_seqretry(&xtime_lock, seq)); 
	return ret;
}
```

更新 jiffies_64 的代码如下：

```c
write_seqlock(&xtime_lock); 
jiffies_64 += 1; 
write_sequnlock(&xtime_lock);
```