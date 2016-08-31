---
layout: post
date: 2016-07-07T15:02:11+08:00
title: libuv 源码阅读
category: 源码阅读
---

http://blog.codingnow.com/2012/01/libuv.html

# 概念

## handles 和 requests
libuv 提供了两个抽象：handles 和 requests。Handles 是 long－lived 的，会在其 active 的时候做特定的操作。Requests 则为 short-lived 操作，request 可以独自执行或被 handle 调用执行。

## I/O loop
I/O loop（或 event loop）是 libuv 的核心。每个 I/O loop 绑定单一的线程。

> The libuv event loop (or any other API involving the loop or handles, for that matter) is not thread-safe except where stated otherwise.

event loop 采用单线程异步 IO 的形式：所有网络操作都使用 non-blocking 套接字，并使用各个平台上性能最好的 poll 机制例如 linux 上的 epoll，OSX 的 kqueue 等等。

<img src="/assets/images/libuv-source-code/illustration-1.png" width="800" />


I/O loop 的流程：

* event loop 在每次循环周期开始前都会缓存当前时间，以减少时间相关的系统调用
* 执行到期定时器的 callback 
* 执行上一轮循环推迟的 I/O callback 
* 执行 Idle handle 的 callback 
* 执行 Prepare handle 的 callback
* 计算 poll timeout
* 阻塞处理 I/O，超时时间为上一步计算的 poll timeout 
* 执行 Check handle 的 callback
* 执行 Close callback

> libuv uses a thread pool to make asynchronous file I/O operations possible, but network I/O is always performed in a single thread, each loop’s thread.

## File I/O
libuv 的异步文件 I/O 是通过线程池实现的。

> libuv currently uses a global thread pool on which all loops can queue work on. 3 types of operations are currently run on this pool:
> * Filesystem operations
> * DNS functions (getaddrinfo and getnameinfo)
> * User specified code via uv_queue_work()

## 主要结构体

### uv_loop_t
> The event loop is the central part of libuv’s functionality. It takes care of polling for i/o and scheduling callbacks to be run based on different sources of events.

uv_loop_t 执行的三种模式
UV_RUN_DEFAULT: Runs the event loop until there are no more active and referenced handles or requests. Returns non-zero if uv_stop() was called and there are still active handles or requests. Returns zero in all other cases.
UV_RUN_ONCE: Poll for i/o once. Note that this function blocks if there are no pending callbacks. Returns zero when done (no active handles or requests left), or non-zero if more callbacks are expected (meaning you should run the event loop again sometime in the future).
UV_RUN_NOWAIT: Poll for i/o once but don’t block if there are no pending callbacks. Returns zero if done (no active handles or requests left), or non-zero if more callbacks are expected (meaning you should run the event loop again sometime in the future).

[API Page](http://docs.libuv.org/en/v1.x/loop.html)

### uv_handle_t
> uv_handle_t is the base type for all libuv handle types.

> Structures are aligned so that any libuv handle can be cast to uv_handle_t. All API functions defined here work with any handle type.

* void uv_ref(uv_handle_t* handle)
    Reference the given handle. References are idempotent, that is, if a handle is already referenced calling this function again will have no effect.

* void uv_unref(uv_handle_t* handle)
    Un-reference the given handle. References are idempotent, that is, if a handle is not referenced calling this function again will have no effect.

uv_unref 主要用于计时器。[例子](https://nikhilm.github.io/uvbook/utilities.html#reference-count)：

These functions can be used to allow a loop to exit even when a watcher is active or to use custom objects to keep the loop alive.

The latter can be used with interval timers. You might have a garbage collector which runs every X seconds, or your network service might send a heartbeat to others periodically, but you don’t want to have to stop them along all clean exit paths or error scenarios. Or you want the program to exit when all your other watchers are done. In that case just unref the timer immediately after creation so that if it is the only watcher running then uv_run will still exit.

```c++
uv_loop_t *loop;
uv_timer_t gc_req;
uv_timer_t fake_job_req;

int main() {
    loop = uv_default_loop();

    uv_timer_init(loop, &gc_req);
    uv_unref((uv_handle_t*) &gc_req);

    uv_timer_start(&gc_req, gc, 0, 2000);

    // could actually be a TCP download or something
    uv_timer_init(loop, &fake_job_req);
    uv_timer_start(&fake_job_req, fake_job, 9000, 0);
    return uv_run(loop, UV_RUN_DEFAULT);
}
```

We initialize the garbage collector timer, then immediately unref it. Observe how after 9 seconds, when the fake job is done, the program automatically exits, even though the garbage collector is still running.

### uv_req_t 
> uv_req_t is the base type for all libuv request types.

### uv_timer_t
> Timer handles are used to schedule callbacks to be called in the future.

### uv_prepare_t
> Prepare handles will run the given callback once per loop iteration, right before polling for i/o.

### uv_check_t
> Check handles will run the given callback once per loop iteration, right after polling for i/o.

### uv_idle_t 
> Idle handles will run the given callback once per loop iteration, right before the uv_prepare_t handles.

> The notable difference with prepare handles is that when there are active idle handles, the loop will perform a zero timeout poll instead of blocking for i/o.

### uv_async_t
> Async handles allow the user to “wakeup” the event loop and get a callback called from another thread.

* int uv_async_init(uv_loop_t* loop, uv_async_t* async, uv_async_cb async_cb)
    Initialize the handle. A NULL callback is allowed.

    *Note Unlike other handle initialization functions, it immediately starts the handle.*

* int uv_async_send(uv_async_t* async)
    Wakeup the event loop and call the async handle’s callback.

    *Note It’s safe to call this function from any thread. The callback will be called on the loop thread.*

> libuv will coalesce calls to uv_async_send(), that is, not every call to it will yield an execution of the callback. For example: if uv_async_send() is called 5 times in a row before the callback is called, the callback will only be called once. If uv_async_send() is called again after the callback was called, it will be called again.

### uv_poll_t

>Poll handles are used to watch file descriptors for readability, writability and disconnection similar to the purpose of poll

### uv_signal_t

> Signal handles implement Unix style signal handling on a per-event loop bases.

### uv_process_t

> Process handles will spawn a new process and allow the user to control it and establish communication channels with it using streams.

### uv_stream_t
> Stream handles provide an abstraction of a duplex communication channel. uv_stream_t is an abstract type, libuv provides 3 stream implementations in the for of uv_tcp_t, uv_pipe_t and uv_tty_t.

### uv_tcp_t
> TCP handles are used to represent both TCP streams and servers. uv_tcp_t is a ‘subclass’ of uv_stream_t.

### uv_pipe_t
Pipe handles provide an abstraction over local domain sockets on Unix and named pipes on Windows. uv_pipe_t is a ‘subclass’ of uv_stream_t.

### uv_tty_t
TTY handles represent a stream for the console. uv_tty_t is a ‘subclass’ of uv_stream_t.

### uv_udp_t
UDP handles encapsulate UDP communication for both clients and servers.

### uv_fs_event_t
FS Event handles allow the user to monitor a given path for changes.

### uv_fs_poll_t
FS Poll handles allow the user to monitor a given path for changes. Unlike uv_fs_event_t, fs poll handles use stat to detect when a file has changed so they can work on file systems where fs event handles can’t.


## 线程池

> libuv provides a threadpool which can be used to run user code and get notified in the loop thread. This thread pool is internally used to run all filesystem operations, as well as getaddrinfo and getnameinfo requests.

> The threadpool is global and shared across all event loops. When a particular function makes use of the threadpool (i.e. when using uv_queue_work()) libuv preallocates and initializes the maximum number of threads allowed by UV_THREADPOOL_SIZE.