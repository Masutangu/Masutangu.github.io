---
layout: post
date: 2024-06-03T22:40:19+08:00
title: 【PyTorch 源码阅读】intrusive_ptr 篇
tags: 
  - 源码阅读
---

# 代码路径

[https://github.com/pytorch/pytorch/blob/main/c10/util/intrusive_ptr.h](https://github.com/pytorch/pytorch/blob/main/c10/util/intrusive_ptr.h)

# 定义

根据注释：

```
/**
 * intrusive_ptr<T> is an alternative to shared_ptr<T> that has better
 * performance because it does the refcounting intrusively
 * (i.e. in a member of the object itself).
 * Your class T needs to inherit from intrusive_ptr_target to allow it to be
 * used in an intrusive_ptr<T>. Your class's constructor should not allow
 *`this` to escape to other threads or create an intrusive_ptr from `this`.
 */
```

可以将 intrusive_ptr 定义为性能更好的 shared_ptr。intrusive_ptr 通过侵入式的方式来实现引用计数，类似于 boost 的 intrusive_ptr。关于 shared_ptr 和 boost intrusive_ptr 的对比，可以阅读 [**Think Intrusive, Gain Performance**](https://www.linkedin.com/pulse/think-intrusive-gain-performance-prakash-nirala) 。

boost intrusive_ptr 存在以下不足：

- 代码冗余，每个使用 intrusive_ptr  的类都需定义：
    - `inline void intrusive_ptr_add_ref(T* t)`
    - `inline void intrusive_ptr_release(T* t)`
    - `long references`
- 不支持 [weak_ptr](https://stackoverflow.com/questions/2400458/is-there-a-boostweak-intrusive-pointer)，无法解决循环引用的问题

为了减少代码冗余，pytorch 定义了基类 intrusive_ptr_target，位于 [intrusive_ptr.h](https://github.com/pytorch/pytorch/blob/main/c10/util/intrusive_ptr.h) 文件中：

```cpp
class intrusive_ptr_target {
  // Here's the scheme:
  //
  //  - refcount == number of strong references to the object
  //    weakcount == number of weak references to the object,
  //      plus one more if refcount > 0
  //    An invariant: refcount > 0  =>  weakcount > 0
  //
  //
  //  - finalizers are called and data_ptr is deallocated when refcount == 0
  //
  //  - Once refcount == 0, it can never again be > 0 (the transition
  //    from > 0 to == 0 is monotonic)
  mutable std::atomic<size_t> refcount_;
  mutable std::atomic<size_t> weakcount_; 
  
  template <typename T, typename NullType>
  friend class intrusive_ptr;
  
  
  template <typename T, typename NullType>
  friend class weak_intrusive_ptr;
}
```

需要使用 intrusive_ptr 的类只需要继承自 intrusive_ptr_target 即可，且支持 weak_ptr（成员变量 weakcount_）。

# 源码阅读

## intrusive_ptr

重点关注 `retain_()` 和 `reset_()` 的调用时机：

```cpp
template <
    class TTarget,
    class NullType = detail::intrusive_target_default_null_type<TTarget>>
class intrusive_ptr final {
  private:
    TTarget* target_;

    void retain_() {
        if (target_ != NullType::singleton()) {
            detail::atomic_refcount_increment(target_->refcount_);
        }
    }

    void reset_() noexcept {
        if (target_ != NullType::singleton() &&
            detail::atomic_refcount_decrement(target_->refcount_) == 0) {
            bool should_delete =
                target_->weakcount_.load(std::memory_order_acquire) == 1;
            if (!should_delete) {
                const_cast<std::remove_const_t<TTarget>*>(target_)->release_resources();
                should_delete = detail::atomic_weakcount_decrement(target_->weakcount_) == 0;
            }
            if (should_delete) {
                delete target_;
            }
        }
    }
};
```

## weak_intrusive_ptr

重点关注 `retain_()` 和 `reset_()` 的调用时机：

```cpp
template <
    typename TTarget,
    class NullType = detail::intrusive_target_default_null_type<TTarget>>
class weak_intrusive_ptr final {
  private:
    TTarget* target_;
	  
    void retain_() {
        if (target_ != NullType::singleton()) {
            detail::atomic_weakcount_increment(target_->weakcount_);
        }
    }
    
    void reset_() noexcept {
        if (target_ != NullType::singleton() &&
            detail::atomic_weakcount_decrement(target_->weakcount_) == 0) {
            delete target_;
        }
        target_ = NullType::singleton();
    }
};
```

## **内存顺序**

C++ 中有六种内存顺序模型：

| Value | Explanation |
| --- | --- |
| memory_order_relaxed | Relaxed operation: there are no synchronization or ordering constraints imposed on other reads or writes, only this operation's atomicity is guaranteed (see https://en.cppreference.com/w/cpp/atomic/memory_order#Relaxed_ordering below). |
| memory_order_consume | A load operation with this memory order performs a consume operation on the affected memory location: no reads or writes in the current thread dependent on the value currently loaded can be reordered before this load. Writes to data-dependent variables in other threads that release the same atomic variable are visible in the current thread. On most platforms, this affects compiler optimizations only (see https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Consume_ordering below). |
| memory_order_acquire | A load operation with this memory order performs the acquire operation on the affected memory location: no reads or writes in the current thread can be reordered before this load. All writes in other threads that release the same atomic variable are visible in the current thread (see https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Acquire_ordering below). |
| memory_order_release | A store operation with this memory order performs the release operation: no reads or writes in the current thread can be reordered after this store. All writes in the current thread are visible in other threads that acquire the same atomic variable (see https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Acquire_ordering below) and writes that carry a dependency into the atomic variable become visible in other threads that consume the same atomic (see https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Consume_ordering below). |
| memory_order_acq_rel | A read-modify-write operation with this memory order is both an acquire operation and a release operation. No memory reads or writes in the current thread can be reordered before the load, nor after the store. All writes in other threads that release the same atomic variable are visible before the modification and the modification is visible in other threads that acquire the same atomic variable. |
| memory_order_seq_cst | A load operation with this memory order performs an acquire operation, a store performs a release operation, and read-modify-write performs both an acquire operation and a release operation, plus a single total order exists in which all threads observe all modifications in the same order (see https://en.cppreference.com/w/cpp/atomic/memory_order#Sequentially-consistent_ordering below). |

详细可以查阅文档 [memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order)，文档中提到：

> The specification of release-consume ordering is being revised, and the use of `memory_order_consume` is temporarily discouraged.
> 

所以关于 consume 的部分直接跳过即可。

### increment/decrement 操作

从注释可以看出，refcount/weakcount 的 decrement 操作需要 `memory_order_acq_rel`，因为调用 destructor 需要准确。refcount 的 increment 操作需要 `memory_order_acq_rel`，因为调用 use_count/unique 需要准确的结果。而 weak_use_count 只用于 testing，因此 weakcount 的 increment 操作只需要 `memory_order_relaxed`：

```cpp
// Increment needs to be acquire-release to make use_count() and
// unique() reliable.
inline size_t atomic_refcount_increment(std::atomic<size_t>& refcount) {
  return refcount.fetch_add(1, std::memory_order_acq_rel) + 1;
}

// weak_use_count() is only used for testing, so we don't need it to
// be reliable. Relaxed should be fine.
inline size_t atomic_weakcount_increment(std::atomic<size_t>& weakcount) {
  return weakcount.fetch_add(1, std::memory_order_relaxed) + 1;
}

// Both decrements need to be acquire-release for correctness. See
// e.g. std::shared_ptr implementation.
inline size_t atomic_refcount_decrement(std::atomic<size_t>& refcount) {
  return refcount.fetch_sub(1, std::memory_order_acq_rel) - 1;
}

inline size_t atomic_weakcount_decrement(std::atomic<size_t>& weakcount) {
  return weakcount.fetch_sub(1, std::memory_order_acq_rel) - 1;
}
```

### lock 操作

weak_intrusive_ptr 的 lock 函数用于将 weak_intrusive_ptr 提升为 intrusive_ptr：

```cpp
intrusive_ptr<TTarget, NullType> lock() const noexcept {
  if (expired()) {
    return intrusive_ptr<TTarget, NullType>();
  } else {
    auto refcount = target_->refcount_.load(std::memory_order_seq_cst);
    do {
      if (refcount == 0) {
        // Object already destructed, no strong references left anymore.
        // Return nullptr.
        return intrusive_ptr<TTarget, NullType>();
      }
    } while (
        !target_->refcount_.compare_exchange_weak(refcount, refcount + 1));
    return intrusive_ptr<TTarget, NullType>(
        target_, raw::DontIncreaseRefcount{});
  }
}
```

这里的实现和 [gcc/libstdc++](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/shared_ptr_base.h#L277) 的实现类似。

`compare_exchange_weak` 有可能 fail spuriously，在 loop 场景下使用 `compare_exchange_weak` 会比 `compare_exchange_strong` 的性能更佳。

关于 `compare_exchange_weak`  和  `compare_exchange_strong`  的比较，可以参考 [**Understanding std::atomic::compare_exchange_weak() in C++11**](https://stackoverflow.com/questions/25199838/understanding-stdatomiccompare-exchange-weak-in-c11)。

`compare_exchange_weak` 原子性地比较 `*this` 和 `expected` 的 [object representation](https://en.cppreference.com/w/cpp/language/object)(until C++20)/[value representation](https://en.cppreference.com/w/cpp/language/object)(since C++20) . 如果是 bitwise 相等, 执行 **read-modify-write operation** 替换为 `desired` . 否则，执行 **load operation**  加载 `*this` 到 `expected`。read-modify-write operation 和 load operation 所采用的内存模型如下：

| Overloads | read‑modify‑write operation | load operation |
| --- | --- | --- |
| (1,2,5,6) | success | failure |
| (3,4,7,8) | order | 1. std::memory_order_acquire if order is std::memory_order_acq_rel 2. std::memory_order_relaxed if order is std::memory_order_release 3. otherwise order |

具体可以查阅文档 [compare_exchange](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange)，文档中提供了一个 lock-free 的例子：

```cpp
template<typename T>
struct node
{
    T data;
    node* next;
    node(const T& data) : data(data), next(nullptr) {}
};
 
template<typename T>
class stack
{
    std::atomic<node<T>*> head;
public:
    void push(const T& data)
    {
        node<T>* new_node = new node<T>(data);
 
        // put the current value of head into new_node->next
        new_node->next = head.load(std::memory_order_relaxed);
 
        // now make new_node the new head, but if the head
        // is no longer what's stored in new_node->next
        // (some other thread must have inserted a node just now)
        // then put that new head into new_node->next and try again
        while (!head.compare_exchange_weak(new_node->next, new_node,
                                           std::memory_order_release,
                                           std::memory_order_relaxed))
            ; // the body of the loop is empty
 
// Note: the above use is not thread-safe in at least 
// GCC prior to 4.8.3 (bug 60272), clang prior to 2014-05-05 (bug 18899)
// MSVC prior to 2014-03-17 (bug 819819). The following is a workaround:
//      node<T>* old_head = head.load(std::memory_order_relaxed);
//      do
//      {
//          new_node->next = old_head;
//      }
//      while (!head.compare_exchange_weak(old_head, new_node,
//                                         std::memory_order_release,
//                                         std::memory_order_relaxed));
    }
};
```

注释提到这里的实现非线程安全，查阅了下 [stackoverflow](https://stackoverflow.com/questions/21879331/is-stdatomic-compare-exchange-weak-thread-unsafe-by-design) 有人给出了解答：

代码示例：

```cpp
   #include <atomic>
   struct Node { Node* next; };
   void Push(std::atomic<Node*> head, Node* node)
   {
       node->next = head.load();
       while(!head.compare_exchange_weak(node->next, node))
           ;
   }
```

g++ 4.8 [assembler]

```cpp
       mov    rdx, rdi                       // （1）存储 head 到 rdx
       mov    rax, QWORD PTR [rdi]           // （2）存储 head.load() 到 rax
       mov    QWORD PTR [rsi], rax           // （3）存储 %rax（即 head.load()）到 [rsi]（node->next） 
   .L3:
       mov    rax, QWORD PTR [rsi]           // （4）存储 [rsi]（node->next） 到 %rax
       lock cmpxchg    QWORD PTR [rdx], rsi  // （5）原子执行 cmpxchg，比较 %rax（node->next）和 rdx（head 指向的地址） ，相等则将 %rsi（node）写入到 rdx（head），不相等则加载 [rdx]（head） 到 rax 中
       mov    QWORD PTR [rsi], rax           // （6）存储 rax（head）到 [rsi]（node->next）
       jne    .L3                            // （7）如果 ZF 为 0（cmpxchg 比较不相等），则跳转到 L3
       rep; ret

```

（5）和（6）的本意是实现 compare_exchange_weak 的语义，即：

```cpp
if (*this == expected)
  *this = desired;    
else                   
  expected = *this;    
```

但实际上无论 `*this` 是否等于 `expected`，对 `expected` 的写操作（汇编语句（6））都会执行，即都会写 `node->next`。考虑这个场景，线程 A 在指令（5）比较成功，此时 `node->next` 已经成功 push 到 stack 中，对其他线程可见，此时线程 A 开始执行指令（6），如果有另一线程 B 开始读取 `node->next`，就会发生数据竞争。

***思考**：weak_intrusive_ptr lock() 的实现为什么需要采用 `memory_order_seq_cst`，看到也有人提问过类似的问题：[https://stackoverflow.com/questions/70331658/how-to-implement-stdweak-ptrlock-with-atomic-operations](https://stackoverflow.com/questions/70331658/how-to-implement-stdweak-ptrlock-with-atomic-operations)。load 和  compare_exchange_weak 使用 `memory_order_relaxed` 或 `memory_order_acq_rel` 是否也 ok？*


## Tagged dispatch mechanism

intrusive_ptr 在实现中使用了 tagged dispatch mechanism，例如：

```cpp
// This constructor will not increase the ref counter for you.
// We use the tagged dispatch mechanism to explicitly mark this constructor
// to not increase the refcount
explicit intrusive_ptr(TTarget* target, raw::DontIncreaseRefcount) noexcept
    : target_(target) {}
```

关于 tagged dispatch，以下来自 GPT：

> Tag dispatching 是一种利用函数重载来根据类型的属性进行函数调度的技术，通常与 traits classes 配合使用。其思想是定义多个具有不同参数类型（标签）的重载函数，这些标签代表了类型的不同属性或特性。通过选择适当的标签或特性，编译器可以将函数调用解析到正确的重载函数上。这种技术在你想为特定类型或特性提供专门的行为或优化时非常有用，而无需使用运行时多态或条件分支。它允许你以更高效和简洁的方式处理不同的情况，通过利用编译器根据涉及的类型或特性选择适当的函数。
> 

Tag dispatching 与 traits class 之间的关系在于，tag dispatching 的属性通常通过  traits class 来访问。以 `advance()` 函数为例，通过 [`iterator_traits`](http://en.cppreference.com/w/cpp/iterator/iterator_traits) 拿到对应的 iterator_category 去调用重载的 advance_dispatch() 函数，具体可以阅读 [**Generic Programming Techniques**](https://www.boost.org/community/generic_programming.html)。
