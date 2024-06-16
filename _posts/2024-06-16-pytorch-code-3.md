---
layout: post
date: 2024-06-16T20:29:46+08:00
title: 【PyTorch 源码阅读】DispatchKeySet 源码篇
tags: 
  - 源码阅读
---

# 代码路径

[https://github.com/pytorch/pytorch/blob/main/c10/core/DispatchKey.h](https://github.com/pytorch/pytorch/blob/main/c10/core/DispatchKey.h)
[https://github.com/pytorch/pytorch/blob/main/c10/core/DispatchKeySet.h](https://github.com/pytorch/pytorch/blob/main/c10/core/DispatchKeySet.h)

# 定义

## BackendComponent

根据注释：

```cpp
// Semantically, each value of BackendComponent identifies a "backend" for our
// dispatch. Some functionalities that we may dispatch to are allowed to
// register different handlers for each backend. The BackendComponent is then
// used to figure out which backend implementation to dispatch to.

// In implementation terms, the backend component identifies a specific "bit" in
// a DispatchKeySet. The bits in the DispatchKeySet are split between the bottom
// ~12 "BackendComponent" bits, while the remaining upper bits are assigned to
// functionalities. When we encounter a functionality bit that is known to be
// customizeable per-backend, then we also look at the lower BackendComponent
// bits and take the highest bit to determine which backend's implementation to
// use.
```

每个 BackendComponent 的值标识了一个调度的后端，**有些功能（functionalities）允许为不同的后端注册不同的处理程序，BackendComponent 用于确定要调度到哪个后端实现**。这样，我们可以根据需要选择适当的后端处理程序来执行特定的功能。

实现上，BackendComponent 由 DispatchKeySet 中一个特定的 bit 位来标记。**DispatchKeySet 中的位可分为 BackendComponent 位和功能位**。当遇到后端自定义的功能位时，会查看 BackendComponent 位，并选择其中最高位来确定使用哪个后端的实现。具体见 [源码阅读-DispatchKeySet](http://masutangu.com/2024/06/16/pytorch-code-3/#dispatchkeyset-1) 小节中的 highestPriorityTypeId 函数。

PyTorch 使用宏来生成 BackendComponent 的枚举值：

```cpp
#define C10_FORALL_BACKEND_COMPONENTS(_, extra) \
  _(CPU, extra)                                 \
  _(CUDA, extra)                                \
  _(HIP, extra)                                 \
  _(XLA, extra)                                 \
  _(MPS, extra)                                 \
  _(IPU, extra)                                 \
  _(XPU, extra)                                 \
  _(HPU, extra)                                 \
  _(VE, extra)                                  \
  _(Lazy, extra)                                \
  _(MTIA, extra)                                \
  _(PrivateUse1, extra)                         \
  _(PrivateUse2, extra)                         \
  _(PrivateUse3, extra)                         \
  _(Meta, extra)

// WARNING!  If we add a new per-backend functionality key that has higher
// priority than Autograd, then make sure you update EndOfRuntimeBackendKeys

#define C10_FORALL_FUNCTIONALITY_KEYS(_) \
  _(Dense, )                             \
  _(Quantized, Quantized)                \
  _(Sparse, Sparse)                      \
  _(NestedTensor, NestedTensor)          \
  _(AutogradFunctionality, Autograd)

enum class BackendComponent : uint8_t {

  // A "backend" is colloquially used to refer to handlers for dispatch
  // which actually implement the numerics of an operation in question.
  //
  // Due to the nature of the enum, these backends are specified in
  // an ordered way, but for most backends this order is not semantically
  // meaningful (e.g., it's valid to reorder these backends without changing
  // semantics).  The only situation when backend ordering is meaningful
  // is when the backend participates in multiple dispatch with another
  // backend; e.g., CPU and CUDA (cuda must have higher priority).

  // These keys don't correspond to individual kernels.
  // Instead, they represent the backends that are allowed to override specific
  // pieces of functionality:
  // - dense kernels (e.g. DispatchKey::CPU)
  // - sparse kernels (e.g. DispatchKey::SparseCPU)
  // - quantized kernels (e.g. DispatchKey::QuantizedCPU)
  // - autograd kernels (e.g. DispatchKey::AutogradCPU)
  // We reserve space in the runtime operator table for this full cross product
  // of
  // [backends in this enum] x [keys below that are explicitly marked as having
  // per-backend functionality]

  InvalidBit = 0,
#define DEFINE_BACKEND_COMPONENT(n, _) n##Bit,
  C10_FORALL_BACKEND_COMPONENTS(DEFINE_BACKEND_COMPONENT, unused)
#undef DEFINE_BACKEND_COMPONENT

  // Define an alias to represent end of backend dispatch keys.
  // If you add new backend keys after PrivateUse3, please also update it here.
  EndOfBackendKeys = MetaBit,
};
```

宏 `C10_FORALL_BACKEND_COMPONENTS` 接受两个参数：`_` 和 `extra`。传入`DEFINE_BACKEND_COMPONENT` 和 `unused` 时，参数 `_` 为 `DEFINE_BACKEND_COMPONENT`，参数 `extra` 为 `unused`。执行 `_(CPU, extra)`，即对 `DEFINE_BACKEND_COMPONENT(n, _)` 进行宏展开，其中 `n` 为 `CPU`，`_` 为 `unused`，展开结果为 `CPU##Bit` 即 `CPUBit`。

再次强调，BackendComponent 定义的这些键并不对应于单个内核。它们代表允许覆盖特定功能的后端，例如：
- 密集内核（例如 DispatchKey::CPU）
- 稀疏内核（例如 DispatchKey::SparseCPU）
- 量化内核（例如 DispatchKey::QuantizedCPU）
- 自动微分内核（例如 DispatchKey::AutogradCPU）

## DispatchKey

根据注释：
```
// Semantically, a dispatch key identifies a possible "level" in our
// dispatch, for which a handler may be registered. Each handler corresponds
// to a type of functionality.
//
// In implementation terms, the dispatch key identifies a specific "bit" in a
// DispatchKeySet.  Higher bit indexes get handled by dispatching first (because
// we "count leading zeros" when we extract the highest priority dispatch
// key.)
//
// Note [DispatchKey Classification]
// This enum actually contains several types of keys, which are explained
// in more detail further down:
// (1) non-customizable backends (e.g. FPGA)
// (2) non-customizable functionalities (e.g. Functionalize)
// (3) functionalized that are customizable per backend (e.g. Dense, Sparse,
// AutogradFunctionality) (4) per-backend instances of customizable
// functionalities (e.g. CPU, SparseCPU, AutogradCPU) (5) alias keys (e.g.
// CompositeImplicitAutograd)
//
// Of the categories above, it's important to note:
// (a) which keys are assigned individual bits in a DispatchKeySet
// (b) which keys are assigned individual slots in the runtime operator table
// ("Runtime keys")
//
// (1), (2) and (3) all get their own dedicated bits in the DispatchKeySet.
// (1), (2) and (4) all get their own dedicated slots in the runtime operator
// table.
```

可以为调度键注册相应的 handler，每个 handler 对应一种特定类型的功能。
实现上，**调度键标识了 DispatchKeySet 中某一特定的 "位"，较高的位优先调度。**

调度键可分为以下几种类别：
1. 不可自定义的后端（例如 FPGA）
2. 不可自定义的功能（例如 Functionalize）
3. 可以根据后端自定义的功能（例如 Dense、Sparse、AutogradFunctionality）
4. 可自定义功能的后端实例（例如 CPU、SparseCPU、AutogradCPU）
5. 别名键（例如 CompositeImplicitAutograd）

在上述类别中：
- **1、2 和 3 在 DispatchKeySet 中有自己的专用位**
- **1、2 和 4 在运行时操作符表（runtime operator table）中有自己的专用 slot**

类似的，PyTorch 使用宏来生成 DispatchKey 的枚举值：

```cpp
#define C10_FORALL_FUNCTIONALITY_KEYS(_) \
  _(Dense, )                             \
  _(Quantized, Quantized)                \
  _(Sparse, Sparse)                      \
  _(NestedTensor, NestedTensor)          \
  _(AutogradFunctionality, Autograd)

enum class DispatchKey : uint16_t {
  // ~~~~~~~~~~~~~~~~~~~~~~~~~~ UNDEFINED ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ //
  // This is not a "real" functionality, but it exists to give us a "nullopt"
  // element we can return for cases when a DispatchKeySet contains no elements.
  // You can think a more semantically accurate definition of DispatchKey is:
  //
  //    using DispatchKey = optional<RealDispatchKey>
  //
  // and Undefined == nullopt.  We didn't actually represent
  // it this way because optional<RealDispatchKey> would take two
  // words, when DispatchKey fits in eight bits.

  Undefined = 0,
  // ~~~~~~~~~~~~~~~~~~~~~~~~~~ Functionality Keys ~~~~~~~~~~~~~~~~~~~~~~ //
  // Every value in the enum (up to EndOfFunctionalityKeys)
  // corresponds to an individual "functionality" that can be dispatched to.
  // This is represented in the DispatchKeySet by assigning each of these enum
  // values to each of the remaining (64 - len(BackendComponent)) bits.
  //
  // Most of these functionalities have a single handler assigned to them,
  // making them "runtime keys".
  // That map to a single slot in the runtime operator table.
  //
  // A few functionalities are allowed to be customizable per backend.
  // See [Note: Per-Backend Functionality Dispatch Keys] for details.

  // See [Note: Per-Backend Functionality Dispatch Keys]
  Dense,
  // Below are non-extensible backends.
  // These are backends that currently don't have their own overrides for
  // Autograd/Sparse/Quantized kernels,
  // and we therefore don't waste space in the runtime operator table allocating
  // space for them.
  // If any of these backends ever need to customize, e.g., Autograd, then we'll
  // need to add a DispatchKey::*Bit for them.
  FPGA,
  ...
  // See [Note: Per-Backend Functionality Dispatch Keys]
  Quantized,
  // See [Note: Per-Backend Functionality Dispatch Keys]
  Sparse,
  AutogradFunctionality,
  ...
  EndOfFunctionalityKeys,  // End of functionality keys.
  
// ~~~~~~~~~~~~~~ "Dense" Per-Backend Dispatch keys ~~~~~~~~~~~~~~~~~~~~ //
// Here are backends which you think of as traditionally specifying
// how to implement operations on some device.

#define DEFINE_PER_BACKEND_KEYS_FOR_BACKEND(n, prefix) prefix##n,

#define DEFINE_PER_BACKEND_KEYS(fullname, prefix)      \
  StartOf##fullname##Backends,                         \
      C10_FORALL_BACKEND_COMPONENTS(                   \
          DEFINE_PER_BACKEND_KEYS_FOR_BACKEND, prefix) \
  EndOf##fullname##Backends = prefix##Meta,

  C10_FORALL_FUNCTIONALITY_KEYS(DEFINE_PER_BACKEND_KEYS)

#undef DEFINE_PER_BACKEND_KEYS
#undef DEFINE_PER_BACKEND_KEYS_FOR_BACKEND

  EndOfRuntimeBackendKeys = EndOfAutogradFunctionalityBackends,

  // ~~~~~~~~~~~~~~~~~~~~~~ Alias Dispatch Keys ~~~~~~~~~~~~~~~~~~~~~~~~~~ //
  // Note [Alias Dispatch Keys]
  // **Alias dispatch keys are synthetic dispatch keys which map to multiple
  // runtime dispatch keys**. Alisa keys have precedence, but they are always
  // lower precedence than runtime keys. You can register a kernel to an
  // alias key, the kernel might be populated to the mapped runtime keys
  // during dispatch table computation.
  // If a runtime dispatch key has multiple kernels from alias keys, which
  // kernel wins is done based on the precedence of alias keys (but runtime
  // keys always have precedence over alias keys).
  // Alias keys won't be directly called during runtime.
  // Define an alias key to represent end of alias dispatch keys.
  // If you add new alias keys after Autograd, please also update it here.
  Autograd,
  StartOfAliasKeys = Autograd,
  EndOfAliasKeys = Autograd, //
};
```

宏 `C10_FORALL_FUNCTIONALITY_KEYS` 生成的运行时键如下：

```cpp
StartOfDenseBackends,
CPU,
CUDA,
Meta,
EndOfDenseBackends = Meta,
StartOfQuantizedBackends,
QuantizedCPU,
QuantizedCUDA,
QuantizedMeta,
EndOfQuantizedBackends = QuantizedMeta,
StartOfSparseBackends,
SparseCPU,
SparseCUDA,
SparseMeta,
EndOfSparseBackends = SparseMeta,
StartOfNestedTensorBackends,
NestedTensorCPU,
NestedTensorCUDA,
NestedTensorMeta,
EndOfNestedTensorBackends = NestedTensorMeta,
StartOfAutogradFunctionalityBackends,
AutogradCPU,
AutogradCUDA,
AutogradMeta,
EndOfAutogradFunctionalityBackends = AutogradMeta,
```

关于注释中提到的 Note: Per-Backend Functionality Dispatch Keys：
```
// [Note: Per-Backend Functionality Dispatch Keys]
// These keys correspond to functionalities that can be customized indivually
// per backend. While they only take up one bit in the `DispatchKeySet` bitset,
// they map to (# backends) slots in the operator table.
// Each of these keys also has a separate set of "runtime keys" in the dispatch
// key enum, per backend, which *do* map to the individual operator table slots.
// For example, the "Sparse" key maps to an individual bit in the
// DispatchKeySet, while `SparseCPU`, `SparseCUDA`, etc all map to individual
// slots in the runtime operator table.
```

Per-Backend Functionality Dispatch Key 对应于可以由后端单独定制的功能。**虽然这些键在`DispatchKeySet`位集中只占用一个位，但在操作符表中映射到【后端数量】个槽位**。每个键在调度键枚举中有一个单独的“运行时键”集合，每个后端有一个枚举值，映射到操作符表单独的槽位。举例来说，“Sparse“ 键映射到 `DispatchKeySet` 中的一个单独位，而 `SparseCPU`、`SparseCUDA` 等则映射到运行时操作符表中的单独槽位。

## DispatchKeySet

根据注释：
```
// A representation of a set of DispatchKeys. **A DispatchKeySet contains both
// "functionality" bits and "backend bits", and every tensor holds its own
// DispatchKeySet.** **The Dispatcher implements multiple dispatch by grabbing the
// keyset on every input tensor, or’ing them together, and dispatching to a
// specific piece of functionality.** **The functionality bits are *ordered***. When
// **multiple functionality bits are set, we use the highest priority**
// **functionality**. 
```

DispatchKeySet 即调度键的集合。DispatchKeySet 包含了**"功能（functionality）"位**和**"后端（backend）位"**，每个 tensor 都有自己的 DispatchKeySet。Dispatcher 通过获取每个 **input tensor** 的 **DispatchKeySet**，进行逻辑或操作，然后调度到特定的功能。功能性位是**有序**的。当设置了多个功能性位时，使用最高优先级的功能。

```
// Note [DispatchKeySet Internal Representation]
// Internally, dispatch keys are packed into 64-bit DispatchKeySet objects
// that get passed around at runtime.
// However, there isn't necessarily a 1-to-1 mapping between bits in the keyset
// and individual dispatch keys.
//
// First: why do we have this distinction, and why not map every dispatch key
// directly to a bit? This is mostly because we have several types of
// functionalities that different backends would like to customize. For example,
// we have:
// - "Dense":     CPU, CUDA, XLA, ... (~12 keys)
// - "Sparse":    SparseCPU, SparseCUDA, ...
// - "Quantized": QuantizedCPU, QuantizedCUDA, QuantizedXLA, ...
// - "Autograd":  AutogradCPU, AutogradCUDA, Autograd XLA, ...
// The problem is that total number of keys grows quadratically with [#
// backends] x [# functionalities], making it very difficult to map each key
// directly to a bit in a bitset without dramatically increasing the size of the
// bitset over time.
```

**调度键和 DispatchKeySet 中的位并不是一对一的关系**，主要是因为 Per-Backend Functionality Dispatch Key 的存在，如下：
- "Dense": CPU、CUDA、XLA 等
- "Sparse": SparseCPU、SparseCUDA 等
- "Quantized": QuantizedCPU、QuantizedCUDA、QuantizedXLA 等
- "Autograd": AutogradCPU、AutogradCUDA、AutogradXLA 等

调度键的总数将随着【后端数量】 x 【功能数量】呈二次增长，如果直接将每个键映射到 DispatchKeySet 中的一个位，将会大大增加 DispatchKeySet 的大小。

```
// The two enums (BackendComponent and DispatchKey) can be divided roughly into
// 5 categories.
//
// (1) "Building block" keys
//    (a) backends: jEverything in the BackendComponent enum (e.g. CPUBit, 
//        CUDABIt) 
//    (b) functionalities: (per-backend) functionality-bit DispatchKeys
//        (e.g. AutogradFunctionality, Sparse, Dense)
// (2) "Runtime" keys
//    (a) "non-customizable backends" (e.g. FPGA)
//    (b) "non-customizable functionalities" (e.g. Functionalize)
//    (c) "per-backend instances of customizable functionalities" (e.g. CPU,
//        SparseCPU, AutogradCPU)
// (3) "Alias" DispatchKeys (see Note [Alias Dispatch Keys])
```

BackendComponent 和 DispatchKey 枚举大致可以分为 5 个类别：
1. "构建块"键  
a. 后端：BackendComponent 枚举中的所有值（CPUBit、CUDABit 等）  
b. 功能：（每个后端）功能位的 DispatchKeys（例如 AutogradFunctionality、Sparse、Dense）

2. "运行时"键  
a. "不可定制的后端"（例如 FPGA）  
b. "不可定制的功能"（例如 Functionalize）  
c. "可定制功能的每个后端实例"（例如 CPU、SparseCPU、AutogradCPU）  

3. "别名" DispatchKeys（见 Note [Alias Dispatch Keys]）

```
// (1) Building block keys always correspond to individual bits in a
// DispatchKeySet. They can also be combined in a DispatchKeySet to form actual
// runtime keys. e.g.
//     auto dense_cpu_ks = DispatchKeySet({DispatchKey::CPUBit,
//     DispatchKey::Dense});
//     // The keyset has the runtime dense-cpu key.
//     dense_cpu_ks.has(DispatchKey::CPU);
//     // And it contains the building block keys too.
//     dense_cpu_ks.has(DispatchKey::CPUBit);
//     dense_cpu_ks.has(DispatchKey::Dense);
//
// Not every backend and not every functionality counts as a "building block
// key". This is mostly to give us more levers to pull in the design space.
// Backend keys and functionality keys that count as "building blocks" will
// contribute to a full cross product of functionality that can be overriden.
```

**“构建块”键始终对应于 DispatchKeySet 中的某个位。**它们也可以被组合形成实际的运行时键，例如：

```cpp
auto dense_cpu_ks = DispatchKeySet({DispatchKey::CPUBit, DispatchKey::Dense});
// 该键集具有运行时的 dense-cpu 键。
dense_cpu_ks.has(DispatchKey::CPU);
// 它还包含构建块键。
dense_cpu_ks.has(DispatchKey::CPUBit);
dense_cpu_ks.has(DispatchKey::Dense);

```

在上述示例中，`dense_cpu_ks` 是一个包含 `DispatchKey::CPUBit` 和 `DispatchKey::Dense` 的调度键集。它既包含了运行时键 `DispatchKey::CPU`，也包含了构建块键 `DispatchKey::CPUBit` 和 `DispatchKey::Dense`。

```
// (2) Every runtime key corresponds directly to a slot in an operator's runtime
// dispatch table, and you can directly register kernels to a runtime dispatch
// key.
//
// For per-backend functionalities like "Dense" or "AutogradFunctionality",
// you can think of the corresponding runtime dispatch keys as "instances" of
// that functionality, per backend. E.g. "CPU", "CUDA", "XLA", etc. are all
// runtime instances of the "Dense" building block key.

// (2a) and (2b) are represented identically in the DispatchKeySet logic:
// - backend-agnostic functionalities (e.g. FuncTorchBatched) are NOT
// customizeable per backend.
//   In order to do so, we'd need to promote it to a per-backend functionality
//   "building block" key.
// - non-customizeable backends (e.g. FPGA) can NOT customize existing
// functionality like Sparse, Autograd, etc.
//   In order to do so, we'd need to promote it to a backend "building block"
//   key.
//
// In both cases, these keys directly correspond to runtime slots in the
// operator table.

```

**“运行时”键直接对应于操作符的运行时调度表中的一个 slot**，可以直接为运行时键注册内核。像 "Dense" 或 "AutogradFunctionality" 这样的 per-backend 功能，可以将相应的运行时调度键视为该功能在每个后端上的 "实例"。例如，"CPU"、"CUDA"、"XLA" 等都是 "Dense" 构建块键的运行时实例。

2a 和 2b 在 DispatchKeySet 逻辑中的表示方式是相同的：
- 与后端无关的功能（例如 FuncTorchBatched）不能在每个后端上进行自定义。要实现这一点，我们需要将其提升为 per-backend 功能 "构建块" 键。
- 不可自定义的后端（例如 FPGA）不能自定义现有的功能，如 Sparse、Autograd 等。要实现这一点，我们需要将其提升为后端 "构建块" 键。

在这两种情况下，这些键直接对应于操作符表中的运行时 slot。

## 总结

### DispatchKey

| 类型 | 功能键/后端键 | 占用位/占用 slot |
| --- | --- | --- |
| 不可自定义的后端 | 后端键 | 占用位/占用slot |
| 不可自定义的功能 | 功能键 | 占用位/占用slot |
| 可以根据后端自定义的功能 | 功能键 | 占用位 |
| 可以自定义功能的后端实例 | 后端键 | 占用slot |

### DispatchKey + BackendComponent

| 类型 | 构建键/运行时键 | 占用位/占用 slot |
| --- | --- | --- |
| BackendComponent 所有枚举值（可定制的后端） | 构建键 | 占用位 |
| （per-backend）的功能（可自定义的功能） | 构建键 | 占用位 |
| 不可自定义的后端 | 运行时键 | 占用位/占用slot |
| 不可自定义的功能 | 运行时键 | 占用位/占用slot |
| 可自定义功能的后端 | 运行时键 | 占用slot |

# 源码阅读

## DispatchKey

主要关注这三个转换函数：
- **toBackendComponent**：计算出该 DispatchKey（需要是 Per-Backend Functionality Dispatch Key）对应的 BackendComponent
- **toFunctionalityKey**：计算出该 DispatchKey（需要是 Per-Backend Functionality Dispatch Key）对应的 Functionality
- **toRuntimePerBackendFunctionalityKey** ：将 BackendComponent 和 Functionality 组合成 Per-Backend Functionality Dispatch Key

```cpp
// See Note [The Ordering of Per-Backend Dispatch Keys Matters!]
// This function relies on the invariant that the dispatch keys between
// StartOfDenseBackends and EndOfRuntimeBackendKeys are ordered by backend
// in the same order as `BackendComponent`.
constexpr BackendComponent toBackendComponent(DispatchKey k) {
  if (k >= DispatchKey::StartOfDenseBackends &&
      k <= DispatchKey::EndOfDenseBackends) {
    return static_cast<BackendComponent>(
        static_cast<uint8_t>(k) -
        static_cast<uint8_t>(DispatchKey::StartOfDenseBackends));
  } else if (
      k >= DispatchKey::StartOfQuantizedBackends &&
      k <= DispatchKey::EndOfQuantizedBackends) {
    return static_cast<BackendComponent>(
        static_cast<uint8_t>(k) -
        static_cast<uint8_t>(DispatchKey::StartOfQuantizedBackends));
  }
  ...
  } else {
    return BackendComponent::InvalidBit;
  }
}

constexpr DispatchKey toFunctionalityKey(DispatchKey k) {
  if (k <= DispatchKey::EndOfFunctionalityKeys) {
    return k;
  } else if (k <= DispatchKey::EndOfDenseBackends) {
    return DispatchKey::Dense;
  } else if (k <= DispatchKey::EndOfQuantizedBackends) {
    return DispatchKey::Quantized;
  } else if (k <= DispatchKey::EndOfSparseBackends) {
    return DispatchKey::Sparse;
  } else if (k <= DispatchKey::EndOfNestedTensorBackends) {
    return DispatchKey::NestedTensor;
  } else if (k <= DispatchKey::EndOfAutogradFunctionalityBackends) {
    return DispatchKey::AutogradFunctionality;
  } else {
    return DispatchKey::Undefined;
  }
}

// Given (DispatchKey::Dense, BackendComponent::CUDABit), returns
// DispatchKey::CUDA.
// See Note [The Ordering of Per-Backend Dispatch Keys Matters!]
// This function relies on the invariant that the dispatch keys between
// StartOfDenseBackends and EndOfRuntimeBackendKeys are ordered by backend
// in the same order as `BackendComponent`.
constexpr DispatchKey toRuntimePerBackendFunctionalityKey(
    DispatchKey functionality_k,
    BackendComponent backend_k) {
  if (functionality_k == DispatchKey::Dense) {
    return static_cast<DispatchKey>(
        static_cast<uint8_t>(DispatchKey::StartOfDenseBackends) +
        static_cast<uint8_t>(backend_k));
  }
  if (functionality_k == DispatchKey::Sparse) {
    return static_cast<DispatchKey>(
        static_cast<uint8_t>(DispatchKey::StartOfSparseBackends) +
        static_cast<uint8_t>(backend_k));
  }
  if (functionality_k == DispatchKey::Quantized) {
    return static_cast<DispatchKey>(
        static_cast<uint8_t>(DispatchKey::StartOfQuantizedBackends) +
        static_cast<uint8_t>(backend_k));
  }
  if (functionality_k == DispatchKey::NestedTensor) {
    return static_cast<DispatchKey>(
        static_cast<uint8_t>(DispatchKey::StartOfNestedTensorBackends) +
        static_cast<uint8_t>(backend_k));
  }
  if (functionality_k == DispatchKey::AutogradFunctionality) {
    return static_cast<DispatchKey>(
        static_cast<uint8_t>(
            DispatchKey::StartOfAutogradFunctionalityBackends) +
        static_cast<uint8_t>(backend_k));
  }
  return DispatchKey::Undefined;
}
```

## DispatchKeySet

```cpp
class DispatchKeySet final {
private:
 uint64_t repr_ = 0;
}
```

### 构造函数
由 BackendComponent / DispatchKey 初始化 DispatchKeySet 中对应的位。**重点关注 DispatchKey 对不同类型键的处理（尤其是 case 3）**：

```cpp
  constexpr explicit DispatchKeySet(BackendComponent k) {
    if (k == BackendComponent::InvalidBit) {
      repr_ = 0;
    } else {
      repr_ = 1ULL << (static_cast<uint8_t>(k) - 1);
    }
  }

  constexpr explicit DispatchKeySet(DispatchKey k) {
    if (k == DispatchKey::Undefined) {
      repr_ = 0;
    } else if (k <= DispatchKey::EndOfFunctionalityKeys) {
      // Case 2: handle "functionality-only" keys
      // These keys have a functionality bit set, but no backend bits
      // These can technically be either:
      // - valid runtime keys (e.g. DispatchKey::AutogradOther,
      // DispatchKey::FuncTorchBatched, etc)
      // - "building block" keys that aren't actual runtime keys (e.g.
      // DispatchKey::Dense or Sparse)
      uint64_t functionality_val = 1ULL
          << (num_backends + static_cast<uint8_t>(k) - 1);
      repr_ = functionality_val;
    } else if (k <= DispatchKey::EndOfRuntimeBackendKeys) {
      // Case 3: "runtime" keys that have a functionality bit AND a backend bit.
      // First compute which bit to flip for the functionality.
      auto functionality_k = toFunctionalityKey(k);
      // The - 1 is because Undefined is technically a "functionality" that
      // doesn't show up in the bitset. So e.g. Dense is technically the second
      // functionality, but the lowest functionality bit.
      uint64_t functionality_val = 1ULL
          << (num_backends + static_cast<uint8_t>(functionality_k) - 1);

      // then compute which bit to flip for the backend
      // Case 4a: handle the runtime instances of "per-backend functionality"
      // keys For example, given DispatchKey::CPU, we should set:
      // - the Dense functionality bit
      // - the CPUBit backend bit
      // first compute which bit to flip for the backend
      auto backend_k = toBackendComponent(k);
      uint64_t backend_val = backend_k == BackendComponent::InvalidBit
          ? 0
          : 1ULL << (static_cast<uint8_t>(backend_k) - 1);
      repr_ = functionality_val + backend_val;
    } else {
      // At this point, we should have covered every case except for alias keys.
      // Technically it would be possible to add alias dispatch keys to a
      // DispatchKeySet, but the semantics are a little confusing and this
      // currently isn't needed anywhere.
      repr_ = 0;
    }
  }

```

### 优先级计算

**indexOfHighestBit** 返回最高有效位的索引：

```cpp
  
  // Returns the index of the most-significant bit in the keyset.
  // This is used to as part of the calculation into the operator table to get:
  // - the highest "functionality" bit in the keyset.
  // - the highest "backend" bit in the keyset.
  uint8_t indexOfHighestBit() const {
    return 64 - llvm::countLeadingZeros(repr_);
  }
```

**highestFunctionalityKey**：返回最高优先级的 Functionality  
**highestBackendKey**：返回最高优先级的 BackendComponent

```cpp

  DispatchKey highestFunctionalityKey() const {
    auto functionality_idx = indexOfHighestBit();
    // This means that none of the functionality bits were set.
    if (functionality_idx < num_backends)
      return DispatchKey::Undefined;
    // The first num_backend bits in the keyset don't correspond to real
    // dispatch keys.
    return static_cast<DispatchKey>(functionality_idx - num_backends);
  }

  BackendComponent highestBackendKey() const {
    // mask to mask out functionality bits
    auto backend_idx =
        DispatchKeySet(repr_ & full_backend_mask).indexOfHighestBit();
    // all zeros across the backend bits means that no backend bits are set.
    if (backend_idx == 0)
      return BackendComponent::InvalidBit;
    return static_cast<BackendComponent>(backend_idx);
  }
```

**highestPriorityTypeId**：返回 DispatchKeySet 中最高优先级的 DispatchKey，如果是 per-backend 功能，则需要调用 toRuntimePerBackendFunctionalityKey，返回 backend 和 functionality 组合成的 DispatchKey：

```cpp
  // returns the DispatchKey of highest priority in the set.
  DispatchKey highestPriorityTypeId() const {
    auto functionality_k = highestFunctionalityKey();
    if (isPerBackendFunctionalityKey(functionality_k)) {
      return toRuntimePerBackendFunctionalityKey(
          functionality_k, highestBackendKey());
    }
    return functionality_k;
  }
```