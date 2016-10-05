---
layout: post
date: 2016-10-05T15:59:32+08:00
title: 浅读 libco 
category: 源码阅读
---

今天花了一天时间，学习了下微信的开源协程库 [libco](https://github.com/tencent-wechat/libco)的代码，写下来做个纪录，有部分细节代码（包括 coctx_swap.S 那段汇编）我还没读懂，以后再补充进来。


## 协程的原理
协程的概念和优点这里不再赘述。我们先介绍下实现协程的原理，再来看看相应的代码。

协程的切换，其实就是由我们手动来管理指令执行的上下文。一般每一个协程有自己的 context_buff 来保存自己的运行上下文（寄存器和栈）。当需要挂起当前协程时，将当前的上下文保存到该协程的 context_buff，并把当前上下文重置为新的协程的 context_buff 即可。

## hook 系统调用
一般会在有 IO 阻塞操作的时候做协程的切换。如何让协程的使用者无需关心这些切换的细节呢？libco 采用 hooking IO 函数的方法。将 IO 设置为非阻塞，提交给 epoll 来管理，并让出 cpu 资源，等到 epoll 事件返回后再 resume 对应的协程。下面的代码会给出相应的例子。

## libco 源码

* 协程相关数据结构

	stCoRoutineEnv_t 管理协程的结构体，每起一个新的协程就压入 pCallStack 中，每挂起一个协程就将其踢出 pCallStack。

	```c++
	struct stCoRoutineEnv_t
	{
		stCoRoutine_t *pCallStack[ 128 ];
		int iCallStackSize;
		stCoEpoll_t *pEpoll;
	};
	```

	stCoRoutine_t 封装了协程对象，coctx_t 保存协程的 context。

	```c++
	struct stCoRoutine_t
	{
		stCoRoutineEnv_t *env;
		pfn_co_routine_t pfn;
		void *arg;
		coctx_t ctx;
		char cStart;
		char cEnd;
		stCoSpec_t aSpec[1024];
		char cIsMain;
		char cEnableSysHook;
		char sRunStack[ 1024 * 128 ];
	};
	```

* epoll 相关数据结构

	管理 epoll 的结构体，pTimeout 管理 Timeout 事件。pstTimeoutList 为超时事件的列表。pstTimeoutList 和 pstActiveList 下面会看到其用法。

	```c++
	struct stCoEpoll_t
	{
		int iEpollFd;
		static const int _EPOLL_SIZE = 1024 * 10;
		struct stTimeout_t *pTimeout;
		struct stTimeoutItemLink_t *pstTimeoutList;  
		struct stTimeoutItemLink_t *pstActiveList;
		co_epoll_res *result; 

	};
	```

	stTimeout_t 封装了 Timeout 结构体。pItems 是一个链表数组，具体用法可以查看 ```int AddTimeout( stTimeout_t *apTimeout,stTimeoutItem_t *apItem ,unsigned long long allNow )``` 函数。
	
	```c++
	struct stTimeout_t
	{
		stTimeoutItemLink_t *pItems;
		int iItemSize;
		unsigned long long ullStart;
		long long llStartIdx;
	};
	```


* hooking IO

	下面以 read 函数为例子，看看 libco 是如何将 io 操作通过 epoll 来管理并与协程结合起来。

	```c++
	ssize_t read( int fd, void *buf, size_t nbyte )
	{
		HOOK_SYS_FUNC( read );
		if( !co_is_enable_sys_hook() )  // 如果没开启hook，则返回系统的 read 函数
		{
			return g_sys_read_func( fd,buf,nbyte );
		}
		rpchook_t *lp = get_by_fd( fd );

		if( !lp || ( O_NONBLOCK & lp->user_flag ) ) 
		{
			ssize_t ret = g_sys_read_func( fd,buf,nbyte );
			return ret;
		}
		int timeout = ( lp->read_timeout.tv_sec * 1000 ) 
					+ ( lp->read_timeout.tv_usec / 1000 );

		struct pollfd pf = { 0 };
		pf.fd = fd;
		pf.events = ( POLLIN | POLLERR | POLLHUP );

		int pollret = poll( &pf,1,timeout );  // 这里调用 poll 了，注意 poll 会挂起当前协程让出cpu，下面的指令需要等到该协程 resume 才会继续执行了。

		ssize_t readret = g_sys_read_func( fd,(char*)buf ,nbyte );
		
		return readret;
		
	}

	int poll(struct pollfd fds[], nfds_t nfds, int timeout)
	{

		HOOK_SYS_FUNC( poll );

		if( !co_is_enable_sys_hook() )
		{
			return g_sys_poll_func( fds,nfds,timeout );
		}

		return co_poll( co_get_epoll_ct(),fds,nfds,timeout );  // 调用的是 co_poll
	}
	```

* epoll 逻辑	

	最主要的代码都在 co_poll 函数和 co_eventloop 函数。

	```c++
	int co_poll( stCoEpoll_t *ctx,struct pollfd fds[], nfds_t nfds, int timeout )
	{
		int epfd = ctx->iEpollFd;

		//1.struct change
		stPoll_t arg;
		memset( &arg,0,sizeof(arg) );

		arg.iEpollFd = epfd;
		arg.fds = fds;
		arg.nfds = nfds;

		stPollItem_t arr[2];
		if( nfds < sizeof(arr) / sizeof(arr[0]) )
		{
			arg.pPollItems = arr;
		}	
		else
		{
			arg.pPollItems = (stPollItem_t*)malloc( nfds * sizeof( stPollItem_t ) );
		}

		memset( arg.pPollItems, 0, nfds * sizeof(stPollItem_t) );

		arg.pfnProcess = OnPollProcessEvent;  // 当 epoll 事件被触发，就会调用该函数来 resume 相应的协程。
		arg.pArg = GetCurrCo( co_get_curr_thread_env() );  // pArg 保存当前的协程，pfnProcess 函数中用该字段来得到需要 resume 的协程对象。
		
		//2.add timeout

		unsigned long long now = GetTickMS();
		arg.ullExpireTime = now + timeout;
		int ret = AddTimeout( ctx->pTimeout,&arg,now );  // 调用 AddTimeout，由 stCoEpoll_t 管理超时。
		
		//3. add epoll

		for(nfds_t i = 0; i < nfds; i++)
		{
			arg.pPollItems[i].pSelf = fds + i;
			arg.pPollItems[i].pPoll = &arg;

			arg.pPollItems[i].pfnPrepare = OnPollPreparePfn;
			struct epoll_event &ev = arg.pPollItems[i].stEvent;

			if( fds[i].fd > -1 )
			{
				ev.data.ptr = arg.pPollItems + i;
				ev.events = PollEvent2Epoll( fds[i].events );

				co_epoll_ctl( epfd,EPOLL_CTL_ADD, fds[i].fd, &ev );  // 添加到 epoll 中监听
			}
			//if fail,the timeout would work
			
		}

		co_yield_env( co_get_curr_thread_env() );  // 让出 cpu，挂起当前协程了。等到 stCoEpoll_t resume 该协程再继续执行下面的指令了

		// 下面都是清理工作 可以不用细看
		RemoveFromLink<stTimeoutItem_t,stTimeoutItemLink_t>( &arg );
		for(nfds_t i = 0;i < nfds;i++)
		{
			int fd = fds[i].fd;
			if( fd > -1 )
			{
				co_epoll_ctl( epfd,EPOLL_CTL_DEL,fd,&arg.pPollItems[i].stEvent );
			}
		}


		if( arg.pPollItems != arr )
		{
			free( arg.pPollItems );
			arg.pPollItems = NULL;
		}
		return arg.iRaiseCnt;
	}

	// 事件循环
	void co_eventloop( stCoEpoll_t *ctx,pfn_co_eventloop_t pfn,void *arg )
	{
		if( !ctx->result )
		{
			ctx->result =  co_epoll_res_alloc( stCoEpoll_t::_EPOLL_SIZE );
		}
		co_epoll_res *result = ctx->result;

		for(;;)
		{
			int ret = co_epoll_wait( ctx->iEpollFd, result, stCoEpoll_t::_EPOLL_SIZE, 1 );

			stTimeoutItemLink_t *active = (ctx->pstActiveList);
			stTimeoutItemLink_t *timeout = (ctx->pstTimeoutList);

			memset( timeout,0,sizeof(stTimeoutItemLink_t) );

			for(int i=0;i<ret;i++)
			{
				stTimeoutItem_t *item = (stTimeoutItem_t*)result->events[i].data.ptr;
				if( item->pfnPrepare )
				{
					item->pfnPrepare( item,result->events[i],active );
				}
				else
				{
					AddTail( active,item );  // 监听到的事件放到 active 链表里
				}
			}


			unsigned long long now = GetTickMS();
			TakeAllTimeout( ctx->pTimeout,now,timeout );  // 超时的事件放到 timeout 链表里

			stTimeoutItem_t *lp = timeout->head;
			while( lp )
			{
				lp->bTimeout = true;
				lp = lp->pNext;
			}

			Join<stTimeoutItem_t,stTimeoutItemLink_t>( active,timeout );  // 合并 active 和 timeout 链表

			lp = active->head;
			while( lp )
			{
				PopHead<stTimeoutItem_t,stTimeoutItemLink_t>( active );
				if( lp->pfnProcess )
				{
					lp->pfnProcess( lp );  // 一个个拿出来处理，调用 pfnProcess 函数，即 OnPollProcessEvent 函数
				}
				lp = active->head;
			}
			if( pfn )
			{
				if( -1 == pfn( arg ) )
				{
					break;
				}
			}
		}
	}

	void OnPollProcessEvent( stTimeoutItem_t * ap )
	{
		stCoRoutine_t *co = (stCoRoutine_t*)ap->pArg;  // 从上面知道，pArg 保存了该事件对应的协程
		co_resume( co );   // resume 相应的协程
	}
	```

* 协程的挂起和恢复

	协程的挂起和恢复由 stCoRoutineEnv_t 的 pCallStack 来管理。

	```
	void co_yield_env( stCoRoutineEnv_t *env )  // 挂起当前协程，恢复其父协程
	{
		stCoRoutine_t *last = env->pCallStack[ env->iCallStackSize - 2 ];  // last 可以认为是父协程
		stCoRoutine_t *curr = env->pCallStack[ env->iCallStackSize - 1 ];  // curr 为当前执行的协程
		env->iCallStackSize--;
		coctx_swap( &curr->ctx, &last->ctx );  // 保持 curr 协程的上下文， 并恢复 last 协程的上下文
	}

	void co_resume( stCoRoutine_t *co )   // 恢复 co 协程
	{
		stCoRoutineEnv_t *env = co->env;
		stCoRoutine_t *lpCurrRoutine = env->pCallStack[ env->iCallStackSize - 1 ];
		if( !co->cStart )
		{
			coctx_make( &co->ctx,(coctx_pfn_t)CoRoutineFunc,co,0 );
			co->cStart = 1;
		}
		env->pCallStack[ env->iCallStackSize++ ] = co;  // 执行协程的时候压入 pCallStack 栈中
		coctx_swap( &(lpCurrRoutine->ctx),&(co->ctx) );  // 恢复 co 协程的上下文
	}
	```


## 流程图
简化版的流程如下图所示：
<img src="/assets/images/learn-libco/illustration-1.png" width="800" />
