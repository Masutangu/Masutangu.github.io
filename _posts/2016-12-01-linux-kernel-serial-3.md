---
layout: post
date: 2016-12-01T16:40:36+08:00
title: Linux 内核系列－进程调度
tags: 
  - 读书笔记
  - Linux
---

本系列文章为阅读《现代操作系统》《UNIX 环境高级编程》和《Linux 内核设计与实现》所整理的读书笔记，源代码取自 Linux-kernel 2.6.34 版本并有做简化。

# 概念

许多适用于进程调度的处理方法同样适用于线程调度。当内核管理线程的时候，调度经常是按照线程级别的，与线程所属的进程基本没有关联。

调度需要考虑 CPU 的利用率，因为进程切换的代价比较高，进程需要从用户态切换到内核态，保存当前状态，包括在进程表中存储寄存器以便之后加载。接着，调度算法选定一个新进程，将新进程的内存映像载入 MMU。除此之外，进程切换会使整个内存高速缓存失效，强迫缓存从内存中动态重新载入两次。


# Linux 中的实现
Linux 提供了抢占式的多任务模式。进程调度策略通常要在两个矛盾的目标中间寻找平衡：**进程响应迅速（响应时间短）** 和 **最大系统利用率（高吞吐量）**。

进程调度的常用策略如下：

* 进程优先级

    基于优先级的调度是最基本的一类调度算法。优先级高的先运行，相同优先级的进程按轮转方式进行调度。

    Linux 中采用了两种不同的优先级范围。第一种是用 nice 值，他的范围是从 -20 到 +19，默认为 0。越大的 nice 值意味着更低的优先级。Linux 中，nice 值代表时间片比例。可以通过 ps-el 命令来查看，NI 标记位表示进程的 nice 值。第二种是实时优先级，默认情况下变化范围从 0 到 99。越高的实时优先级值意味着进程优先级越高。任何实时进程的优先级都高于普通进程。可以通过 ps-eo 指定 rtprio 查看进程的实时优先级。显示 “-” 表示该进程不是实时进程。

* 时间片

    时间片表示进程在被抢占前可以持续运行的时间。Linux 的 CFS 调度器并不是直接分配时间片到进程，而是将处理器的使用比例分给进程，这个比例还会受 nice 值的影响。Linux 的 CFS 调度器，抢占时机取决于新进程消耗的处理器使用比，如果消耗的使用比比当前进程小，则新进程立刻投入运行，抢占当前进程。否则将推迟运行。

## Linux 调度算法

Linux 调度器是以模块方式提供的，这种模块化结构被称为**调度器类**。每个调度器都有一个优先级，基础的调度器代码定义在 kernel/sched.c 文件中，其会按照优先级顺序遍历调度类，拥有一个可执行进程的最高优先级的调度器类胜出，由其选择下一个执行的程序。

完全公平调度（CFS）是一个针对普通进程的调度类，在 Linux 中称为 SCHED\_NORMAL，CFS 算法实现定义在 kernel/sched\_fair.c 中。

CFS 基于一个简单的理念：进程调度的效果应如同系统具备一个理想的多进程任务处理器，每个进程都能获得 1/n 的处理器时间（n是指可运行进程的数量）。同时，我们可以调度给他们无限小的时间周期，所以在任何可测量周期内，我们给予 n 个进程中每个进程同样多的运行时间。

CFS 的做法是允许每个进程运行一段时间、循环轮转、选择运行最少的进程做为下一个运行进程，而不再采用分配给每个进程时间片的做法了。CFS 在所有可运行进程总数的基础上计算出一个进程应该运行多久，而不是依靠 nice 值来计算时间片。nice 值在 CFS 中被作为进程获得处理器运行比的权重。CFS 为完美多任务中的无限小周期的近似值设立了一个小目标。而这个目标称为“目标延迟”。CFS 还引入每个进程获得的时间片底线，称为最小粒度，默认值为 1ms。任何进程获得的处理器时间是由自己和其他可运行所有可运行进程 nice 值的相对差值决定的。nice 值对时间片的作用是几何加权，nice 值对应的绝对时间是处理器的使用比。

## Linux 调度的实现

### 时间记账
每次系统时钟节拍发送时，时间片都会被减少一个节拍。当进程的时间片被减少到 0 时，它就会被另一个时间片尚未为 0 的可运行进程抢占。

* 调度器结构

    CFS 使用调度器结构（定义在<linux/sched.h> 的 ```struct sched_entity``` 中）来追踪进程运行记账。

    ```c
    struct sched_entity {
        struct load_weight	load;		/* for load-balancing */
        struct rb_node		run_node;
        struct list_head	group_node;
        unsigned int		on_rq;

        u64			exec_start;
        u64			sum_exec_runtime;
        u64			vruntime;
        u64			prev_sum_exec_runtime;

        u64			last_wakeup;
        u64			avg_overlap;

        u64			nr_migrations;

        u64			start_runtime;
        u64			avg_wakeup;
    }
    ```

    进程描述符 ```struct task_struct``` 的成员变量 se 即为 ```sched_entity``` 类型。

    vruntime 变量存放进程的虚拟运行时间，该运行时间的计算经过了所有可运行进程总数的标准化。虚拟时间是以 ns 为单位的，所以 vruntime 和定时器节拍不再相关。定义在 kernel/sched\_fair.c 文件的 ```update_curr()``` 函数实现了记账功能：

    ```c
    /* cpu runqueue to which this cfs_rq is attached */
    static inline struct rq *rq_of(struct cfs_rq *cfs_rq)
    {
        return cfs_rq->rq;
    }

    static void update_curr(struct cfs_rq *cfs_rq)
    {
        struct sched_entity *curr = cfs_rq->curr;
        u64 now = rq_of(cfs_rq)->clock;
        unsigned long delta_exec;

        if (unlikely(!curr))
            return;

        /*
        * Get the amount of time the current task was running
        * since the last time we changed load (this cannot
        * overflow on 32 bits):
        */
        delta_exec = (unsigned long)(now - curr->exec_start);
        if (!delta_exec)
            return;

        __update_curr(cfs_rq, curr, delta_exec);
        curr->exec_start = now;
    }
    ```

    ```update_curr()``` 计算了当前进程的执行时间，保存在变量 ```delta_exec``` 中，然后又将其传给 ```__update_curr()``` 进行加权计算，最后将权重值与当前运行进程的 vruntime 相加。

    ```c
    /*
    * Update the current task's runtime statistics. Skip current tasks that
    * are not in our scheduling class.
    */
    static inline void
    __update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr,
            unsigned long delta_exec)
    {
        unsigned long delta_exec_weighted;

        schedstat_set(curr->exec_max, max((u64)delta_exec, curr->exec_max));

        curr->sum_exec_runtime += delta_exec;
        schedstat_add(cfs_rq, exec_clock, delta_exec);
        delta_exec_weighted = calc_delta_fair(delta_exec, curr);

        curr->vruntime += delta_exec_weighted;
        update_min_vruntime(cfs_rq);
    }

    static void update_min_vruntime(struct cfs_rq *cfs_rq)
    {
        u64 vruntime = cfs_rq->min_vruntime;

        if (cfs_rq->curr)
            vruntime = cfs_rq->curr->vruntime;

        if (cfs_rq->rb_leftmost) {
            struct sched_entity *se = rb_entry(cfs_rq->rb_leftmost,
                            struct sched_entity,
                            run_node);

            if (!cfs_rq->curr)
                vruntime = se->vruntime;
            else
                vruntime = min_vruntime(vruntime, se->vruntime);
        }

        cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, vruntime);
    }
    ```

    ```update_curr()``` 是由系统定时器周期性调用的，无论是在进程处于可运行状态还是阻塞处于不可运行状态。

### 进程选择

当 CFS 需要选择下一个运行进程时，它会挑选一个具有最小 vruntime 的进程。CFS 使用红黑树来组织可运行进程队列，其节点的键值是可运行进程的 vruntime。

* 挑选下一个任务

    CFS 选择 vruntime 最小的那个，对应的就是树中最左侧的叶子节点。实现这一过程的函数是 ```__pick_next_entity()```，定义在 ```kernel/sched_fair.c``` 中：

    ```c
    static struct sched_entity *__pick_next_entity(struct cfs_rq *cfs_rq)
    {
        struct rb_node *left = cfs_rq->rb_leftmost;

        if (!left)
            return NULL;

        return rb_entry(left, struct sched_entity, run_node);
    }
    ```

    注意 ```__pick_next_entity()``` 函数并不会遍历数找到最左叶子节点，该值已经缓存在 ```rb_leftmost``` 字段中。如果该函数返回 NULL，表示没有可运行进程，CFS 调度器选择 idle 任务运行。

* 添加进程

    添加进程和缓存最左叶子节点，发生在进程变成可运行状态（被唤醒）或者通过 ```fork()``` 调用第一次创建进程时，由调用 ```enqueue_entity()``` 实现：

    ```c
    static void enqueue_entity(
        struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
    {
        /*
        * Update the normalized vruntime before updating min_vruntime
        * through callig update_curr().
        */
        if (!(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATE))
            se->vruntime += cfs_rq->min_vruntime;  // 这里在下文有解释

        /*
        * Update run-time statistics of the 'current'.
        */
        update_curr(cfs_rq);
        account_entity_enqueue(cfs_rq, se);

        if (flags & ENQUEUE_WAKEUP) {
            place_entity(cfs_rq, se, 0);
            enqueue_sleeper(cfs_rq, se);
        }

        update_stats_enqueue(cfs_rq, se);
        check_spread(cfs_rq, se);
        if (se != cfs_rq->curr)
            __enqueue_entity(cfs_rq, se);
    }

    static void
    place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
    {
        u64 vruntime = cfs_rq->min_vruntime;

        /*
        * The 'current' period is already promised to the current tasks,
        * however the extra weight of the new task will slow them down a
        * little, place the new task so that it fits in the slot that
        * stays open at the end.
        */
        if (initial && sched_feat(START_DEBIT))
            vruntime += sched_vslice(cfs_rq, se);

        /* sleeps up to a single latency don't count. */
        if (!initial && sched_feat(FAIR_SLEEPERS)) {
            unsigned long thresh = sysctl_sched_latency;

            /*
            * Convert the sleeper threshold into virtual time.
            * SCHED_IDLE is a special sub-class.  We care about
            * fairness only relative to other SCHED_IDLE tasks,
            * all of which have the same weight.
            */
            if (sched_feat(NORMALIZED_SLEEPER) && (!entity_is_task(se) ||
                    task_of(se)->policy != SCHED_IDLE))
                thresh = calc_delta_fair(thresh, se);

            /*
            * Halve their sleep time's effect, to allow
            * for a gentler effect of sleepers:
            */
            if (sched_feat(GENTLE_FAIR_SLEEPERS))
                thresh >>= 1;  // /* 补偿减为调度周期的一半 */

            vruntime -= thresh;
        }

        /* ensure we never gain time by being placed backwards. */
        vruntime = max_vruntime(se->vruntime, vruntime);

        se->vruntime = vruntime;
    }

    ```

    enqueue_entity 更新运行时间和一些统计数据，然后调用 ```__enqueue_entity()``` 进行插入操作。

    ```c
    /*
    * Enqueue an entity into the rb-tree:
    */
    static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
    {
        struct rb_node **link = &cfs_rq->tasks_timeline.rb_node;
        struct rb_node *parent = NULL;
        struct sched_entity *entry;
        s64 key = entity_key(cfs_rq, se);
        int leftmost = 1;

        /*
        * Find the right place in the rbtree:
        */
        while (*link) {
            parent = *link;
            entry = rb_entry(parent, struct sched_entity, run_node);
            /*
            * We dont care about collisions. Nodes with
            * the same key stay together.
            */
            if (key < entity_key(cfs_rq, entry)) {
                link = &parent->rb_left;
            } else {
                link = &parent->rb_right;
                leftmost = 0;
            }
        }

        /*
        * Maintain a cache of leftmost tree entries (it is frequently
        * used):
        */
        if (leftmost)
            cfs_rq->rb_leftmost = &se->run_node;

        rb_link_node(&se->run_node, parent, link);
        rb_insert_color(&se->run_node, &cfs_rq->tasks_timeline);
    }
    ```

    > 如果新进程的 vruntime 初值为 0 的话，那么它在相当长的时间内都会保持抢占CPU的优势，老的进程就会饿死，因此 CFS 在每个 CPU 的运行队列 cfs_rq 都维护一个 min_vruntime 字段，记录该运行队列中所有进程的 vruntime 最小值，新进程的初始 vruntime 值就以它所在运行队列的 min_vruntime 为基础来设置，与老进程保持在合理的差距范围内。
    >
    > 如果休眠进程的 vruntime 保持不变，而其他运行进程的 vruntime 一直在增加，那么等到休眠进程被唤醒的时候，它的 vruntime 比别人小很多，会使其长时间拥有抢占 CPU 的优势。因此 CFS 在休眠进程被唤醒时重新设置 vruntime 值，以 min_vruntime 值为基础，给予一定的补偿，但不能补偿太多。
    >
    > 在多 CPU 的系统上，不同的CPU的负载不一样，有的 CPU 更忙一些，而每个 CPU 都有自己的运行队列，每个队列中的进程的 vruntime 也走得有快有慢。如果一个进程从 min_vruntime 更小的 CPU A上迁移到 min_vruntime 更大的 CPU B 上，可能就会占便宜了，因为 CPU B 的运行队列中进程的 vruntime 普遍比较大，迁移过来的进程就会获得更多的 CPU 时间片。为了避免这种场景，当进程从一个 CPU 的运行队列中出来 (调用 ```dequeue_entity```) 的时候，它的 vruntime 要减去队列的 min_vruntime 值；而当进程加入另一个CPU的运行队列 (调用 ```enqueue_entity```) 时，它的 vruntime 要加上该队列的 min_vruntime 值。这样，进程从一个 CPU 迁移到另一个 CPU 之后，vruntime 保持相对公平。
    >
    > 参考自[《从几个问题开始理解CFS调度器》](http://linuxperf.com/?p=42)

* 删除进程

    删除进程发生在进程阻塞或终止时：

    ```c++
    static void
    dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int sleep)
    {
        /*
        * Update run-time statistics of the 'current'.
        */
        update_curr(cfs_rq);

        update_stats_dequeue(cfs_rq, se);
        if (sleep) {
    #ifdef CONFIG_SCHEDSTATS
            if (entity_is_task(se)) {
                struct task_struct *tsk = task_of(se);

                if (tsk->state & TASK_INTERRUPTIBLE)
                    se->sleep_start = rq_of(cfs_rq)->clock;
                if (tsk->state & TASK_UNINTERRUPTIBLE)
                    se->block_start = rq_of(cfs_rq)->clock;
            }
    #endif
        }

        clear_buddies(cfs_rq, se);

        if (se != cfs_rq->curr)
            __dequeue_entity(cfs_rq, se);
        account_entity_dequeue(cfs_rq, se);
        update_min_vruntime(cfs_rq);

        /*
        * Normalize the entity after updating the min_vruntime because the
        * update can refer to the ->curr item and we need to reflect this
        * movement in our normalized position.
        */
        if (!sleep)
            se->vruntime -= cfs_rq->min_vruntime;
    }

    static void __dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
    {
        if (cfs_rq->rb_leftmost == &se->run_node) {
            struct rb_node *next_node;

            next_node = rb_next(&se->run_node);
            cfs_rq->rb_leftmost = next_node;
        }

        rb_erase(&se->run_node, &cfs_rq->tasks_timeline);
    }
    ```

### 调度器的入口

进程调度的主要入口是函数 ```schedule()```，定义在 kernel/sched.c 中。schedule() 先找到一个最高优先级的调度类，调度类有各自的可运行队列，由调度类返回最高优先级的进程：

```c
static inline struct task_struct *
    pick_next_task(struct rq *rq)
    {
        const struct sched_class *class;
        struct task_struct *p;

        /*
        * Optimization: we know that if all tasks are in
        * the fair class we can call that function directly:
        */
        if (likely(rq->nr_running == rq->cfs.nr_running)) {
            p = fair_sched_class.pick_next_task(rq);
            if (likely(p))
                return p;
        }

        class = sched_class_highest;
        for ( ; ; ) {
            p = class->pick_next_task(rq);
            if (p)
                return p;
            /*
            * Will never be NULL as the idle class always
            * returns a non-NULL p:
            */
            class = class->next;
        }
    }
```

函数开始部分的优化，CFS 是普通进程的调度类，如果所有可运行进程数量等于 CFS 类对应的可运行进程数，意味着可以直接选择 CFS 为调度类。

每个调度类都实现了 ```pick_next_task()``` 函数，其返回下一个可运行进程的指针。```pick_next_task()``` 实现中会调用 ```pick_next_entity()```，该函数会调用 ```__pick_next_entity()``` 函数。

### 睡眠和唤醒

睡眠时进程把自己标记成休眠状态，从可执行红黑树中移出，放入等待队列中，然后调用 ```schedule()``` 选择和执行下一个进程。唤醒的过程刚好相反：进程被设置为可运行状态，然后从等待队列中移到可执行红黑树中。

* 等待队列

    内核用 wake_queue_head_t 表示等待队列。休眠和唤醒实现比较复杂，因为要避免竞争条件:

    ```c
    DEFINE_WAIT(wait);

    add_wait_queue(q, &wait);
    while (!condition) {  /* condition 是我们等待的事件 */
        prepare_to_wait(&q, &wait, TASK_INTERRUPTIBLE);
        if (signal_pending(current))
            /* 处理信号 */
            schedule();
    }
    finish_wait(&q, &wait);
    ```

    步骤如下：

        1. 调用宏 DEFINE_WAIT() 创建一个等待队列的项
        2. 调用 add_wait_queue() 把自己加入到队列中，该队列会在进程等待条件满足时唤醒它
        3. 调用 prepare_to_wait() 方法将进程的状态变更为 TASK_INTERRUPTIBLE 或 TASK_UNINTERRUPTIBLE
        4. 如果状态被设置为 TASK_INTERRUPTIBLE，则信号唤醒进程，检查并处理信号
        5. 当进程被唤醒时，会再次检查条件是否为真。如果是，则退出循环。如果不是，再次调用 schedule() 
        6. 当条件满足后，进程设置为 TASK_RUNNING 并调用 finish_wait() 方法将自己移出等待队列
        

    位于 fs/notify/inotify/inotify_user.c 的 ```inotify_read()```，负责从通知文件描述符中读取信息。其实现是等待队列的典型用法：

    ```c
    static ssize_t inotify_read(struct file *file, char __user *buf,
                size_t count, loff_t *pos)
    {
        struct fsnotify_group *group;
        struct fsnotify_event *kevent;
        char __user *start;
        int ret;
        DEFINE_WAIT(wait);

        start = buf;
        group = file->private_data;

        while (1) {
            prepare_to_wait(&group->notification_waitq, &wait, TASK_INTERRUPTIBLE);

            mutex_lock(&group->notification_mutex);
            kevent = get_one_event(group, count);
            mutex_unlock(&group->notification_mutex);

            if (kevent) {
                ret = PTR_ERR(kevent);
                if (IS_ERR(kevent))
                    break;
                ret = copy_event_to_user(group, kevent, buf);
                fsnotify_put_event(kevent);
                if (ret < 0)
                    break;
                buf += ret;
                count -= ret;
                continue;
            }

            ret = -EAGAIN;
            if (file->f_flags & O_NONBLOCK)
                break;
            ret = -EINTR;
            if (signal_pending(current))
                break;

            if (start != buf)
                break;

            schedule();
        }

        finish_wait(&group->notification_waitq, &wait);
        if (start != buf && ret != -EFAULT)
            ret = buf - start;
        return ret;
    }

    void
    prepare_to_wait(wait_queue_head_t *q, wait_queue_t *wait, int state)
    {
        unsigned long flags;

        wait->flags &= ~WQ_FLAG_EXCLUSIVE;
        spin_lock_irqsave(&q->lock, flags);
        if (list_empty(&wait->task_list))
            __add_wait_queue(q, wait);
        set_current_state(state);
        spin_unlock_irqrestore(&q->lock, flags);
    }

    static inline void __add_wait_queue(wait_queue_head_t *head, wait_queue_t *new)
    {
        list_add(&new->task_list, &head->task_list);
    }
    ```

* 唤醒

    唤醒操作通过函数 ```wake_up()``` 进行，他会唤醒指定的等待队列上的所有进程。其调用 ```try_to_wake_up()```，该函数负责将进程设置为 TASK_RUNNING 状态，调用 ```enqueue_task()``` 将进程放入红黑树。如果被唤醒的进程优先级比当前正在执行的进程优先级高，还需要设置 need_resched 标志。通常哪段代码促使条件达成，就负责调用 ```wake_up()``` 函数。

## 抢占和上下文切换

上下文切换由定义在 kernel/sched.c 中的 ```context_switch()``` 函数负责。每当新进程准备投入运行时，```schedule()``` 就会调用该函数，它完成两项基本工作：

* 调用声明在 <asm/mmu_context.h> 中的 switch_mm()，该函数负责把虚拟内存从上一个进程映射切换到新进程中。
* 调用声明在 <asm/system.h> 中的 switch_to()，该函数负责从上一个进程的处理器状态切换到新进程的处理器状态，包括保存、恢复栈信息和寄存器信息，还有其他和体系结构相关的状态信息，都必须以每个进程为对象进行管理和保存。

内核提供 need\_resched 标志来表明是否需要重新调度。当某个进程应该被抢占时，```scheduler_tick()``` 就会设置这个标志；当一个优先级高的进程进入可执行状态时，```try_to_wake_up()``` 也会设置这个标志。内核检查该标志，当其被设置时调用 ```schedule()``` 来切换到新进程。

再次返回到用户空间以及从中断返回的时候，内核也会检查 need\_resched 标志。如果已被设置，内核会在继续执行前调用调度程序。

每个进程都包含一个 need\_resched 标志，这是因为访问进程描述符内的数值要比访问一个全局变量快（因为 current 宏速度很快并且描述符通常都在高速缓存中）。2.6 版本中 need\_resched 被移到 thread\_info 结构体里，用一个特别的标志变量中的一位来表示。

### 用户抢占

内核即将返回用户空间时，如果 need\_resched 标识被设置，会导致 schedule() 被调用，此时就会发生用户抢占。简而言之，用户抢占在以下情况产生：

* 从系统调用返回用户空间时
* 从中断处理程序返回用户空间时

### 内核抢占

Linux 系统中，只要重新调度是安全的，内核就可以在任何时间抢占正在执行的任务。只要没有持有锁，内核就可以抢占。

为了支持内核抢占，每个进程的 thread\_info 引入 preempt\_count 计数器。该计数器初始值为 0，每当使用锁的时候数值加 1，释放锁时数值减 1。当数值为 0 时，内核就可以抢占。从中断返回内核空间时，内核会检查 need\_resched 和 preempt\_count 的值。如果 need\_resched 被设置，并且 preempt_count 为 0，此时调度程序会被调用。

如果内核中的进程被阻塞了，或显式调用了 ```schedule()```，内核抢占也会显式地发生。 

简而言之，内核抢占会发生在：

* 中断处理程序正在执行，且返回内核空间之前
* 内核代码再一次具有可抢占性
* 如果内核中的任务显式调用 schedule()
* 如果内核中的任务阻塞


## 实时调度策略

Linux 提供了两种实时调度策略：SCHED\_FIFO 和 SCHED\_RR。而普通的、非实时的调度策略是 SCHED_NORMAL。

SCHED\_FIFO 实现了一种简单的先入先出的调度算法。处于可运行状态的 SCHED\_FIFO 级的进程会比任何 SCHED\_NORMAL 级的进程都先得到调度。一旦一个 SCHED_FIFO 级进程处于可执行状态，就会一直执行，直到受阻塞或显式释放处理器。只有更高优先级的 SCHED\_FIFO 或者 SCHED\_RR 任务才能抢占 SCHED\_FIFO 任务。

SCHED\_RR 与 SCHED\_FIFO 大体相同，只是 SCHED_RR 级的进程在耗尽事先分配的时间片后就不再继续执行。SCHED\_RR 即带有时间片的 SCHED\_FIFO，时间片只能用来重新调度同一优先级的进程。