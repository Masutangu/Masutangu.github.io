---
layout: post
date: 2016-11-27T13:49:16+08:00
title: Linux 内核系列－进程通信
category: 读书笔记
---

本系列文章为阅读《现代操作系统》和《Linux 内核设计与实现》所整理的读书笔记，源代码取自 Linux-kernel 2.6.34 版本并有做简化。

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


忙等待存在优先级反转的问题。假设存在 H 和 L 两个进程，L 优先级较低，调度规则规定只要 H 处于就绪状态就可以允许。在某一时刻，L 处于临界区中，此时 H 变到就绪态，然后开始忙等待。但由于 H 就绪时 L 不会被调度，也就无法离开临界区，所以 H 将永远忙等待下去。

## 信号量
信号量使用整型变量来累计唤醒次数。信号量的取值可以为 0 或者为正值。信号量有 up 和 down 这两种操作。

使用信号量解决生产者－消费者问题：

```
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

Pthread 提供多种同步机制，包括互斥量和条件变量。**条件变量总是结合互斥量一起使用。**（为什么？另外为什么信号量不需要？）另外条件变量不会存在内存中，需要避免丢失信号。

```
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




 

 