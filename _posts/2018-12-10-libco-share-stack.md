---
layout: post
date: 2018-12-10T22:54:42+08:00
title: Libco 之共享栈
tags: 
  - 源码阅读
  - 协程
---


# 代码埋点

结构体：

```c
struct stStackMem_t
{
	stCoRoutine_t* ocupy_co;  // 被哪个协程占用
	int stack_size;
	char* stack_bp; //stack_buffer + stack_size
	char* stack_buffer;
};

struct stShareStack_t
{
	unsigned int alloc_idx;
	int stack_size;
	int count;
	stStackMem_t** stack_array;
};

struct stCoRoutineAttr_t
{
	int stack_size;
	stShareStack_t*  share_stack;
	stCoRoutineAttr_t()
	{
		stack_size = 128 * 1024;
		share_stack = NULL;
	}
}__attribute__ ((packed));
```

创建协程时，如果是共享栈，则调用 ```co_get_stackmem``` 返回预先分配好的内存：

```c

static stStackMem_t *co_get_stackmem(stShareStack_t *share_stack) {
  if (!share_stack) {
    return NULL;
  }
  int idx = share_stack->alloc_idx % share_stack->count;
  share_stack->alloc_idx++;

  return share_stack->stack_array[idx];
}

struct stCoRoutine_t *co_create_env(stCoRoutineEnv_t *env,
                                    const stCoRoutineAttr_t *attr,
                                    pfn_co_routine_t pfn, void *arg) {
  stCoRoutineAttr_t at;
  if (attr) {
    memcpy(&at, attr, sizeof(at));  // 如果 attr 是栈上分配的 可能会有问题 安全起见 memcpy 一份
  }
  
  ...

  stCoRoutine_t *lp = (stCoRoutine_t *)malloc(sizeof(stCoRoutine_t));

  memset(lp, 0, (long)(sizeof(stCoRoutine_t)));

  lp->env = env;
  lp->pfn = pfn;
  lp->arg = arg;

  stStackMem_t *stack_mem = NULL;
  if (at.share_stack) {  // 是否使用共享栈
    stack_mem = co_get_stackmem(at.share_stack);
    at.stack_size = at.share_stack->stack_size;
  } else {
    stack_mem = co_alloc_stackmem(at.stack_size);
  }
  lp->stack_mem = stack_mem;

  lp->ctx.ss_sp = stack_mem->stack_buffer;
  lp->ctx.ss_size = at.stack_size;

  lp->cStart = 0;
  lp->cEnd = 0;
  lp->cIsMain = 0;
  lp->cEnableSysHook = 0;
  lp->cIsShareStack = at.share_stack != NULL;  // 标记该协程是否使用共享栈

  lp->save_size = 0;
  lp->save_buffer = NULL;

  return lp;
}
```

切换协程时，如果是共享栈，需要把当前运行栈保存到自己的 ```save_buffer```。切换回来后，需要从自己的 ```save_buffer``` 恢复运行栈：

```c
// 将协程共享栈的内容 拷贝到 save_buffer 中保存
void save_stack_buffer(stCoRoutine_t *ocupy_co) {
  /// copy out
  stStackMem_t *stack_mem = ocupy_co->stack_mem;
  int len = stack_mem->stack_bp - ocupy_co->stack_sp;

  if (ocupy_co->save_buffer) {
    free(ocupy_co->save_buffer), ocupy_co->save_buffer = NULL;
  }

  ocupy_co->save_buffer = (char *)malloc(len);  // malloc buf;
  ocupy_co->save_size = len;

  memcpy(ocupy_co->save_buffer, ocupy_co->stack_sp, len);
}

void co_swap(stCoRoutine_t *curr, stCoRoutine_t *pending_co) {
  stCoRoutineEnv_t *env = co_get_curr_thread_env();

  // get curr stack sp
  char c;
  curr->stack_sp = &c;

  if (!pending_co->cIsShareStack) {  // 没有使用共享栈
    env->pending_co = NULL;
    env->ocupy_co = NULL;
  } else {
    env->pending_co = pending_co;
    // get last occupy co on the same stack mem
    stCoRoutine_t *ocupy_co = pending_co->stack_mem->ocupy_co;
    // set pending co to ocupy thest stack mem;
    pending_co->stack_mem->ocupy_co = pending_co;

    env->ocupy_co = ocupy_co;
    if (ocupy_co && ocupy_co != pending_co) {
      save_stack_buffer(ocupy_co);  // 将运行栈保存到自己的 save_buffer
    }
  }

  // swap context
  coctx_swap(&(curr->ctx), &(pending_co->ctx));

  // stack buffer may be overwrite, so get again;
  stCoRoutineEnv_t *curr_env = co_get_curr_thread_env();
  stCoRoutine_t *update_ocupy_co = curr_env->ocupy_co;
  stCoRoutine_t *update_pending_co = curr_env->pending_co;

  if (update_ocupy_co && update_pending_co &&
      update_ocupy_co != update_pending_co) {
    // resume stack buffer
    if (update_pending_co->save_buffer && update_pending_co->save_size > 0) {
      // 从 save_buffer 恢复运行栈到共享栈
      memcpy(update_pending_co->stack_sp, update_pending_co->save_buffer,
             update_pending_co->save_size);
    }
  }
}
```

# 图例

私有栈图示：

<img src="/assets/images/libco-share-stack/illustration-1.png" width="800" />

共享栈图示：

<img src="/assets/images/libco-share-stack/illustration-2.png" width="800" />

# 共享栈模式隐藏的坑

由于共享栈模式下协程的运行栈共用了地址空间，协程切换时，之前协程在运行栈上分配的局部变量的地址不再有效。看看下面的例子：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/time.h>
#include <sys/epoll.h>
#include <errno.h>
#include <string.h>
#include "coctx.h"
#include "co_routine.h"
#include "co_routine_inner.h"

int* global_val = NULL;

void* RoutineFuncC(void* args) {
  return NULL;
}

void* RoutineFuncB(void* args) {
  int routine_id = 2; 
  printf("global_val %d\n", *global_val);
  global_val = &routine_id;

  stCoRoutine_t* co = 0;
  co_create(&co, (stCoRoutineAttr_t*)args, RoutineFuncC, args);
  co_resume(co);
  return NULL;
}

void* RoutineFuncA(void* args) {
  int routine_id = 1;
  global_val = &routine_id;
  printf("global_val %d\n", *global_val);
  
  stCoRoutine_t* co = 0;
  co_create(&co, (stCoRoutineAttr_t*)args, RoutineFuncB, args);
  co_resume(co);
  return NULL;
}

int main() {
  stShareStack_t* share_stack = co_alloc_sharestack(1, 1024 * 128);
  stCoRoutineAttr_t attr;
  attr.stack_size = 0;
  attr.share_stack = share_stack;

  stCoRoutine_t* co = 0;
  co_create(&co, &attr, RoutineFuncA, &attr);
  co_resume(co);

  co_eventloop(co_get_epoll_ct(), NULL, NULL);
  return 0;
}
```

执行，观察输出：

```
./example_copystack 
global_val 1
global_val 2
```

发现这里 global_val 指向的变量的值被篡改了。我们 gdb 看看汇编：

```
(gdb) disassemble RoutineFuncA
Dump of assembler code for function RoutineFuncA(void*):
   0x0000000000400ea5 <+0>:     push   %rbp
   0x0000000000400ea6 <+1>:     mov    %rsp,%rbp
   0x0000000000400ea9 <+4>:     push   %rbx
   0x0000000000400eaa <+5>:     sub    $0x28,%rsp
   0x0000000000400eae <+9>:     mov    %rdi,-0x28(%rbp)
   0x0000000000400eb2 <+13>:    movl   $0x1,-0x14(%rbp)
   0x0000000000400eb9 <+20>:    mov    0x203c70(%rip),%rax        # 0x604b30
   0x0000000000400ec0 <+27>:    lea    -0x14(%rbp),%rdx
   0x0000000000400ec4 <+31>:    mov    %rdx,(%rax)
   0x0000000000400ec7 <+34>:    mov    0x203c62(%rip),%rax        # 0x604b30
   0x0000000000400ece <+41>:    mov    (%rax),%rax
   0x0000000000400ed1 <+44>:    mov    (%rax),%eax
   0x0000000000400ed3 <+46>:    mov    %eax,%esi
   0x0000000000400ed5 <+48>:    lea    0x2b54(%rip),%rdi        # 0x403a30 <__dso_handle+8>
   0x0000000000400edc <+55>:    mov    $0x0,%eax
   0x0000000000400ee1 <+60>:    callq  0x400c10 <printf@plt>
   0x0000000000400ee6 <+65>:    movq   $0x0,-0x20(%rbp)
   0x0000000000400eee <+73>:    mov    -0x28(%rbp),%rbx
   0x0000000000400ef2 <+77>:    mov    -0x28(%rbp),%rdx
   0x0000000000400ef6 <+81>:    lea    -0x20(%rbp),%rax
   0x0000000000400efa <+85>:    mov    %rdx,%rcx
   0x0000000000400efd <+88>:    mov    0x203c4c(%rip),%rdx        # 0x604b50
   0x0000000000400f04 <+95>:    mov    %rbx,%rsi
   0x0000000000400f07 <+98>:    mov    %rax,%rdi
   0x0000000000400f0a <+101>:   callq  0x4016bf <co_create(stCoRoutine_t**, stCoRoutineAttr_t const*, pfn_co_routine_t, void*)>
   0x0000000000400f0f <+106>:   mov    -0x20(%rbp),%rax
   0x0000000000400f13 <+110>:   mov    %rax,%rdi
   0x0000000000400f16 <+113>:   callq  0x401790 <co_resume(stCoRoutine_t*)>
   0x0000000000400f1b <+118>:   mov    $0x0,%eax
   0x0000000000400f20 <+123>:   add    $0x28,%rsp
   0x0000000000400f24 <+127>:   pop    %rbx
   0x0000000000400f25 <+128>:   leaveq 
   0x0000000000400f26 <+129>:   retq   
End of assembler dump.
```

从：
```
   0x0000000000400eb2 <+13>:    movl   $0x1,-0x14(%rbp)
   0x0000000000400eb9 <+20>:    mov    0x203c70(%rip),%rax        # 0x604b30
   0x0000000000400ec0 <+27>:    lea    -0x14(%rbp),%rdx
   0x0000000000400ec4 <+31>:    mov    %rdx,(%rax)
```

可以看出 ```RoutineFuncA``` 中的 ```routine_id``` 变量存放于 ```-0x14(%rbp)```，```global_val``` 存的是 ```routine_id``` 变量的地址，gdb 调试下：


```
(gdb) tbreak *0x0000000000400ea5
Temporary breakpoint 1 at 0x400ea5: file example_copystack.cpp, line 31.
(gdb) r
(gdb) i r rbp rsp
rbp            0x7b5070 0x7b5070
rsp            0x7b5048 0x7b5048
(gdb) ni
0x0000000000400ea6      31      void* RoutineFuncA(void* args) {
(gdb) i r rbp rsp
rbp            0x7b5070 0x7b5070
rsp            0x7b5040 0x7b5040
(gdb) x /2wx 0x7b5040
0x7b5040:       0x007b5070      0x00000000
(gdb) tbreak * 0x0000000000400ec4
Temporary breakpoint 2 at 0x400ec4: file example_copystack.cpp, line 33.
(gdb) c
Continuing.

Temporary breakpoint 2, 0x0000000000400ec4 in RoutineFuncA (args=0x7fffffffe1d0) at example_copystack.cpp:33
33        global_val = &routine_id;
(gdb) i r rdx
rdx            0x7b502c 8081452
```

从 
```
   0x0000000000400ec0 <+27>:    lea    -0x14(%rbp),%rdx
   0x0000000000400ec4 <+31>:    mov    %rdx,(%rax)
   0x0000000000400ec7 <+34>:    mov    0x203c62(%rip),%rax        # 0x604b30
```

可以看到 ```global_val``` 存放的内容为 0x7b502c 也即 ```routine_id``` 变量的地址。

下面看看 RoutineFuncB 的汇编：

```
(gdb) disassemble RoutineFuncB
Dump of assembler code for function RoutineFuncB(void*):
   0x0000000000400e23 <+0>:     push   %rbp
   0x0000000000400e24 <+1>:     mov    %rsp,%rbp
   0x0000000000400e27 <+4>:     push   %rbx
   0x0000000000400e28 <+5>:     sub    $0x28,%rsp
   0x0000000000400e2c <+9>:     mov    %rdi,-0x28(%rbp)
   0x0000000000400e30 <+13>:    movl   $0x2,-0x14(%rbp)
   0x0000000000400e37 <+20>:    mov    0x203cf2(%rip),%rax        # 0x604b30
   0x0000000000400e3e <+27>:    mov    (%rax),%rax
   0x0000000000400e41 <+30>:    mov    (%rax),%eax
   0x0000000000400e43 <+32>:    mov    %eax,%esi
   0x0000000000400e45 <+34>:    lea    0x2be4(%rip),%rdi        # 0x403a30 <__dso_handle+8>
   0x0000000000400e4c <+41>:    mov    $0x0,%eax
   0x0000000000400e51 <+46>:    callq  0x400c10 <printf@plt>
   0x0000000000400e56 <+51>:    mov    0x203cd3(%rip),%rax        # 0x604b30
   0x0000000000400e5d <+58>:    lea    -0x14(%rbp),%rdx
   0x0000000000400e61 <+62>:    mov    %rdx,(%rax)
   0x0000000000400e64 <+65>:    movq   $0x0,-0x20(%rbp)
   0x0000000000400e6c <+73>:    mov    -0x28(%rbp),%rbx
   0x0000000000400e70 <+77>:    mov    -0x28(%rbp),%rdx
   0x0000000000400e74 <+81>:    lea    -0x20(%rbp),%rax
   0x0000000000400e78 <+85>:    mov    %rdx,%rcx
   0x0000000000400e7b <+88>:    mov    0x203ce6(%rip),%rdx        # 0x604b68
   0x0000000000400e82 <+95>:    mov    %rbx,%rsi
   0x0000000000400e85 <+98>:    mov    %rax,%rdi
   0x0000000000400e88 <+101>:   callq  0x4016bf <co_create(stCoRoutine_t**, stCoRoutineAttr_t const*, pfn_co_routine_t, void*)>
   0x0000000000400e8d <+106>:   mov    -0x20(%rbp),%rax
   0x0000000000400e91 <+110>:   mov    %rax,%rdi
   0x0000000000400e94 <+113>:   callq  0x401790 <co_resume(stCoRoutine_t*)>
   0x0000000000400e99 <+118>:   mov    $0x0,%eax
   0x0000000000400e9e <+123>:   add    $0x28,%rsp
   0x0000000000400ea2 <+127>:   pop    %rbx
   0x0000000000400ea3 <+128>:   leaveq 
   0x0000000000400ea4 <+129>:   retq   
End of assembler dump.
```


同样可以看出 ```RoutineFuncB``` 中的 ```routine_id``` 变量也存放于 ```-0x14(%rbp)```，gdb 下：

```
(gdb) tbreak *0x0000000000400e23
Temporary breakpoint 3 at 0x400e23: file example_copystack.cpp, line 20.
(gdb) c
Continuing.
global_val 1

Temporary breakpoint 3, RoutineFuncB (args=0x7fffffffe1d0) at example_copystack.cpp:20
(gdb) i r rbp rsp  // 注意到 RoutineFuncB 的 rbp 和 RoutineFuncA 的 rbp 相同
rbp            0x7b5070 0x7b5070
rsp            0x7b5048 0x7b5048
(gdb) tbreak *0x0000000000400e37
Temporary breakpoint 4 at 0x400e37: file example_copystack.cpp, line 22.
(gdb) c
Continuing.

Temporary breakpoint 4, RoutineFuncB (args=0x7fffffffe1d0) at example_copystack.cpp:22
22        printf("global_val %d\n", *global_val);
(gdb) x /2wx $rbp-0x14   // 0x7b502c 指向的值被修改为 2
0x7b502c:       0x00000002      0x00000000
(gdb) tbreak *0x0000000000400e3e
Temporary breakpoint 5 at 0x400e3e: file example_copystack.cpp, line 22.
(gdb) c
Continuing.

Temporary breakpoint 5, 0x0000000000400e3e in RoutineFuncB (args=0x7fffffffe1d0) at example_copystack.cpp:22
22        printf("global_val %d\n", *global_val);
(gdb) i r rax
rax            0x604db0 6311344
(gdb) x /2wx 0x604db0  // global_val 指向的地址为 0x7b502c
0x604db0 <global_val>:  0x007b502c      0x00000000
```

可以看出，**因为使用了共享栈，每个协程的 rbp 一致**（stack_buffer 是同一块 malloc 的内存，ss_sp 指向 stack_buffer，因此每个协程的 ss_sp 值相同，进而 sp 也相同），导致不同协程内分配局部变量的地址会冲突（私有栈则没有这个问题，因为每个协程的栈空间是各自 malloc 的）。

在 RoutineFuncA 中定义的局部变量的地址为 0x007b502c，此地址保存在全局变量 global_val 中。之后启动了 RoutineFuncB，定义了另外的局部变量，地址刚好也为 0x007b502c，则全局变量 global_val 指向的变量实际上是 RoutineFuncB 中的临时变量了，所以打印出的值也就是 RoutineFuncB 中局部变量的值。**因此，在使用共享栈模式时，切记不要在协程之间传递局部变量的地址。**


