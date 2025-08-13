# 深入解析 C++ 替换函数 (Replacement Functions)

在 C++ 中，内存管理是一个核心且复杂的议题。[`new`](https://en.cppreference.com/w/cpp/language/new.html) 和 [`delete`](https://en.cppreference.com/w/cpp/language/delete.html) 表达式是 C++ 程序员日常使用的工具，用于动态分配和释放内存。然而，C++ 标准提供了一种强大的机制，允许开发者在全局范围内重载 [`operator new`](https://en.cppreference.com/w/cpp/memory/new/operator_new.html) 和 [`operator delete`](https://en.cppreference.com/w/cpp/memory/new/operator_delete.html)，从而完全接管程序的动态内存分配行为。这些全局重载的函数被称为 [“替换函数” (Replacement Functions)](https://en.cppreference.com/w/cpp/language/replacement_function.html)，因为它们会替换掉 C++ 标准库提供的默认实现。

本文将详细探讨替换函数的概念、工作原理、不同形式的签名、实际应用场景以及实现时需要注意的关键陷阱。

## 什么是替换函数？

替换函数是用户定义的、具有特定签名的全局 `operator new`、`operator new[]`、`operator delete` 和 `operator delete[]` 函数。当一个程序定义了这些全局函数中的任何一个，链接器会使用用户定义的版本，而不是标准库提供的默认版本。

这种机制为底层内存管理打开了一扇大门，使得实现以下高级功能成为可能：

*   **自定义内存分配器**：例如，使用内存池 (Memory Pool) 来提高特定大小对象分配的性能，减少内存碎片。
*   **内存使用分析与调试**：记录每次内存分配和释放的详细信息（如大小、位置、时间），用于检测内存泄漏、越界写等问题。
*   **与特殊硬件交互**：在嵌入式系统或 GPU 编程中，内存可能需要从特定的物理地址或共享内存区域分配。
*   **强制特定的内存对齐**：为需要 SIMD 指令集或有其它特殊对齐要求的对象提供保证。

## 替换函数的种类与签名

C++ 标准定义了多种可被替换的全局分配和释放函数。它们可以分为两大类：分配函数和释放函数，每一类又包含单一对象和数组的形式。

### 1. [分配函数 (Allocation Functions)](https://en.cppreference.com/w/cpp/memory/new/operator_new.html)

当使用 [`new T`](https://en.cppreference.com/w/cpp/language/new.html) 或 [`new T[n]`](https://en.cppreference.com/w/cpp/language/new.html) 表达式时，编译器会调用一个合适的 `operator new` 或 `operator new[]` 函数来获取原始内存。

可被替换的全局 `operator new` 签名包括：

```cpp
// 1. 普通版本 (throwing)
// 若分配失败，抛出 std::bad_alloc 异常
void* operator new(std::size_t count);

// 2. 不抛出异常版本 (nothrow)
// 若分配失败，返回 nullptr
void* operator new(std::size_t count, const std::nothrow_t& tag) noexcept;

// 3. C++17 新增：带对齐参数的版本 (throwing)
void* operator new(std::size_t count, std::align_val_t al);

// 4. C++17 新增：带对齐参数的不抛出异常版本 (nothrow)
void* operator new(std::size_t count, std::align_val_t al, const std::nothrow_t& tag) noexcept;

// -- 数组版本 --
// 行为与单一对象版本类似，只是由 new T[n] 表达式调用

void* operator new[](std::size_t count);
void* operator new[](std::size_t count, const std::nothrow_t& tag) noexcept;
void* operator new[](std::size_t count, std::align_val_t al);
void* operator new[](std::size_t count, std::align_val_t al, const std::nothrow_t& tag) noexcept;
```

*   `count`：需要分配的字节数。编译器会自动计算，例如 `new MyClass` 的 `count` 就是 `sizeof(MyClass)`。
*   `std::nothrow_t`：一个空的 tag 类型，用于标识不抛出异常的版本。通过 `new(std::nothrow) T` 语法调用。
*   `std::align_val_t`：一个枚举类型，用于传递 C++17 的对齐要求。通过 `new(std::align_val_t{alignof(T)}) T` 语法调用。

### 2. [释放函数 (Deallocation Functions)](https://en.cppreference.com/w/cpp/memory/new/operator_delete.html)

当使用 [`delete p`](https://en.cppreference.com/w/cpp/language/delete.html) 或 [`delete[] p`](https://en.cppreference.com/w/cpp/language/delete.html) 表达式时，编译器首先调用对象的析构函数，然后调用一个合适的 `operator delete` 或 `operator delete[]` 函数来释放内存。

可被替换的全局 `operator delete` 签名包括：

```cpp
// 1. 普通版本
// 必须处理 ptr 为 nullptr 的情况（标准规定此时应无操作）
void operator delete(void* ptr) noexcept;

// 2. C++14 新增：带大小的版本 (Sized Deallocation)
// 编译器可能会传递对象的大小，用于优化内存回收
void operator delete(void* ptr, std::size_t size) noexcept;

// 3. C++17 新增：带对齐参数的版本
void operator delete(void* ptr, std::align_val_t al) noexcept;

// 4. C++17 新增：带大小和对齐参数的版本
void operator delete(void* ptr, std::size_t size, std::align_val_t al) noexcept;

// -- 数组版本 --

void operator delete[](void* ptr) noexcept;
void operator delete[](void* ptr, std::size_t size) noexcept;
void operator delete[](void* ptr, std::align_val_t al) noexcept;
void operator delete[](void* ptr, std::size_t size, std::align_val_t al) noexcept;
```

*   `ptr`：要释放的内存指针。
*   `size`：（可选）编译器传递的要释放内存块的大小。这是一个优化特性，实现不能假设 `size` 总是有效（例如，对于不完整类型，`size` 可能为 0）。
*   `al`：（可选）内存块的对齐方式。

**重要**：如果同时定义了带大小和不带大小的 `delete` 版本，编译器会优先选择带大小的版本。如果只定义了不带大小的版本，它将作为所有情况下的回退选项。

## 工作原理：`new`/`delete` 表达式与 `operator new`/`delete` 函数

理解 `new` 表达式和 `operator new` 函数之间的区别至关重要。

*   **`new` 表达式** 是一个语言层面的操作，它执行两个步骤：
    1.  调用 `operator new` 函数分配足够大的原始内存。
    2.  如果内存分配成功，在这块内存上调用对象的构造函数来初始化对象。

*   **`delete` 表达式** 同样执行两个步骤：
    1.  调用对象的析构函数。
    2.  调用 `operator delete` 函数释放内存。

替换函数仅仅是重载了上述过程中的第 1 步（分配）和第 2 步（释放），而构造函数和析构函数的调用由编译器自动处理。

```cpp
// 伪代码表示 new MyClass(arg1, arg2) 的过程
void* raw_memory = ::operator new(sizeof(MyClass)); // 步骤 1: 调用分配函数
try {
  new(raw_memory) MyClass(arg1, arg2); // 步骤 2: 在原始内存上调用构造函数 (Placement New)
} catch (...) {
  // 若构造函数抛出异常，需释放已分配的内存
  ::operator delete(raw_memory, sizeof(MyClass)); // 优先调用 sized-delete
  throw; // 重新抛出异常
}
```

### 异常保证：`new` 表达式一定会抛出异常吗？

这是一个核心问题。答案是：**非 `nothrow` 版本的 `new` 表达式在设计上就是为了在失败时通过抛出异常来报告错误的。** 失败可能发生在两个阶段：内存分配阶段或对象构造阶段。

#### 1. 内存分配失败

C++ 提供了两种内存分配范式，由 `new` 表达式的语法决定调用哪个版本的 `operator new`：

*   **抛出异常式 (Throwing Version)**：`T* p = new T;`
    这是默认行为。它会调用 `operator new(size_t)`。此函数的契约是：成功时返回有效指针，**失败时必须抛出 `std::bad_alloc` 异常**。因此，使用此语法的代码需要准备处理该异常。

    ```cpp
    try {
      // 这行代码在内存不足时会抛出 std::bad_alloc
      int* large_array = new int[1000000000L];
    } catch (const std::bad_alloc& e) {
      // 在此处理内存分配失败
      std::cerr << "Memory allocation failed: " << e.what() << std::endl;
    }
    ```

*   **不抛出异常式 (Nothrow Version)**：`T* p = new(std::nothrow) T;`
    这需要显式指定 `std::nothrow`。它会调用 `operator new(size_t, const std::nothrow_t&)`。此函数的契约是：成功时返回有效指针，**失败时返回 `nullptr`**。它本身不应抛出异常（因此被标记为 `noexcept`）。

    ```cpp
    int* large_array = new(std::nothrow) int[1000000000L];
    if (large_array == nullptr) {
      // 在此通过检查空指针来处理内存分配失败
      std::cerr << "Memory allocation failed." << std::endl;
    } else {
      delete[] large_array;
    }
    ```

#### 2. 对象构造失败

即使内存分配成功，`new` 表达式仍然可能失败，因为对象的**构造函数**本身可能会抛出异常。

```cpp
class Risky {
public:
  Risky() {
    throw std::runtime_error("Constructor failed!");
  }
};

try {
  // 1. operator new(sizeof(Risky)) 成功分配内存。
  // 2. Risky::Risky() 在新内存上被调用，但它抛出了异常。
  Risky* p = new Risky();
} catch (std::runtime_error const& e) {
  // 捕获到的是构造函数抛出的异常。
  std::cout << "Caught exception from constructor: " << e.what() << std::endl;
}
```

在这种情况下，C++ 运行时系统会提供一个重要的安全保证：**如果构造函数抛出异常，运行时会自动调用相应的 `operator delete` 函数来释放刚刚分配的内存**，防止内存泄漏。然后，构造函数抛出的异常会继续向外传播。

#### 3. `new(std::nothrow)` 与构造函数异常：一个关键细节

一个常见的误解是，`new(std::nothrow)` 可以抑制所有来自 `new` 表达式的异常。这是不正确的。`std::nothrow` 的作用域**仅限于内存分配阶段**，它并不影响构造函数阶段的行为。

如果使用 `new(std::nothrow)` 时，内存分配成功，但在随后的构造函数调用中抛出了异常，那么整个 `new(std::nothrow)` 表达式的结果是**将构造函数的异常原封不动地传播出去**，而不是返回 `nullptr`。

其内部流程如下：

1.  **内存分配成功**：`operator new(size, std::nothrow)` 被调用并返回一个有效的内存指针。
2.  **对象构造失败**：构造函数在新分配的内存上执行，并抛出异常。
3.  **自动清理**：运行时系统捕获到构造函数的异常，并立即调用对应的 `operator delete` 来释放第一步中分配的内存。
4.  **异常传播**：清理完成后，构造函数最初抛出的异常继续向外传播。

下面的代码清晰地展示了这一过程：

```cpp
#include <iostream>
#include <new>
#include <stdexcept>

class FailingConstructor {
public:
  FailingConstructor() {
    std::cout << "Step 2: Entering FailingConstructor's constructor..." << std::endl;
    throw std::runtime_error("Constructor deliberately failed!");
  }
};

// 全局替换函数，用于观察调用过程
void* operator new(std::size_t size, std::nothrow_t const&) noexcept {
  std::cout << "Step 1: Custom operator new(size, nothrow) called." << std::endl;
  return std::malloc(size);
}

void operator delete(void* ptr) noexcept {
  std::cout << "Step 3: Custom operator delete called to clean up memory." << std::endl;
  std::free(ptr);
}

int main() {
  try {
    // 这个表达式并不会返回 nullptr，而是会传播构造函数的异常
    new(std::nothrow) FailingConstructor;
  } catch (std::runtime_error const& e) {
    std::cout << "Step 4: Caught an exception from the new(nothrow) expression." << std::endl;
    std::cout << "  Exception message: " << e.what() << std::endl;
  }
}
```

程序的输出将清晰地显示分配、构造、释放、捕获异常的完整流程，证明了 `new(nothrow)` 表达式本身也会因构造函数而抛出异常。

#### 总结

| 使用形式               | 内存分配失败的行为    | 构造函数失败的行为                      |
| :--------------------- | :-------------------- | :-------------------------------------- |
| `new T;`               | 抛出 `std::bad_alloc` | 抛出构造函数的异常 (并自动释放内存)     |
| `new(std::nothrow) T;` | 返回 `nullptr`        | **抛出构造函数的异常** (并自动释放内存) |

### 高级用法：组合对齐与 nothrow

当既需要不抛出异常的保证，又需要传递 C++17 的对齐要求时，需要在 `new` 表达式的括号中同时提供这两个参数。

#### 正确的 `new` 表达式语法

正确的语法是将 `std::align_val_t` 和 `std::nothrow` 对象作为 placement-new 的参数，按特定顺序传递：

```cpp
new (std::align_val_t{alignment}, std::nothrow) Type;
```

*   `std::align_val_t{alignment}`: 这是 C++17 引入的用于传递对齐值的类型。`alignment` 必须是一个有效的对齐值（通常是 2 的幂）。
*   `std::nothrow`: 这是一个全局 `const` 对象，类型为 `std::nothrow_t`。它作为一个标签 (tag)，告诉编译器去选择不抛出异常的 `operator new` 重载。
*   **参数顺序**: 标准规定，`std::align_val_t` 参数必须在 `std::nothrow_t` 参数之前。

#### 工作原理

当编译器遇到上述 `new` 表达式时，它会寻找并调用以下签名的替换函数（或标准库的默认实现）：

```cpp
void* operator new(std::size_t count, std::align_val_t al, const std::nothrow_t& tag) noexcept;
```

这个函数的契约是：

1.  它会尝试分配 `count` 字节的内存，并确保其起始地址满足 `al` 指定的对齐要求。
2.  如果分配成功，它返回一个指向该内存的有效指针。
3.  如果分配失败，它**必须返回 `nullptr`**，并且**不能抛出任何异常**（因此函数被标记为 `noexcept`）。

#### 完整代码示例

```cpp
#include <iostream>
#include <new>      // For std::nothrow, std::align_val_t
#include <cstdint>  // For std::uintptr_t

// 一个需要特殊对齐的结构体
struct AlignedData {
  // alignas 强制要求此类型的对象实例具有 64 字节的对齐
  alignas(64) char data[128];
};

int main() {
  constexpr std::size_t alignment = 64;
  std::cout << "Requesting alignment: " << alignment << " bytes." << std::endl;
  std::cout << "------------------------------------------" << std::endl;

  // 使用同时带有对齐和 nothrow 参数的 new 表达式
  AlignedData* ptr = new(std::align_val_t{alignment}, std::nothrow) AlignedData;

  // 1. 检查内存分配是否成功 (nothrow 的效果)
  if (ptr == nullptr) {
    std::cerr << "Memory allocation failed. The pointer is nullptr." << std::endl;
    return 1;
  }
  std::cout << "Memory allocation successful." << std::endl;

  // 2. 验证地址是否满足对齐要求 (align_val_t 的效果)
  auto address_value = reinterpret_cast<std::uintptr_t>(ptr);
  if (address_value % alignment == 0) {
    std::cout << "Address " << std::hex << address_value << std::dec
              << " is correctly aligned to " << alignment << " bytes." << std::endl;
  } else {
    std::cerr << "Error: Address " << std::hex << address_value << std::dec
              << " is NOT aligned to " << alignment << " bytes." << std::endl;
  }

  // 正常 delete 即可，编译器会自动匹配正确的释放函数
  delete ptr;
  std::cout << "Memory deallocated successfully." << std::endl;

  return 0;
}
```

#### 关于内存释放 (`delete`) 的重要说明

对于用 `new(std::align_val_t{...}, std::nothrow)` 创建的指针，**使用普通的 `delete` 即可**。C++ 运行时系统足够智能，能够自动调用匹配的、带有对齐参数的 `operator delete` 函数进行释放，确保分配和释放的对称性。

## 应用场景与代码示例

### 场景一：内存使用跟踪与泄漏检测

一个常见的用途是跟踪每一次内存分配和释放，以帮助调试内存问题。

```cpp
#include <cstdio>
#include <cstdlib>
#include <new>
#include <atomic>

// 用于统计当前分配的内存总量
static std::atomic<size_t> g_allocated_memory = 0;

// 抛出异常版本
void* operator new(std::size_t size) {
  void* ptr = std::malloc(size);
  if (!ptr) {
    throw std::bad_alloc{};
  }
  g_allocated_memory += size;
  printf("Allocated %zu bytes at %p. Total allocated: %zu\n", size, ptr, g_allocated_memory.load());
  return ptr;
}

// 释放函数（普通版本）
void operator delete(void* ptr) noexcept {
  // 注意：标准要求 delete 一个 nullptr 是安全的，所以 malloc/free 实现中无需检查
  // 但为了演示 sized deallocation 的匹配，我们这里不直接调用 free
  // 实际项目中，需要一种方式来获取大小，或者只依赖非 sized 版本
  // 此处我们无法知道 size，这是一个简化示例
  printf("Deallocating at %p. Cannot determine size here.\n", ptr);
  std::free(ptr);
}

// 带大小的释放函数（C++14+）
void operator delete(void* ptr, std::size_t size) noexcept {
  if (!ptr) return;
  g_allocated_memory -= size;
  printf("Deallocated %zu bytes at %p. Total allocated: %zu\n", size, ptr, g_allocated_memory.load());
  std::free(ptr);
}

// 数组版本也应成对实现
void* operator new[](std::size_t size) {
  return ::operator new(size);
}

void operator delete[](void* ptr) noexcept {
  ::operator delete(ptr);
}

void operator delete[](void* ptr, std::size_t size) noexcept {
  ::operator delete(ptr, size);
}

int main() {
  int* p1 = new int;
  double* p2 = new double;
  char* arr = new char[100];

  delete p1;
  delete p2;
  delete[] arr;

  printf("Final allocated memory: %zu\n", g_allocated_memory.load());
}
```

**分析**：

*   这个例子重载了 `operator new` 和 `operator delete`，并在其中加入了 `printf` 日志和 `g_allocated_memory` 计数器。
*   编译器在 `delete p1` 时，会知道 `p1` 指向一个 `int`，因此会优先调用 `operator delete(void*, std::size_t)` 的版本。
*   注意线程安全：全局变量 `g_allocated_memory` 使用 `std::atomic` 来保证多线程下的原子性操作。

### 场景二：实现一个简单的内存池

对于大量频繁分配和释放的小对象，使用内存池可以显著提升性能。内存池预先分配一大块内存，然后在其内部分配小块，避免了频繁的系统调用。

```cpp
#include <cstdio>
#include <cstdlib>
#include <new>
#include <mutex>

// 一个极简的固定大小块内存池
class FixedBlockPool {
public:
  FixedBlockPool() {
    // 预分配 1024 个 64 字节的块
    m_pool = static_cast<char*>(std::malloc(POOL_SIZE * BLOCK_SIZE));
    if (!m_pool) {
      throw std::bad_alloc{};
    }

    // 将所有块链接成一个空闲链表
    m_free_list = reinterpret_cast<Node*>(m_pool);
    Node* current = m_free_list;
    for (size_t i = 0; i < POOL_SIZE - 1; ++i) {
      current->next = reinterpret_cast<Node*>(m_pool + (i + 1) * BLOCK_SIZE);
      current = current->next;
    }
    current->next = nullptr;
  }

  ~FixedBlockPool() {
    std::free(m_pool);
  }

  void* allocate() {
    std::lock_guard<std::mutex> lock(m_mutex);
    if (!m_free_list) {
      return nullptr;
    }
    Node* block = m_free_list;
    m_free_list = block->next;
    return block;
  }

  void deallocate(void* ptr) {
    if (!ptr) return;
    std::lock_guard<std::mutex> lock(m_mutex);
    Node* block = static_cast<Node*>(ptr);
    block->next = m_free_list;
    m_free_list = block;
  }

  // 公开 BLOCK_SIZE 以便全局函数访问
  static constexpr size_t get_block_size() { return BLOCK_SIZE; }

private:
  static constexpr size_t BLOCK_SIZE = 64;
  static constexpr size_t POOL_SIZE = 1024;

  struct Node {
    Node* next;
  };

  char* m_pool = nullptr;
  Node* m_free_list = nullptr;
  std::mutex m_mutex;
};

FixedBlockPool g_pool; // 全局内存池实例

// === 单一对象版本 ===

// 只为小于等于 BLOCK_SIZE 的分配使用内存池
void* operator new(std::size_t size) {
  if (size <= FixedBlockPool::get_block_size() && size > 0) {
    if (void* ptr = g_pool.allocate()) {
      printf("Pool allocated %zu bytes at %p\n", size, ptr);
      return ptr;
    }
  }
  // 如果内存池耗尽、请求大小不符或大小为0，回退到系统分配
  void* ptr = std::malloc(size);
  if (!ptr && size > 0) throw std::bad_alloc{};
  printf("System (single) allocated %zu bytes at %p\n", size, ptr);
  return ptr;
}

void operator delete(void* ptr, std::size_t size) noexcept {
  if (!ptr) return;
  if (size <= FixedBlockPool::get_block_size() && size > 0) {
    printf("Pool deallocated %zu bytes at %p\n", size, ptr);
    g_pool.deallocate(ptr);
  } else {
    printf("System (single) deallocated %zu bytes at %p\n", size, ptr);
    std::free(ptr);
  }
}

void operator delete(void* ptr) noexcept {
  // 这个版本无法安全地判断大小，所以无法选择正确的释放方式。
  // 在真实的系统中，这通常意味着需要一种方法来从指针本身查找到元数据。
  // 在此示例中，我们假设 sized deallocation 总是可用，这个函数仅作备用。
  printf("Warning: Deallocating at %p without size info. Falling back to std::free.\n", ptr);
  std::free(ptr);
}

// === 数组版本 ===

void* operator new[](std::size_t size) {
  // 数组分配通常大小不一，不适合此处的固定大小内存池。
  // 因此，我们直接委托给系统的分配逻辑。
  // 一个简单的方法是直接调用我们已经实现的 `::operator new(size)`。
  printf("-> Array new[] redirects to single new\n");
  return ::operator new(size);
}

void operator delete[](void* ptr, std::size_t size) noexcept {
  // 与 `new[]` 对应，委托给单一对象的 `delete`。
  printf("-> Array delete[] (sized) redirects to single delete\n");
  ::operator delete(ptr, size);
}

void operator delete[](void* ptr) noexcept {
  // 与 `new[]` 对应，委托给单一对象的 `delete`。
  printf("-> Array delete[] (unsized) redirects to single delete\n");
  ::operator delete(ptr);
}

struct MyData {
  char data[48];
};

int main() {
  printf("--- Creating single object MyData ---\n");
  MyData* d1 = new MyData(); // 应该从内存池分配 (48 bytes <= 64)

  printf("\n--- Creating array of 1000 ints ---\n");
  int* p = new int[1000];    // 应该从系统分配 (4000 bytes > 64)

  printf("\n--- Deleting array ---\n");
  delete[] p;

  printf("\n--- Deleting single object ---\n");
  delete d1;

  return 0;
}
```

**分析**：

*   这个例子展示了如何根据分配大小来选择不同的分配策略。
*   **线程安全**是必须考虑的。全局内存池是共享资源，所有操作都需要互斥锁 (`std::mutex`) 保护。
*   `delete` 的匹配问题：当重载了 `operator delete` 后，如何判断一个指针是来自内存池还是系统 `malloc` 是一个挑战。带大小的版本 (Sized Deallocation) 极大地简化了这个问题。如果没有 `size` 信息，通常需要在分配的内存旁存储元数据，但这会增加内存开销和实现的复杂性。

## 实现时的关键注意事项

替换函数的强大能力伴随着巨大的责任。不正确的实现会导致难以调试的崩溃、性能问题或未定义行为。

### 1. 禁止在替换函数内部使用可能导致内存分配的 C++ 特性

这是一个最致命的陷阱。在 `operator new` 的实现中，绝对不能直接或间接地调用任何可能再次触发 `operator new` 的代码，否则会引发**无限递归**，最终导致栈溢出。

**危险的操作包括**：

*   抛出需要动态分配内存的异常对象 (`throw std::runtime_error("...")`)。
*   使用标准库容器，如 `std::vector`、`std::string`。
*   使用 I/O Streams，如 `std::cout`，因为它们内部可能会分配缓冲区。

**安全的替代方案**：

*   日志记录：使用 C 风格的 I/O，如 `printf` 或直接使用系统调用 `write`。
*   错误处理：对于抛出异常的版本，应 `throw std::bad_alloc{}`，这是一个特例，标准保证它不会导致递归分配。对于 `nothrow` 版本，返回 `nullptr`。
*   数据结构：如果内部需要数据结构，必须使用预分配的或基于栈的存储。

### 2. 尊重 `new` 表达式的契约

如上文 “异常保证” 部分所述，`new` 表达式与其调用的 `operator new` 之间存在严格的契约。

*   **对于抛出版本的 `operator new`**：在分配失败时，**必须**抛出 `std::bad_alloc` 异常。**绝不能**返回 `nullptr`。如果返回了 `nullptr`，`new` 表达式会认为分配成功，并继续在 `nullptr` 上调用构造函数，这会立即导致未定义行为。

    ```cpp
    // 错误的替换函数实现，违反了契约
    void* operator new(std::size_t size) {
      void* p = std::malloc(size);
      // 错误！对于抛出版本，失败时不应返回 nullptr，而应抛出异常。
      return p; // 如果 malloc 失败，p 为 nullptr，将导致严重问题。
    }

    // 正确的实现
    void* operator new(std::size_t size) {
      void* p = std::malloc(size);
      if (!p) {
        throw std::bad_alloc{}; // 正确：失败时抛出异常
      }
      return p;
    }
    ```

*   **对于 `nothrow` 版本的 `operator new`**：在分配失败时，**必须**返回 `nullptr`，并且函数本身**不能**抛出异常（应标记为 `noexcept`）。

### 3. 保证线程安全

全局分配/释放函数是整个程序共享的资源。在多线程环境中，它们必须是线程安全的。任何对共享数据（如内存池链表、计数器）的访问都必须通过互斥锁或其它同步原语进行保护。

### 4. 正确处理 `nullptr` 和 `0` 字节分配

*   `operator delete(nullptr)` 和 `operator delete[](nullptr)` 必须是无害的空操作。
*   `operator new(0)` 的行为是实现定义的。一个常见的做法是返回一个合法、唯一但不可解引用的指针。后续对该指针的 `delete` 必须能被正确处理。

### 5. 成对实现 `new` 和 `delete`

如果重载了 `operator new`，就必须重载对应的 `operator delete`。如果分配策略很特殊（例如来自内存池），那么释放策略也必须与之对应，否则会导致内存损坏或泄漏。**特别地，如果替换了 `new`，就必须同时替换 `new[]`**，否则会导致链接错误。

### 6. 理解带大小释放 (Sized Deallocation) 的不确定性

这是一个非常关键的实践要点。虽然 C++14 引入了带大小的 `operator delete`，但它的调用时机并非总是如预期那样。

*   **对于单一对象 `delete p;`**：编译器通常**会**调用带大小的版本 `operator delete(void*, size_t)`。因为 `p` 的类型在编译时已知，`sizeof(*p)` 是一个常量，传递它是一个简单且有效的优化。

*   **对于数组 `delete[] p;`**：编译器通常**不会**调用带大小的版本 `operator delete[](void*, size_t)`，而是调用不带大小的版本 `operator delete[](void*)`。
    *   **原因**：当执行 `new T[n]` 时，运行时系统通常会在分配的内存块头部存储一个 “数组 Cookie” （通常是数组的元素数量 `n`）。当 `delete[] p` 时，系统通过读取这个 Cookie 来确定要调用多少个析构函数。底层的 `free` 函数本身已经知道要释放的内存块的总大小，因此再次计算并传递这个大小被主流编译器 （GCC, Clang, MSVC） 认为是不必要的开销。
    *   **例外**：在某些特定情况下，例如删除一个**带有虚析构函数的类**的数组时，编译器可能会为了处理复杂的析构逻辑而选择调用带大小的版本。

*   **实践准则**：**必须始终提供一个功能完备的、不带大小的 `operator delete` 和 `operator delete[]` 实现。** 它们是所有情况下的最终回退方案。绝对不能依赖带大小的版本总会被调用，尤其是在处理数组时。这直接影响到自定义分配器的设计：如果释放时需要知道大小，那么不带大小的 `delete` 版本必须有办法从指针本身推断出大小（例如，通过在分配的内存块中存储元数据）。

### 7. C++20 及以后的 `destroying` deallocation

C++20 引入了 `destroying` 版本的 `operator delete`，它将析构和内存释放合并为一步。这是一个更高级的优化，允许分配器在知道对象类型后进行更智能的回收。如果目标环境是 C++20，也应考虑实现这些版本以获得最佳性能。

## 结论

C++ 的替换函数是一个强大但危险的高级特性。它赋予了开发者对程序内存生命周期的终极控制权。通过全局重载 `operator new` 和 `operator delete`，可以实现高度优化的自定义内存池、精密的内存调试工具或与特殊硬件的内存接口。

然而，使用此功能时必须格外小心。开发者需要全权负责线程安全、严格遵守 `new` 表达式的契约、避免递归分配、正确处理各种边界情况，并确保分配与释放逻辑的配对。在大多数应用中，标准库的默认分配器已经足够高效和健壮。只有在性能分析表明默认分配器确实成为瓶颈，或者有特殊内存管理需求的场景下，才应考虑编写替换函数。这是一种“高手”工具，用得好能创造奇迹，用得不好则会带来灾难。

---
