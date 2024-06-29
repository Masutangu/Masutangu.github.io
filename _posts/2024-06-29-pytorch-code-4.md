---
layout: post
date: 2024-06-29T16:46:37+08:00
title: 【PyTorch 源码阅读】Dispatcher 源码篇

tags: 
  - 源码阅读
---

# 类

## Dispatcher & OperatorDef

[https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/dispatch/Dispatcher.h](https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/dispatch/Dispatcher.h)

```cpp
/**
 * Top-level dispatch interface for dispatching via the dynamic dispatcher.
 * Most end users shouldn't use this directly; if you're trying to register
 * ops look in op_registration
 */
class TORCH_API Dispatcher final {
  private:
    struct OperatorDef final {
	    explicit OperatorDef(OperatorName&& op_name)
	    : op(std::move(op_name)) {}
	
	    impl::OperatorEntry op;
	
	    // These refer to the number of outstanding RegistrationHandleRAII
	    // for this operator.  def_count reflects only def() registrations
	    // (in the new world, this should only ever be 1, but old style
	    // registrations may register the schema multiple times, which
	    // will increase this count).  def_and_impl_count reflects the number
	    // of combined def() and impl() registrations.  When the last def() gets
	    // unregistered, we must immediately call the Deregistered listeners, but we
	    // must not actually delete the handle as there are other outstanding RAII
	    // destructors which will try to destruct and they had better still have a
	    // working operator handle in this case
	    size_t def_count = 0;
	    size_t def_and_impl_count = 0;
	  };
  private:
		std::list<OperatorDef> operators_;
		LeftRight<ska::flat_hash_map<OperatorName, OperatorHandle>> operatorLookupTable_;
};
```

## OperatorHandle

[https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/dispatch/Dispatcher.h](https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/dispatch/Dispatcher.h)

```cpp

/**
 * This is a handle to an operator schema registered with the dispatcher.
 * This handle can be used to register kernels with the dispatcher or
 * to lookup a kernel for a certain set of arguments.
 */
class TORCH_API OperatorHandle {
  private:
	  // Storing a direct pointer to the OperatorDef even though we
	  // already have the iterator saves an instruction in the critical
	  // dispatch path. The iterator is effectively a
	  // pointer-to-std::list-node, and (at least in libstdc++'s
	  // implementation) the element is at an offset 16 bytes from that,
	  // because the prev/next pointers come first in the list node
	  // struct. So, an add instruction would be necessary to convert from the
	  // iterator to an OperatorDef*.
	  Dispatcher::OperatorDef* operatorDef_;
	  
	  // We need to store this iterator in order to make
	  // Dispatcher::cleanup() fast -- it runs a lot on program
	  // termination (and presuambly library unloading).
	  std::list<Dispatcher::OperatorDef>::iterator operatorIterator_;
};
```

## OperatorEntry

[https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/dispatch/OperatorEntry.h](https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/dispatch/OperatorEntry.h)

```cpp
// Internal data structure that records information about a specific operator.
// It's not part of the public API; typically, users will interact with
// OperatorHandle instead.
//
// Concurrent writes to OperatorEntry are protected by the GLOBAL Dispatcher
// lock (this is important because some methods in OperatorEntry access
// dispatcher state)
class TORCH_API OperatorEntry final {
  private:
    std::array<KernelFunction, c10::num_runtime_entries> dispatchTable_;
	  DispatchKeyExtractor dispatchKeyExtractor_;
		// kernels_ stores all registered kernels for the corresponding dispatch key
	  //
	  // If an operator library gets loaded that overwrites an already existing kernel,
	  // both kernels will be in that list but only the newer one will be in
	  // dispatchTable. If any of the kernels go away (say the library gets
	  // unloaded), we remove the kernel from this list and update the
	  // dispatchTable if necessary.
	  // Kernels in the list are ordered by registration time descendingly,
	  // newer registrations are before older registrations.
	  // We do not combine dispatchTable and kernels into one hash map because
	  // kernels is a larger data structure and accessed quite infrequently
	  // while dispatchTable is accessed often and should be kept small to fit
	  // into CPU caches.
	  // Invariants:
	  //  - dispatchTable[dispatch_key] == kernels_[dispatch_key].front()
	  //  - dispatchTable[dispatch_key] does not exist if and only if
	  //    kernels_[dispatch_key] does not exist
	  //  - If kernels_[dispatch_key] exists, then it has elements.
	  //    It is never an empty list.
	  ska::flat_hash_map<DispatchKey, std::list<AnnotatedKernel> > kernels_;
};
```

## DispatchKeyExtractor

[https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/dispatch/DispatchKeyExtractor.h](https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/dispatch/DispatchKeyExtractor.h)

```cpp

/**
 * An instance of DispatchKeyExtractor knows how to get a dispatch key given
 * a list of arguments for an operator call.
 *
 * The instance is specific for a certain operator as:
 *  - In boxed dispatch, different operators have different ways to extract
 *    the dispatch key (e.g. different numbers of arguments), and we precompute
 *    the stack locations we should look at; and
 *  - In all dispatch, some backends should be excluded from dispatch because
 *    they have been registered as fallthrough.  The set of excluded backends
 *    varies from operator, as some operators may have overridden the
 *    fallthrough with custom behavior.
 *
 *   Note - this should maintain identical impl to the py dispatcher key extraction logic
 *   at pytorch/torch/dispatcher.py
 */
struct TORCH_API DispatchKeyExtractor final {
};
```

# 注册运算符

注册 operator 有两种方式：native_functions.yaml 和 custom operators。在 [README.md](https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/op_registration/README.md) 中对这两种做了比较：

PyTorch 的公共 API 中的所有运算符都在 native_functions.yaml 中定义。只需在其中添加一个条目，并编写相应的 C++ 内核函数即可。那何时不应该使用 native_functions.yaml 呢？主要是以下四种场景：

- 编写一个不应该成为 PyTorch 公共 API 的新运算符
- 编写一个新的运算符，但不想更改核心 PyTorch 代码库
- 为 PyTorch 编写 C++ 扩展，或者在您的 .py 模型文件中使用内联 C++
- 编写一个后端库，比如 XLA 或 ORT，该库为 `native_functions.yaml` 中定义的所有运算符添加新的内核

## custom operators

有两种方式：RegisterOperators  和 TORCH_LIBRARY。关于这两种的对比，在 [**Difference between TORCH_LIBRARY() and RegisterOperators::op()**](https://discuss.pytorch.org/t/difference-between-torch-library-and-registeroperators-op/92270) 中作者提到他们的内部实现机制是一样的，TORCH_LIBRARY API 更推荐使用。

### RegisterOperators

[https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/op_registration/op_registration.h](https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/op_registration/op_registration.h)

```cpp
/**
 * An instance of this class handles the registration for one or more operators.
 * Make sure you keep the RegisterOperators instance around since it will
 * deregister the operator it's responsible for in its destructor.
 *
 * Example:
 *
 * > namespace {
 * >   class my_kernel_cpu final : public c10::OperatorKernel {
 * >   public:
 * >     Tensor operator()(Tensor a, Tensor b) {...}
 * >   };
 * > }
 * >
 * > static auto registry = c10::RegisterOperators()
 * >     .op(c10::RegisterOperators::options()
 * >         .schema("my_op")
 * >         .kernel<my_kernel_cpu>(DispatchKey::CPU));
 */
 class TORCH_API RegisterOperators final {
 };
```

注释中给出示例：
```cpp
namespace {
  class my_kernel_cpu final : public c10::OperatorKernel {
    public:
      Tensor operator()(Tensor a, Tensor b) {...}
  };
}
  
static auto registry = c10::RegisterOperators()
            .op(c10::RegisterOperators::options()
            .schema("my_op")
            .kernel<my_kernel_cpu>(DispatchKey::CPU));
```


```cpp
namespace {
  class my_kernel_cpu final : public c10::OperatorKernel {
    public:
      Tensor operator()(Tensor a, Tensor b) {...}
  };
}

static auto registry = c10::RegisterOperators()
            .op(c10::RegisterOperators::options()
            .schema("my_op")
            .catchAllKernel<my_kernel_cpu>());
```

`op` 最终会调用 `registerOp_` 进行运算符的注册：

```cpp
void RegisterOperators::registerOp_(Options&& options) {
  ...
  registrars_.emplace_back(
    Dispatcher::singleton().registerDef(std::move(schema), "registered by RegisterOperators")
  );

  for (auto& kernel : options.kernels) {
    registrars_.emplace_back(
      Dispatcher::singleton().registerImpl(op_name, kernel.dispatch_key, std::move(kernel.func), std::move(kernel.cpp_signature), std::move(kernel.inferred_function_schema), "registered by RegisterOperators")
    );
  }
}
```

### TORCH_LIBRARY

[https://github.com/pytorch/pytorch/blob/main/torch/library.h](https://github.com/pytorch/pytorch/blob/main/torch/library.h)
[https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/library.cpp](https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/core/library.cpp)

```cpp
/// This object provides the API for defining operators and providing
/// implementations at dispatch keys.  Typically, a torch::Library
/// is not allocated directly; instead it is created by the
/// TORCH_LIBRARY() or TORCH_LIBRARY_IMPL() macros.
///
/// Most methods on torch::Library return a reference to itself,
/// supporting method chaining.
///
/// ```
/// // Examples:
///
/// TORCH_LIBRARY(torchvision, m) {
///    // m is a torch::Library
///    m.def("roi_align", ...);
///    ...
/// }
///
/// TORCH_LIBRARY_IMPL(aten, XLA, m) {
///    // m is a torch::Library
///    m.impl("add", ...);
///    ...
/// }
/// ```
///
class TORCH_API Library final {
};
```

使用方式可参考 [https://pytorch.org/tutorials/advanced/dispatcher.html](https://pytorch.org/tutorials/advanced/dispatcher.html) 。重点关注 `_def` 和 `_impl` 方法：

```cpp
#define DEF_PRELUDE "def(\"", schema.operator_name(), "\"): "
Library& Library::_def(c10::FunctionSchema&& schema, c10::OperatorName* out_name, const std::vector<at::Tag>& tags, _RegisterOrVerify rv) & {
	registrars_.emplace_back(
        c10::Dispatcher::singleton().registerDef(
          std::move(schema),
          debugString(file_, line_),
          tags
        )
      );
 }
 
 Library& Library::_impl(const char* name_str, CppFunction&& f, _RegisterOrVerify rv) & {
    auto dispatch_key = f.dispatch_key_.has_value() ? f.dispatch_key_ : dispatch_key_;
    registrars_.emplace_back(
        c10::Dispatcher::singleton().registerImpl(
          std::move(name),
          dispatch_key,
          std::move(f.func_),
          std::move(f.cpp_signature_),
          std::move(f.schema_),
          debugString(std::move(f.debug_), file_, line_)
        )
      );
}
```

## Dispatcher

从上面 RegisterOperators  和 TORCH_LIBRARY 的实现可以看出，他们都调用了 Dispatcher 的 `registerDef` 和 `registerImpl` 方法：

```cpp
/**
   * Register a new operator schema.
   *
   * If a schema with the same operator name and overload name already exists,
   * this function will check that both schemas are exactly identical.
   */
  RegistrationHandleRAII registerDef(FunctionSchema schema, std::string debug, std::vector<at::Tag> tags = {});

  /**
   * Register a kernel to the dispatch table for an operator.
   * If dispatch_key is nullopt, then this registers a fallback kernel.
   *
   * @return A RAII object that manages the lifetime of the registration.
   *         Once that object is destructed, the kernel will be deregistered.
   */
  // NB: steals the inferred function schema, as we may need to hold on to
  // it for a bit until the real schema turns up
  RegistrationHandleRAII registerImpl(OperatorName op_name, c10::optional<DispatchKey> dispatch_key, KernelFunction kernel, c10::optional<impl::CppSignature> cpp_signature, std::unique_ptr<FunctionSchema> inferred_function_schema, std::string debug);
```

### registerDef

```cpp
RegistrationHandleRAII Dispatcher::registerDef(FunctionSchema schema, std::string debug, std::vector<at::Tag> tags) {
  OperatorName op_name = schema.operator_name();
  auto op = findOrRegisterName_(op_name);

  op.operatorDef_->op.registerSchema(std::move(schema), std::move(debug), std::move(tags));

  // NB: do not increment the counts until AFTER error checking
  ++op.operatorDef_->def_count;
  ++op.operatorDef_->def_and_impl_count;

  return RegistrationHandleRAII([guard = this->guard_, this, op, op_name] {
    deregisterDef_(op, op_name);
  });
}
```

`registerDef` 主要的逻辑就是注册 op（存到 `operators_` 和 `operatorLookupTable_` 中）：

```cpp
// Postcondition: caller is responsible for disposing of registration when they
// are done
OperatorHandle Dispatcher::findOrRegisterName_(const OperatorName& op_name) {
  const auto found = findOp(op_name);
  if (found != c10::nullopt) {
    return *found;
  }

  operators_.emplace_back(OperatorName(op_name));
  OperatorHandle handle(--operators_.end());
  operatorLookupTable_.write([&] (ska::flat_hash_map<OperatorName, OperatorHandle>& operatorLookupTable) {
    operatorLookupTable.emplace(op_name, handle);
  });

  return handle;
}
```

### registerImpl

```cpp
RegistrationHandleRAII Dispatcher::registerImpl(
  OperatorName op_name,
  c10::optional<DispatchKey> dispatch_key,
  KernelFunction kernel,
  c10::optional<impl::CppSignature> cpp_signature,
  std::unique_ptr<FunctionSchema> inferred_function_schema,
  std::string debug
) {
  auto op = findOrRegisterName_(op_name);

  auto handle = op.operatorDef_->op.registerKernel(
    *this,
    dispatch_key,
    std::move(kernel),
    std::move(cpp_signature),
    std::move(inferred_function_schema),
    std::move(debug)
  );

  ++op.operatorDef_->def_and_impl_count;

  return RegistrationHandleRAII([guard = this->guard_, this, op, op_name, dispatch_key, handle] {
    deregisterImpl_(op, op_name, dispatch_key, handle);
  });
}
```

`registerKernel` 主要逻辑是调用 OperatorEntry  的 `registerKernel` 方法注册 kernel。

## OperatorEntry

### registerKernel

```cpp

OperatorEntry::AnnotatedKernelContainerIterator OperatorEntry::registerKernel(
  const c10::Dispatcher& dispatcher,
  c10::optional<DispatchKey> dispatch_key,
  KernelFunction kernel,
  c10::optional<CppSignature> cpp_signature,
  std::unique_ptr<FunctionSchema> inferred_function_schema,
  std::string debug
) {
  // Add the kernel to the kernels list,
  // possibly creating the list if this is the first kernel.
  auto& k = dispatch_key.has_value() ? kernels_[*dispatch_key] : kernels_[DispatchKey::CompositeImplicitAutograd];
  k.emplace_front(std::move(kernel), std::move(inferred_function_schema), std::move(debug));
  AnnotatedKernelContainerIterator inserted = k.begin();
  // update the dispatch table, i.e. re-establish the invariant
  // that the dispatch table points to the newest kernel
  if (dispatch_key.has_value()) {
    updateDispatchTable_(dispatcher, *dispatch_key);
  } else {
    updateDispatchTableFull_(dispatcher);
  }
  return inserted;
}
```

`registerKernel` 主要做两件事：

- 写入到 `kernels_` 中
- 更新 `dispatchTable_`，即运行时调度键表。其为数组类型，以调度键为下标

`updateDispatchTable_` 和 `updateDispatchTableFull_` 最终都是调用 `updateDispatchTableEntry_`：

```cpp
// synchronizes the dispatch table entry for a given dispatch key
// with the current state of kernel registrations in the dispatcher.
// note that this is not a complete update, due to relationships between
// dispatch keys (e.g. runtime keys and their associated autograd keys,
// or alias keys and their associated keysets).
// This function should be considered a private helper for updateDispatchTable_()
void OperatorEntry::updateDispatchTableEntry_(const c10::Dispatcher& dispatcher, DispatchKey dispatch_key) {
  const auto dispatch_ix = getDispatchTableIndexForDispatchKey(dispatch_key);
  if (C10_UNLIKELY(dispatch_ix == -1)) {
    return;
  }
  dispatchTable_[dispatch_ix] = computeDispatchTableEntry(dispatcher, dispatch_key);
}
```

`getDispatchTableIndexForDispatchKey` 会返回该 dispatch_key 在 `dispatchTable_` 中的下标。

# 调用运算符

调用运算符步骤如下：

- 调用 `findSchema` ，通过运算符名字找到对应的 OperatorHandle
- 调用 OperatorHandle 的 `call/callBox` 方法

## Dispatcher

`findSchema` 从 `operatorLookupTable_` 找到 OperatorName 对应的 OperatorHandle：

```cpp
c10::optional<OperatorHandle> Dispatcher::findSchema(const OperatorName& overload_name) {
  auto it = findOp(overload_name);
  if (it.has_value()) {
    if (it->hasSchema()) {
      return it;
    } else {
      return c10::nullopt;
    }
  } else {
    return it;
  }
}

c10::optional<OperatorHandle> Dispatcher::findOp(const OperatorName& overload_name) {
  return operatorLookupTable_.read([&] (const ska::flat_hash_map<OperatorName, OperatorHandle>& operatorLookupTable) -> c10::optional<OperatorHandle> {
    auto found = operatorLookupTable.find(overload_name);
    if (found == operatorLookupTable.end()) {
      return c10::nullopt;
    }
    return found->second;
  });
}
```

call/callBoxed 调用运算符对应的函数：

**call Unbox**

```cpp
 
// See [Note: Argument forwarding in the dispatcher] for why Args doesn't use &&
template<class Return, class... Args>
C10_ALWAYS_INLINE_UNLESS_MOBILE Return Dispatcher::call(const TypedOperatorHandle<Return(Args...)>& op, Args... args) const {
  auto dispatchKeySet = op.operatorDef_->op.dispatchKeyExtractor()
    .template getDispatchKeySetUnboxed<Args...>(args...);
  const KernelFunction& kernel = op.operatorDef_->op.lookup(dispatchKeySet);
  return kernel.template call<Return, Args...>(op, dispatchKeySet, std::forward<Args>(args)...);
}

```

**call Box**

```cpp
inline void Dispatcher::callBoxed(const OperatorHandle& op, Stack* stack) const {
  // note: this doesn't need the mutex because write operations on the list keep iterators intact.
  const auto& entry = op.operatorDef_->op;
  auto dispatchKeySet = entry.dispatchKeyExtractor().getDispatchKeySetBoxed(stack);
  const auto& kernel = entry.lookup(dispatchKeySet);
  kernel.callBoxed(op, dispatchKeySet, stack);
}

```

分为三个步骤：

### 计算 DispatchKeySet

通过 DispatchKeyExtractor 的 `getDispatchKeySetBoxed`/`getDispatchKeySetUnboxed` 方法计算 DispatchKeySet：

```cpp
    DispatchKeySet getDispatchKeySetBoxed(const torch::jit::Stack* stack) const {
    DispatchKeySet ks;
    dispatch_arg_indices_reverse_.for_each_set_bit([&] (size_t reverse_arg_index) {
      const auto& ivalue = torch::jit::peek(*stack, 0, reverse_arg_index + 1);
      if (C10_LIKELY(ivalue.isTensor())) {
        // NB: Take care not to introduce a refcount bump (there's
        // no safe toTensorRef method, alas)
        ks = ks | ivalue.unsafeToTensorImpl()->key_set();
      } else if (C10_UNLIKELY(ivalue.isTensorList())) {
        for (const at::Tensor& tensor : ivalue.toTensorList()) {
          ks = ks | tensor.key_set();
        }
      }
      // Tensor?[] translates to a c10::List<IValue> so we need to peek inside
      else if (C10_UNLIKELY(ivalue.isList())) {
        for (const auto& elt : ivalue.toListRef()) {
          if (elt.isTensor()) {
            ks = ks | elt.toTensor().key_set();
          }
        }
      }
    });
   
     return impl::computeDispatchKeySet(ks, nonFallthroughKeys_);
  }
  
  template<class... Args>
  DispatchKeySet getDispatchKeySetUnboxed(const Args&... args) const {
    auto ks = detail::multi_dispatch_key_set(args...);
    // Keys that are fallthrough should be skipped
    if (requiresBitsetPerBackend_) {
      auto backend_idx = ks.getBackendIndex();
      return impl::computeDispatchKeySet(ks, nonFallthroughKeysPerBackend_[backend_idx]);
    } else {
      return impl::computeDispatchKeySet(ks, nonFallthroughKeys_);
    }
  }
```

`getDispatchKeySetBoxed`/`getDispatchKeySetUnboxed` 的逻辑很简单，即对所有参数的 DispatchKeySet 进行合并，然后调用 `computeDispatchKeySet`，结合本地包含集（local include set）和 本地排除集（local exclude set），计算出最终的 DispatchKeySet：

```cpp
// Take a DispatchKeySet for a Tensor and determine what the actual dispatch
// DispatchKey should be, taking into account TLS, and skipping backends which
// fall through.
//
// Unlike Tensor::key_set(), the value of this on a tensor can change depending
// on TLS.
//
// NB: If there is no valid dispatch key, this will return Undefined
static inline DispatchKeySet computeDispatchKeySet(
    DispatchKeySet ks,
    // The key mask lets us eliminate (by zero entries) keys which should not
    // be considered for dispatch.  There are two cases when we use this:
    //
    // - If an operator's dispatch table contains a fallthrough entry, we
    //   should bypass it entirely when finding the key
    // - If a user invokes with redispatch, the mask lets us
    //   zero out the key the user asked us to stop.
    //
    // These excluded backends are NOT tracked in the TLS, but must be applied
    // AFTER TLS (since the backend may have been introduced for consideration
    // by the included TLS), which is why you have to pass them in to this
    // function (as opposed to just applying it to the input 'ks').
    DispatchKeySet key_mask
) {
  c10::impl::LocalDispatchKeySet local = c10::impl::tls_local_dispatch_key_set();

  return (((ks | local.included_) - local.excluded_) & key_mask);
}
```

### 查找 Kernel

调用 `getDispatchTableIndexForDispatchKeySet` 返回 DispatchKeySet 中优先级最高的键对应的下标，并返回 dispatchTable_ 中该下标的 kernel：

```cpp
  const KernelFunction& lookup(DispatchKeySet ks) const {
    const auto idx = ks.getDispatchTableIndexForDispatchKeySet();
    const auto& kernel = dispatchTable_[idx];
    return kernel;
  }
```

# 总结
结合代码实现和 [【PyTorch 源码阅读】dispatch 机制概念篇](https://masutangu.com/2024/06/10/pytorch-code-2/)：

- OperatorEntry 中的 `dispatchTable_` 即函数指针表（dispatch table），也就是[【PyTorch 源码阅读】DispatchKeySet 源码篇](https://masutangu.com/2024/06/16/pytorch-code-3/)中的运行时操作符表
- 由 DispatchKeyExtractor 的 `getDispatchKeySetBoxed`/`getDispatchKeySetUnboxed` 方法实现 dispatch key set 的合并
