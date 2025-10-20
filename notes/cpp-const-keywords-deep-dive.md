# C++ 的常量之旅：从 `const` 到 `consteval`

在 C++ 的世界里，“常量” 是一个多层次、多维度的概念。它早已超越了 “不可修改的变量” 这一简单定义。随着 C++ 标准的演进，从最初的 `const` 到 C++11 的 `constexpr`，再到 C++20 的 `consteval` 和 `constinit`，C++ 为开发者提供了一套强大而精细的工具集，用于在不同维度——运行时不变性、编译期求值能力、强制编译期求值——上表达和约束代码。

理解这些关键字之间的差异与联系，对于编写安全、高效、现代的 C++ 代码至关重要。本文将系统性地剖析这些 `const` 家族成员，阐明它们的含义、用法和设计哲学。

## 1. 基石：`const` - 运行时的不变性

`const` 是 C++ 中最古老、最基础的常量关键字。它的核心语义是 **“只读” (read-only)**，承诺一个对象在初始化之后，其状态不会再被修改。这是一种**运行时**的约束。

### 1.1 `const` 修饰变量

当 `const` 修饰一个变量时，该变量必须在定义时进行初始化，并且之后不能再被赋值。

```cpp
const int MAX_SIZE = 1024;
// MAX_SIZE = 2048; // 编译错误：不能给 const 变量赋值

// const 变量的初始化可以来自一个运行时计算
int get_value() { return 42; }
const int RUNTIME_CONST = get_value(); // 合法，在运行时初始化
```

### 1.2 `const` 与指针

`const` 与指针的结合是 C++ 的一个经典主题，它能精确地控制指针本身和指针所指向数据的可变性。

*   **指向常量的指针 (Pointer to `const`)**

  ```cpp
  // 不能通过指针 p 修改所指向的值，但指针 p 本身可以指向其它地址
  const int* p;
  int x = 10;
  p = &x;
  // *p = 20; // 编译错误
  int y = 30;
  p = &y; // 合法
  ```

  记忆技巧：“`const` 在 `*` 左边，常量的是指针指向的内容”。

*   **常量指针 (`const` Pointer)**

  ```cpp
  int x = 10;
  // 指针 p 本身是常量，必须在定义时初始化，之后不能再指向其它地址
  // 但可以通过 p 修改其所指向的值
  int* const p = &x;
  *p = 20; // 合法
  int y = 30;
  // p = &y; // 编译错误
  ```

  记忆技巧：“`const` 在 `*` 右边，常量的是指针本身”。

*   **指向常量的常量指针**

  ```cpp
  int x = 10;
  // 指针 p 和其指向的内容都是常量
  const int* const p = &x;
  // *p = 20; // 编译错误
  int y = 30;
  // p = &y; // 编译错误
  ```

### 1.3 `const` 与引用

引用比指针简单。对 `const` 的引用（常量引用）意味着不能通过该引用修改其引用的对象。

```cpp
int x = 10;
const int& ref = x;
// ref = 20; // 编译错误
```

常量引用是函数参数传递中最常用的技术之一，它可以避免不必要的拷贝，同时保证函数内部不会意外修改传入的实参。

```cpp
#include <string>
#include <vector>

// 高效且安全：通过常量引用接收，避免拷贝，且保证不修改原始数据
void process_data(const std::string& data) {
  // ...
}
```

### 1.4 `const` 成员函数

在类定义中，`const` 可以修饰成员函数，它位于函数签名之后。

```cpp
class MyClass {
public:
  int get_value() const {
    // m_value = 10; // 编译错误：不能在 const 成员函数中修改成员变量
    return m_value;
  }

  void set_value(int value) {
    m_value = value;
  }

private:
  int m_value = 0;
};
```

`const` 成员函数是对调用它的对象的一种承诺：**“此函数不会修改对象的数据成员”**。

在 `const` 成员函数内部：

*   `this` 指针的类型是 `const MyClass*`。
*   不能修改非 `static` 成员变量。
*   只能调用其它的 `const` 成员函数。

#### `mutable` 关键字

`mutable` 是 `const` 的一个例外。如果一个成员变量被 `mutable` 修饰，那么它可以在 `const` 成员函数中被修改。这通常用于实现缓存、计数器等不影响对象逻辑状态的成员。

```cpp
class Cache {
public:
  std::string get_data() const {
    if (!m_is_valid) {
      // 合法：因为 m_data 和 m_is_valid 是 mutable 的
      m_data = "Computed Data";
      m_is_valid = true;
    }
    return m_data;
  }
private:
  mutable std::string m_data;
  mutable bool m_is_valid = false;
};
```

## 2. 演进：[`constexpr`](https://en.cppreference.com/w/cpp/language/constexpr.html) - 编译期求值的能力

`constexpr` (Constant Expression) 是 C++11 引入的，它将常量的概念从**运行时**扩展到了**编译期**。`constexpr` 的核心语义是 **“有能力在编译期被求值”**。

它有两种主要用法：修饰变量和修饰函数。

### 2.1 `constexpr` 变量

`constexpr` 变量是真正的**编译期常量**。它的值必须在编译阶段就能确定下来。

```cpp
constexpr int COMPILE_TIME_CONST = 42; // OK
constexpr int ANOTHER_CONST = COMPILE_TIME_CONST * 2; // OK

int get_value() { return 5; }
// constexpr int val = get_value(); // 编译错误：get_value() 的值在运行时才能确定
```

`constexpr` 变量自带 `const` 属性，它不仅是编译期常量，在运行时也是不可修改的。

### 2.2 `constexpr` 函数/构造函数

这是 `constexpr` 最强大的地方。`constexpr` 函数具备**双重身份**：

1.  **当使用编译期常量作为参数调用时**：它可以在编译期执行，并产生一个编译期常量结果。
2.  **当使用运行时变量作为参数调用时**：它的行为就像一个普通函数。并且，由于标准规定 `constexpr` 函数是隐式 `inline` 的，它具备了内联函数的所有链接特性，这使得编译器极有可能对其进行内联优化，以消除函数调用开销。

```cpp
// 一个 constexpr 函数
constexpr int factorial(int n) {
  return n <= 1 ? 1 : n * factorial(n - 1);
}

void usage() {
  // 场景 1: 在编译期求值
  // 因为结果用于初始化一个 constexpr 变量，编译器会强制在编译期计算 factorial(5)
  constexpr int val1 = factorial(5); // val1 的值是 120，在编译时计算完成

  // 场景 2: 在运行时求值
  int x = 6;
  // 因为参数 x 是一个运行时变量，所以 factorial(x) 会在运行时被调用
  int val2 = factorial(x); // val2 的值是 720，在运行时计算
}
```

`constexpr` 函数的限制（随着 C++ 版本迭代，限制越来越宽松）：

*   函数体必须相对简单，不能有 `static` 变量、`try-catch` 等。
*   C++11 中，函数体只能包含一个 `return` 语句。
*   C++14 及以后，限制大大放宽，允许使用局部变量、循环和分支语句。

`constexpr` 的主要用途是元编程、创建编译期查找表、进行静态断言等，它极大地提升了 C++ 在编译期进行计算的能力。

## 3. 极致：[`consteval`](https://en.cppreference.com/w/cpp/language/consteval.html) - 强制的编译期求值

C++20 引入了 `consteval`，它比 `constexpr` 更为严格。`consteval` 用于声明一个**立即函数 (Immediate Function)**。

核心语义：**“必须在编译期求值”**。

`consteval` 函数是纯粹为编译器编写的工具函数。对它的任何调用都必须产生一个编译期常量。任何试图在运行时调用它的行为都会导致编译失败。

```cpp
// 一个 consteval 函数
consteval int square(int n) {
  return n * n;
}

void usage() {
  // 合法：在编译期调用
  constexpr int val1 = square(10); // 编译期计算出 100

  // 编译错误：尝试在运行时调用
  int x = 5;
  // int val2 = square(x); // 编译错误！
                           // 错误信息: call to consteval function 'square' is not a constant expression
}
```

### `consteval` vs. `constexpr` 函数

| 特性         | `constexpr` 函数                              | `consteval` 函数           |
| :----------- | :-------------------------------------------- | :------------------------- |
| **求值时机** | **灵活**: **可以**在编译期，也**可以**在运行时 | **严格**: **必须**在编译期 |
| **意图**     | “我**有能力**在编译期运行”                    | “我**只能**在编译期运行”   |
| **用途**     | 通用的、可在两阶段运行的函数                  | 纯粹的编译期元编程工具     |

`consteval` 的一个典型应用场景是与 `std::source_location` (C++20) 结合，用于在编译期捕获调用点的源代码信息（如文件名、行号）。这些信息在运行时是无意义的，因此强制在编译期执行是符合逻辑的。

## 4. 保障：[`constinit`](https://en.cppreference.com/w/cpp/language/constinit.html) - 编译期的初始化保障

`constinit` 也是 C++20 引入的。它不是一个类型修饰符，而是一个**初始化断言**。

核心语义：**“我断言这个具有静态或线程存储期的变量，必须通过常量表达式进行初始化”**。

它主要解决一个长期存在的问题：**静态初始化顺序问题 (Static Initialization Order Fiasco)**。当不同编译单元中的全局变量存在初始化依赖时，它们的初始化顺序是不确定的，可能导致灾难性后果。

`constinit` 通过强制**静态初始化**（在程序加载前完成，零开销）来避免**动态初始化**（在 `main` 函数执行前或首次使用时进行），从而消除顺序依赖问题。

```cpp
// 假设 get_magic_number() 是一个 constexpr 函数
constexpr int get_magic_number() { return 42; }

// 使用 constinit
constinit int global_magic_number = get_magic_number(); // OK

// 编译错误示例
int get_runtime_number() { return 10; }
// constinit int another_number = get_runtime_number(); // 编译错误！
                                                        // 初始化器不是一个常量表达式
```

### `constinit` vs. `constexpr` 全局变量

|            | `constexpr int G = 10;`    | `constinit int G = 10;`            |
| :--------- | :------------------------- | :--------------------------------- |
| **常量性** | `G` 是 `const` 的，不可修改 | `G` 默认不是 `const`，**可以修改**  |
| **初始化** | 强制常量初始化             | 强制常量初始化                     |
| **用途**   | 定义一个全局的编译期常量   | 安全地初始化一个**可变**的全局状态 |

```cpp
// global_counter 必须在编译期被初始化为 0，
// 但它在程序运行期间是可变的。
// 这避免了它与其它全局变量的初始化顺序问题。
constinit int global_counter = 0;

void increment_counter() {
  global_counter++; // 合法
}
```

## 5. 总结与选择指南

为了清晰地理解和选择合适的关键字，可以参考下表：

| 关键字 | 核心语义 | 应用对象 | 求值/约束时机 | 是否可变 |
| :-------------- | :-------------------------------------- | :------------------------- | :---------------- | :------------ |
| **`const`**     | 只读                                    | 变量, 指针, 引用, 成员函数 | 运行时            | 否 (初始化后) |
| **`constexpr`** | 编译期常量 (变量) / 编译期可求值 (函数) | 变量, 函数, 构造函数       | 编译期 & 运行时   | 否 (变量)     |
| **`consteval`** | 必须在编译期求值                        | 函数                       | 仅编译期          | -             |
| **`constinit`** | 必须进行常量初始化                      | 静态/线程存储期变量        | 编译期 (仅初始化) | 是 (默认)     |

### 选择指南

1.  **当需要一个运行时不可变的对象时**：
    *   使用 `const`。这是最通用的选择。

2.  **当需要一个在编译期就确定其值的真正常量时**（例如数组大小、模板参数）：
    *   使用 `constexpr` 变量。

3.  **当需要编写一个既能用于编译期计算，也能在运行时使用的工具函数时**：
    *   使用 `constexpr` 函数。例如，一个数学库函数 `sqrt`。

4.  **当需要编写一个纯粹的元编程或代码生成工具，其逻辑只在编译期有意义时**：
    *   使用 `consteval` 函数。例如，解析格式化字符串。

5.  **当需要安全地初始化一个全局或静态变量，以避免初始化顺序问题时**（特别是这个变量后续需要被修改）：
    *   使用 `constinit`。

通过精准地运用 `const` 家族的这些关键字，可以编写出意图更明确、更安全、性能更高的 C++ 代码，充分利用 C++ 强大的编译期计算能力，将更多的错误检查和计算任务从运行时提前到编译期。

---
