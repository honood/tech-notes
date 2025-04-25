# C++ 中 `-9'223'372'036'854'775'808` 为何输出为正数？深入理解整型字面量与一元负号

在使用 C++ 编程时，有时会遇到一些看似反直觉的结果。一个典型的例子是执行以下代码：

```cpp
#include <iostream>

int main() {
  // 针对 long long 的情况
  std::cout << -9'223'372'036'854'775'808 << std::endl;
  return 0;
}
```

一个常见的期望是这段代码输出 `long long int` 类型的最小值，即 `-9223372036854775808`。然而，实际的输出结果却是其绝对值：

```
9223372036854775808
```

这个结果可能令人困惑。为何取负操作似乎 “失效” 了？理解这个现象需要深入探究 C++ 如何处理[整数（整型）字面量](https://en.cppreference.com/w/cpp/language/integer_literal)以及[一元负号运算符 (`-`)](https://en.cppreference.com/w/cpp/language/operator_arithmetic#Unary_arithmetic_operators) 的规则。

## 核心：表达式的解析顺序

关键在于理解 C++ 编译器如何解析表达式 `-9'223'372'036'854'775'808`。它并**不是**一个单一的负数字面量。实际上，它由两部分组成：

1.  **一元负号运算符 (`-`)**
2.  **整数（整型）字面量 (`9'223'372'036'854'775'808`)**

编译器首先确定整数字面量 `9'223'372'036'854'775'808` 的类型，然后才将一元负号运算符应用于该确定类型的值。

## 第一步：确定字面量 `9'223'372'036'854'775'808` 的类型

C++ 标准规定了如何为不带[后缀（如 `L`, `LL`, `U`, `ULL` 等）](https://en.cppreference.com/w/cpp/language/integer_literal)的十进制整数字面量确定类型。编译器会按以下顺序尝试找到能容纳该值的**最小**的有符号类型：

1.  `int`
2.  `long int`
3.  `long long int`

考虑字面量 `9'223'372'036'854'775'808`。这个值等于 2<sup>63</sup>。

在大多数现代 64 位系统上：

*   `int` 通常是 32 位。
*   `long int` 通常是 64 位（或者有时是 32 位）。
*   `long long int` **保证**至少是 64 位。

对于 64 位的 `long long int`，其表示范围通常是 [-2<sup>63</sup>, 2<sup>63</sup> - 1]，即 [-9'223'372'036'854'775'808, 9'223'372'036'854'775'807]。

比较字面量 `9'223'372'036'854'775'808` (2<sup>63</sup>) 和 `long long int` 的最大值 `9'223'372'036'854'775'807` (2<sup>63</sup> - 1)，可以发现：

```
9'223'372'036'854'775'808 > 9'223'372'036'854'775'807
```

这意味着这个字面量的值**超出了** `long long int` 所能表示的最大正数范围。

根据 C++ 标准，如果一个十进制整数字面量无法被任何有符号整数类型容纳，编译器会继续尝试将其视为**无符号整数类型**。查找顺序通常是：

1.  `unsigned int`
2.  `unsigned long int`
3.  `unsigned long long int`

对于 64 位的 `unsigned long long int`，其表示范围通常是 [0, 2<sup>64</sup> - 1]，即 [0, 18'446'744'073'709'551'615]。

字面量 `9'223'372'036'854'775'808` 显然在这个范围内。

因此，编译器最终确定整数字面量 `9'223'372'036'854'775'808` 的类型是 `unsigned long long int`。

## 验证字面量类型推断：实验方法

理论规则描述了编译器如何推断整数字面量的类型。在特定的编译环境中，可以通过一些 C++ 特性来实验性地验证这些规则，确认某个字面量实际被推断为何种类型。以字面量 `2'147'483'648` (2<sup>31</sup>) 为例：

### 方法 1: 使用 `decltype` 和 `typeid`

`decltype(expression)` 获取表达式的编译时类型，`typeid(type).name()` 返回表示该类型名称的字符串（可能经过编译器修饰）。

```cpp
#include <iostream>
#include <typeinfo>

int main() {
  // 获取字面量 2'147'483'648 的推断类型信息
  std::type_info const& literal_type = typeid(decltype(2'147'483'648));
  std::cout << "Type name of literal 2'147'483'648: " << literal_type.name() << std::endl;

  // 对比标准类型名称
  std::cout << "Type name of int:             " << typeid(int).name() << std::endl;
  std::cout << "Type name of unsigned int:    " << typeid(unsigned int).name() << std::endl;
  std::cout << "Type name of long int:        " << typeid(long int).name() << std::endl;
  std::cout << "Type name of long long int:   " << typeid(long long int).name() << std::endl;
  return 0;
}
```

*   **解读：** 比较第一行输出与后面标准类型的输出。在 `long int` 为 64 位的系统（如 64 位 Linux/macOS）上，预期输出通常与 `long int` 的名称匹配。在 `long int` 为 32 位而 `long long int` 为 64 位的系统（如 64 位 Windows）上，预期输出通常与 `long long int` 的名称匹配。注意 `name()` 返回值是实现定义的。

### 方法 2: 使用 `sizeof`

`sizeof` 返回类型或表达式占用的字节数，可辅助判断。

```cpp
#include <iostream>

int main() {
  std::cout << "Size of literal 2'147'483'648: " << sizeof(decltype(2'147'483'648)) << " bytes" << std::endl;
  // 对比标准类型大小
  std::cout << "Size of int:             " << sizeof(int) << " bytes" << std::endl;
  std::cout << "Size of long int:        " << sizeof(long int) << " bytes" << std::endl;
  std::cout << "Size of long long int:   " << sizeof(long long int) << " bytes" << std::endl;
  return 0;
}
```

*   **解读：** 如果 `int` 占 4 字节而字面量占 8 字节，则排除 `int`。结合 `long int` 和 `long long int` 的大小信息，可以推断类型。

### 方法 3: 使用 `static_assert` 和 `<type_traits>` (C++11+)

`static_assert` 可以在编译时验证类型是否符合预期。

```cpp
#include <type_traits> // for std::is_same

int main() {
  // 编译时验证字面量 2'147'483'648 的类型
  // 示例：假设在 long int 为 64 位的系统上
  static_assert(std::is_same<decltype(2'147'483'648), long int>::value,
                "Expect 2'147'483'648 to be long int on this system");

  // 示例：假设在 long int 为 32 位，long long 为 64 位的系统上
  // static_assert(std::is_same<decltype(2'147'483'648), long long int>::value,
  //               "Expect 2'147'483'648 to be long long int on this system");

  volatile int dummy = 0; // 防止优化
  return 0;
}
```

*   **解读：** 根据目标平台调整断言。如果编译成功，则假设正确；如果编译失败并显示消息，则假设错误。

### 应用到 `-9...8` 的字面量

这些方法同样适用于文章开头的字面量 `9'223'372'036'854'775'808`。由于其值大于 `long long` 的最大正值，预期会被推断为 `unsigned long long`。例如：

```cpp
#include <type_traits>

static_assert(std::is_same<decltype(9'223'372'036'854'775'808), unsigned long long int>::value,
              "Literal 9...8 should be unsigned long long int");
```

这些实验方法有助于在具体环境中确认编译器的行为，加深对类型推断规则的理解。

## 第二步：应用一元负号运算符 (`-`)

确定了字面量的类型后，分析表达式 `-9'223'372'036'854'775'808`。这实际上是对一个类型为 `unsigned long long int`、值为 2<sup>63</sup> 的数应用一元负号。

C++ 标准对无符号类型应用一元负号有明确规定：

*   结果的类型**仍然是该无符号类型**（本例中是 `unsigned long long int`）。
*   结果的值是通过**模运算 ([Modular Arithmetic](https://en.wikipedia.org/wiki/Modular_arithmetic))** 计算得出的。这里的 “模运算” 指的是计算机底层处理固定位数无符号整数运算的基础原理：
    *   一个 `n` 位的无符号整数可以表示 0 到 2<sup>n</sup> - 1 共 2<sup>n</sup> 个值。运算结果超出此范围时会发生 “回绕 (wrap-around)”，所有结果都等效于**模 2<sup>n</sup>**。
    *   一元负号 `-operand` 在此模算术体系中寻求 `operand` 的**加法逆元**，即一个值 `y` 使得 `(operand + y) mod 2^n = 0`。
    *   考虑表达式 `operand + (2^n - operand)`，其数学结果是 `2^n`。在模 2<sup>n</sup> 下，`(operand + (2^n - operand)) mod 2^n = 2^n mod 2^n = 0`。这表明 `2^n - operand` 正是 `operand` 在模 2<sup>n</sup> 意义下的加法逆元。
    *   当 `operand` 为 0 时，`2^n - 0 = 2^n`，模 2<sup>n</sup> 结果为 0。当 `operand` 在 [1, 2<sup>n</sup> - 1] 范围内时，`2^n - operand` 的值也在 [1, 2<sup>n</sup> - 1] 范围内，其模 2<sup>n</sup> 的结果就是自身。
*   因此，C++ 标准规定，一元负号应用于无符号操作数 `operand` 时，其结果值等于 **2<sup>n</sup> - operand**，其中 `n` 是该无符号类型所占的位数（对于 `unsigned long long int` 通常是 64）。这个规则直接体现了模 2<sup>n</sup> 算术中加法逆元的计算，并确保结果仍在无符号类型的表示范围内。

计算过程如下：

1.  结果 = 2<sup>64</sup> - 9'223'372'036'854'775'808
2.  结果 = 2<sup>64</sup> - 2<sup>63</sup>
3.  结果 = (2 * 2<sup>63</sup>) - (1 * 2<sup>63</sup>)
4.  结果 = (2 - 1) * 2<sup>63</sup>
5.  结果 = 1 * 2<sup>63</sup>
6.  结果 = 2<sup>63</sup> = 9'223'372'036'854'775'808

所以，表达式 `-9'223'372'036'854'775'808` 的最终计算结果是值 `9'223'372'036'854'775'808`，并且其类型是 `unsigned long long int`。

## 第三步：`std::cout` 输出

最后，`std::cout` 接收到这个类型为 `unsigned long long int`、值为 `9'223'372'036'854'775'808` 的参数。`std::cout` 会根据参数的实际类型（无符号长长整型）来选择合适的重载函数，并将其十进制值输出到标准输出流。

因此，屏幕上打印出了 `9223372036854775808`。

## 实例分析：应用到 `int` 和 `unsigned int`

上述原则同样适用于其它整数类型，如下面的代码示例所示。

**前提假设:**

*   假设系统中的 `int` 是 32 位有符号整数，范围是 [-2<sup>31</sup>, 2<sup>31</sup>-1]。
*   假设系统中的 `unsigned int` 是 32 位无符号整数，范围是 [0, 2<sup>32</sup>-1]。
*   关键数字：`2'147'483'648` 等于 2<sup>31</sup>。

**代码及解释:**

```cpp
#include <iostream>
#include <climits> // 用于对照 INT_MIN, INT_MAX 等
#include <limits>  // 用于对照 std::numeric_limits<int>::min(), std::numeric_limits<int>::max() 等

int main() {
  unsigned int x = 2'147'483'648;
  // 1. 初始化 x:
  //    - 字面量 `2'147'483'648` (2^31) 大于 `int` 最大值。
  //    - 它会被推断为 `long int` (若 64 位) 或 `long long int` (若 long 32 位)。设此类型为 T。
  //    - 将类型为 T、值为 2'147'483'648 的右值赋给 `unsigned int x`。
  //    - 发生隐式类型转换 T -> unsigned int。值 2'147'483'648 在 `unsigned int` 范围内。
  //    - x 被成功初始化为 2'147'483'648。
  std::cout << " x = " << x << std::endl;
  // 2. 输出 x:
  //    - 预期输出: x = 2147483648
  std::cout << "-x = " << -x << std::endl;
  // 3. 计算并输出 -x:
  //    - 对 `unsigned int` 类型的 `x` (值为 2^31) 应用一元负号 `-`。
  //    - 结果类型仍为 `unsigned int`，值等于 2^32 - x = 2^32 - 2^31 = 2^31 = 2'147'483'648。
  // 4. 输出 -x:
  //    - 预期输出: -x = 2147483648

  int y = 2'147'483'648;
  // 5. 初始化 y:
  //    - 字面量 `2'147'483'648` 的类型为 T (`long` 或 `long long`)。
  //    - 将类型为 T、值为 2'147'483'648 的右值赋给 `int y`。
  //    - 值 2'147'483'648 超出 `int` 的范围。
  //    - 发生从 T 到 int 的隐式转换，行为是 “实现定义 (implementation-defined)”。
  //    - 常见情况（二进制补码）下，位模式被截断或重新解释，y 可能得到 `INT_MIN` (-2'147'483'648)。编译器通常会发出警告。
  std::cout << " y = " << y << std::endl;
  // 6. 输出 y:
  //    - 预期输出: y = -2147483648 (常见实现定义结果)
  std::cout << "-y = " << -y << std::endl;
  // 7. 计算并输出 -y:
  //    - 对 `int` 类型的 `y` (值为 `INT_MIN`, -2^31) 应用一元负号 `-`。
  //    - 对有符号最小值取负导致算术溢出，是 “未定义行为 (undefined behavior)”。
  //    - 在许多常见平台上，作为未定义行为的一种表现，结果可能仍是 `INT_MIN` (-2'147'483'648)。
  // 8. 输出 -y:
  //    - 预期输出: -y = -2147483648 (常见未定义行为结果)

  int z = -2'147'483'648;
  // 9. 初始化 z:
  //    - 计算表达式 `-2'147'483'648` 并赋值给 `z`。
  //    - 字面量 `2'147'483'648` 的类型为 T (`long` 或 `long long`)。
  //    - 对类型为 T、值为 2'147'483'648 的数应用一元负号 `-`。结果是类型为 T 的值 -2'147'483'648。
  //    - 将类型为 T、值为 -2'147'483'648 的右值赋给 `int z`。
  //    - 值 -2'147'483'648 恰好在 32 位 `int` 的范围内 (等于 `INT_MIN`)。
  //    - 赋值是良定义的，z 被初始化为 -2'147'483'648。
  std::cout << " z = " << z << std::endl;
  // 10. 输出 z:
  //    - 预期输出: z = -2147483648
  std::cout << "-z = " << -z << std::endl;
  // 11. 计算并输出 -z:
  //    - 对 `int` 类型的 `z` (值为 `INT_MIN`) 应用一元负号 `-`。
  //    - 与 `-y` 情况相同，未定义行为。
  //    - 常见结果是 `-2'147'483'648`。
  // 12. 输出 -z:
  //    - 预期输出: -z = -2147483648 (常见未定义行为结果)

  return 0;
}
```

**示例代码预期输出总结:**

```
 x = 2147483648
-x = 2147483648
 y = -2147483648
-y = -2147483648
 z = -2147483648
-z = -2147483648
```

这个示例进一步强化了几个关键点：字面量的类型推断（优先有符号，可能升级到 `long` 或 `long long`）、赋值时的隐式类型转换、无符号数取负的模运算规则、超出范围赋值给有符号类型的实现定义行为，以及对有符号最小值取负的未定义行为。

## 总结与启示

这个现象的根源在于 C++ 对整数字面量类型推断的规则以及对无符号数应用一元负号的特殊处理（基于模 2<sup>n</sup> 的算术）。`-9'223'372'036'854'775'808`（以及示例中的 `-2'147'483'648` 在其初始化 `z` 的上下文中）不是一个单一的负数字面量，而是由一元负号运算符作用于一个（可能被推断为更大或无符号类型的）正数字面量构成的表达式。

这种行为虽然符合标准，但可能违反直觉。好的编译器通常会在此类情况（如对无符号数取负、有符号溢出、实现定义的转换）下发出警告，提示可能存在潜在的问题。

## 如何正确表示整数类型的最小值？

若确实需要在代码中表示或使用整数类型的最小值（如 `long long int` 或 `int` 的最小值），存在以下几种更安全、更清晰的方法：

1.  **使用标准库常量（推荐）：**
    ```cpp
    #include <iostream>
    #include <limits> // 或者 #include <climits>

    int main() {
      // C++11 及以后推荐使用 <limits>
      long long min_ll = std::numeric_limits<long long>::min();
      int min_int = std::numeric_limits<int>::min();
      std::cout << "LLONG_MIN: " << min_ll << std::endl;
      std::cout << "INT_MIN:   " << min_int << std::endl;

      // C 风格，来自 <climits>
      // long long min_ll_c = LLONG_MIN;
      // int min_int_c = INT_MIN;
      // std::cout << "LLONG_MIN (climits): " << min_ll_c << std::endl;
      // std::cout << "INT_MIN (climits):   " << min_int_c << std::endl;
      return 0;
    }
    ```

2.  **通过计算得到（注意后缀和类型）：**
    ```cpp
    #include <iostream>

    int main() {
      // 对 long long: -(2^63 - 1) - 1
      // 必须使用 LL 后缀确保 9'223'372'036'854'775'807 被视为 long long
      long long min_ll = -9'223'372'036'854'775'807LL - 1LL;
      std::cout << "LLONG_MIN (calc): " << min_ll << std::endl;

      // 对 int (假设 32 位): -(2^31 - 1) - 1
      // 字面量 2'147'483'647 在 int 范围内，无需后缀
      int min_int = -2'147'483'647 - 1;
      std::cout << "INT_MIN (calc):   " << min_int << std::endl;
      return 0;
    }
    ```
    在计算中，使用类型后缀（如 `LL`）或确保操作数在目标类型的范围内，可以避免因类型推断错误导致的问题。

理解这些底层规则和验证方法有助于编写更健壮、更可预测的 C++ 代码，并能更好地解读编译器的警告信息。

---
