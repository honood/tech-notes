# 深入解析 C++ 继承中的访问权限与特性变更

在 C++ 面向对象编程中，继承是构建类层次结构和实现代码复用的核心机制。除了继承接口和实现，C++ 还提供了一套精巧的机制，允许派生类在继承基类成员时，对其访问权限和其它特性进行调整。这种能力为类的设计者提供了更细粒度的控制，但也需要谨慎使用，以避免破坏设计原则。

本文将全面、深入地探讨派生类改变基类成员访问权限的规则、`using` 声明在继承中的多种应用，及其在软件设计中的意义。

## 核心机制：[`using` 声明](https://en.cppreference.com/w/cpp/language/using_declaration.html#In_class_definition)

在派生类中调整继承成员特性的主要工具是 [**`using` 声明**](https://en.cppreference.com/w/cpp/language/using_declaration.html#In_class_definition)。其基本语法是在派生类的定义中，将 `using Base::member;` 语句放置在目标访问权限区域 (`public`、`protected` 或 `private`) 之下。

对于成员函数和数据成员，`using` 声明的作用是将其名称从基类作用域引入到派生类作用域，并根据其在派生类中声明的位置，为其赋予新的访问级别。

```cpp
class Base {
public:
  void public_func() {}
  int public_data = 1;

protected:
  void protected_func() {}
  int protected_data = 2;

private:
  // private 成员对派生类不可见
  void private_func() {}
  int private_data = 3;
};

class Derived : public Base {
public:
  // 将 protected 成员 'protected_func' 的访问权限提升为 public
  using Base::protected_func;
  // 将 protected 数据成员 'protected_data' 的访问权限提升为 public
  using Base::protected_data;

private:
  // 将 public 成员 'public_func' 的访问权限降低为 private
  using Base::public_func;
  // 将 public 数据成员 'public_data' 的访问权限降低为 private
  using Base::public_data;

  // 编译错误：派生类无法访问基类的 private 成员，因此也无法通过 using 声明引入
  // using Base::private_func; // COMPILE ERROR!
  // using Base::private_data; // COMPILE ERROR!
};
```

需要强调的是，**派生类永远无法访问基类的 `private` 成员**。这是 C++ 封装性的基石。因此，任何尝试通过 `using` 或其它方式来改变基类 `private` 成员访问性的行为都会导致编译失败。

## 一、提升访问权限 (Increasing Visibility)

这是 `using` 声明在继承中最常见的应用场景，其目的是将基类中受保护的 (`protected`) 成员暴露给派生类的使用者，使其变为公有的 (`public`)。

### 设计动机

基类的设计者可能会提供一些辅助函数，这些函数对于派生类的实现非常有用，但它们不应成为基类通用公共接口的一部分。因此，它们被声明为 `protected`。然而，某个具体的派生类可能认为，将这些辅助功能公开给自己的客户是有价值的，此时便可以提升其访问权限。

### 代码示例: `protected` -> `public`

假设有一个通用的文本处理器基类，它提供了一个受保护的核心处理步骤。一个具体的派生类希望将这个步骤公开，以便用户可以单独调用它。

```cpp
#include <iostream>
#include <string>

class BaseTextProcessor {
protected:
  // 一个受保护的辅助函数，用于文本处理的某个核心步骤
  void core_processing_step(const std::string& text) {
    std::cout << "Core processing step: " << text << std::endl;
  }
};

class PublicProcessor : public BaseTextProcessor {
public:
  // 通过 using 声明，将 core_processing_step 的访问权限提升为 public
  using BaseTextProcessor::core_processing_step;

  void full_process(const std::string& text) {
    std::cout << "PublicProcessor starts full process..." << std::endl;
    // 在派生类内部，可以直接调用继承的 protected 成员
    core_processing_step(text);
    std::cout << "PublicProcessor finished." << std::endl;
  }
};

int main() {
  PublicProcessor proc;

  // 由于权限被提升，外部代码可以直接调用该函数
  proc.core_processing_step("Hello, World!");

  // 也可以通过派生类提供的完整流程来间接调用
  proc.full_process("Data");
}
```

## 二、降低访问权限 (Decreasing Visibility)

虽然不那么常见，但派生类也可以将基类的公有 (`public`) 成员的访问权限降低为受保护的 (`protected`) 或私有的 (`private`)。

### 设计动机

此操作通常用于在派生类中 “限制” 或 “禁用” 基类的某个功能。当基类的某个公共接口不适用于特定的派生类，或者直接调用它可能会破坏派生类的内部状态时，设计者可以选择隐藏它，并可能提供一个更适合的替代接口。

### 代码示例: `public` -> `private`

想象一个 `BaseWidget` 类，它有一个通用的 `draw()` 方法。现在有一个 `SpecialButton` 派生类，它要求绘制操作必须与点击事件绑定，不希望用户能独立调用 `draw()`。

```cpp
#include <iostream>

class BaseWidget {
public:
  // 一个通用的绘制函数
  void draw() {
    std::cout << "BaseWidget::draw()" << std::endl;
  }
};

class SpecialButton : public BaseWidget {
private:
  // 将基类的 draw() 接口在 SpecialButton 中的访问权限降级为 private
  using BaseWidget::draw;

public:
  // 提供一个新的、封装了绘制逻辑的公共接口
  void click_and_draw() {
    std::cout << "Button was clicked!" << std::endl;
    // 在内部实现中，仍然可以合法地调用这个现在是 private 的 draw() 方法
    draw();
  }
};

int main() {
  SpecialButton button;
  button.click_and_draw();

  // 编译错误！'draw' is a private member of 'SpecialButton'
  // button.draw(); // COMPILE ERROR!
}
```

### 设计警示：里氏替换原则 (Liskov Substitution Principle)

降低公共成员的访问权限通常会**违反里氏替换原则 (LSP)**。LSP 指出，程序中任何使用基类引用的地方，都应该能够无缝地替换为派生类的引用，而程序的行为不应产生错误。当降低访问权限时，可能出现设计上的不一致性，是一种需要审慎使用的技术。

## 三、特殊应用：继承构造函数

`using` 声明在应用于构造函数时，其语义与应用于普通成员函数截然不同。为了理解其特殊性，首先需要明确 C++ 的一个基本规则。

### 基础：构造函数默认不被继承

在 C++ 的核心设计中，构造函数是特殊的，它们**默认不会被派生类继承**。原因在于，构造函数的根本职责是初始化其所属的特定类的成员。基类的构造函数只了解如何初始化基类的成员，对派生类可能新增的成员一无所知，因此它无法胜任构造一个完整派生类对象的任务。

派生类对象的构造遵循一个 **“调用链”** 模型，而非继承模型。派生类的构造函数必须在其成员初始化列表中调用其直接基类的某个构造函数，来完成对基类子对象的初始化。

```cpp
class Base {
public:
  Base(int value) : _value(value) {}
private:
  int _value;
};

// C++11 之前的标准做法
class Derived : public Base {
public:
  // Derived 必须定义自己的构造函数...
  Derived(int v_base, double v_derived)
    : Base(v_base), _extra_data(v_derived) // ...并显式调用 Base 的构造函数
  {
  }
private:
  double _extra_data;
};
```

这种手动编写转发构造函数的方式，在基类拥有多个构造函数时会变得非常繁琐。

### C++11 的解决方案：`using Base::Base;`

为了解决上述样板代码问题，C++11 引入了 `using` 声明用于构造函数的特殊语法，俗称 **“继承构造函数”**。

#### 核心区别

-   **普通成员函数**：`using Base::func;` 的作用是**改变** `func` 在派生类中的**访问权限**。
-   **构造函数**：`using Base::Base;` 的作用不是改变权限，而是**指示编译器为派生类自动生成一组 “转发构造函数”**。

#### 机制详解

`using Base::Base;` 声明会为派生类 `Derived` 生成一组构造函数，这些构造函数与基类 `Base` 的所有可见（非 `private`）、非特殊的构造函数具有相同的参数列表。每个生成的构造函数体内的唯一操作就是用相同的参数调用基类对应的构造函数。

#### 关键规则

1.  **代码生成**：它是一个代码生成指令，而非简单的名称引入。编译器会查找基类 `Base` 的所有可见（非 `private`）、非特殊的构造函数。
2.  **签名匹配**：对于找到的每个基类构造函数，编译器会在派生类 `Derived` 中生成一个具有相同参数列表的构造函数。
3.  **转发调用**：每个生成的构造函数体内的唯一操作就是用相同的参数调用基类对应的构造函数。
4.  **权限保留**：生成的派生类构造函数的访问权限 (`public`, `protected`) **与基类中对应的构造函数完全相同**。`using Base::Base;` 语句本身的位置（在 `public` 或 `private` 区域）对此没有影响。
5.  **冲突解决**：如果派生类显式定义了一个与将被继承的构造函数签名相同的构造函数，则以显式定义的版本为准，同签名的基类构造函数将不会被继承。
6.  **特殊构造函数**：默认、拷贝和移动构造函数不会通过这种方式继承。编译器会根据常规规则为派生类生成它们。

### 代码示例：继承构造函数

```cpp
#include <iostream>
#include <string>

class Message {
public:
  Message(int id) : _id(id), _content("Default") {}
  Message(int id, std::string content) : _id(id), _content(content) {}

protected:
  // 受保护的构造函数，用于内部或派生类
  Message(std::string const& raw) : _id(-1), _content(raw) {}

private:
  int _id;
  std::string _content;
};

class Email : public Message {
public:
  // C++11 之前的做法：手动编写所有转发构造函数，非常繁琐
  // Email(int id) : Message(id) {}
  // Email(int id, std::string content) : Message(id, content) {}

  // C++11 及之后的做法：一条语句继承所有可见的构造函数
  //
  // 这一行语句，让 Email 类 “继承” 了 Message 的所有 public 构造函数
  // 编译器会自动生成 Email(int) 和 Email(int, std::string)
  using Message::Message;

  void setRecipient(std::string const& r) { _recipient = r; }

private:
  std::string _recipient;
};

int main() {
  // 可行：继承了 public Message(int)
  Email e1(101);

  // 可行：继承了 public Message(int, std::string)
  Email e2(102, "Hello");

  // e1 和 e2 的 _recipient 成员会被默认初始化（对于 std::string 是空字符串）

  // 编译错误：'Message::Message(const string&)' is protected
  // 构造函数被 “继承” 了，但它在 Email 中仍然是 protected 的！
  // Email e3("raw data"); // COMPILE ERROR!
}
```

这个例子清晰地表明，`using Base::Base;` 的效果是为派生类生成了具有**原始访问权限**的构造函数，这与它用于普通成员函数时可以改变访问权限的行为截然不同。

## 四、虚函数的特殊处理

对于虚函数，访问权限的变更更为灵活，它可以通过两种方式实现。

### 方式一：通过重写 (Overriding)

当派生类重写 (`override`) 一个基类虚函数时，可以为这个重写版本指定**任意的访问权限**，无论基类中该函数的原始权限是什么。

**核心机制解析：编译时访问检查 vs. 运行时动态派发**

1.  **编译时访问检查**：编译器在检查函数调用是否合法时，依据的是**静态类型 (Static Type)**。在基类成员函数内部调用其它成员函数（即使是 `private` 的），总是合法的。
2.  **运行时动态派发**：对于虚函数调用，程序在运行时会根据对象的**实际类型 (Dynamic Type)** 来决定调用哪个版本。这个过程通过虚函数表 (vtable) 实现，此时访问权限已在编译阶段检查完毕，不再是决定因素。

### 代码示例：`private virtual` -> `public override`

这个例子解释了为何 `Publisher` 的 `public` 方法 `notify()` 可以调用其 `private virtual` 的 `notifyOne()`，并最终执行到 `SettingsPublisher` 中 `public override` 的版本。这是**模板方法 (Template Method) 设计模式**的经典实现。

```cpp
#include <iostream>
#include <vector>
#include <string_view>

class Subscriber {
public:
  virtual ~Subscriber() = default;
  virtual void update(std::string_view message) = 0;
};

class Publisher {
public:
  virtual ~Publisher() = default;
  void subscribe(Subscriber* s) { _subscribers.push_back(s); }

  // 算法骨架，公开接口
  void notify() const {
    for (auto* s : _subscribers) {
      // 编译时检查: 在 Publisher 内部调用自身的 private 成员，合法。
      notifyOne(s, "update available");
    }
  }

private:
  // 可变步骤，声明为 private virtual 以强制派生类实现，同时隐藏细节
  virtual void notifyOne(Subscriber* s, std::string_view msg) const = 0;

  std::vector<Subscriber*> _subscribers;
};

class SettingsPublisher : public Publisher {
public:
  // 运行时行为: 动态派发会调用此处的重写版本
  void notifyOne(Subscriber* s, std::string_view msg) const override {
    s->update(msg);
  }
};
```

### 方式二：使用 `using` 声明（不重写）

如果派生类不想提供新的实现，仅仅是想改变基类虚函数的访问权限，同样可以使用 `using` 声明。

```cpp
#include <iostream>

class Base {
protected:
  virtual void hook() { std::cout << "Base::hook\n"; }
};

class Derived : public Base {
public:
  // 不重写 hook，只是将其访问权限从 protected 提升为 public
  using Base::hook;
};

int main() {
  Derived d;
  // 调用的是基类的实现，因为 Derived 并没有重写它
  d.hook(); // 输出 "Base::hook"
}
```

## 五、非虚函数的处理

对于非虚函数，`using` 声明改变其访问权限的规则与虚函数完全一致。`using` 声明只关心成员的名称和其所在的作用域，而不关心函数是否为虚函数。

```cpp
#include <iostream>

class Base {
protected:
  void utility_func() { std::cout << "Base::utility_func\n"; }
};

class Derived : public Base {
public:
  // 将非虚函数的权限从 protected 提升为 public
  using Base::utility_func;
};

int main() {
  Derived d;
  d.utility_func(); // 输出 "Base::utility_func"
}
```

## 总结

下表总结了 C++ 继承中**针对成员函数和数据成员**的访问权限变更规则。注意，继承构造函数 (`using Base::Base;`) 是一个特例，其行为是生成代码而非改变权限。

| 原始权限 (基类) | 目标权限 (派生类) | 主要方式 | 是否可行 | 备注与设计考量 |
| :--- | :--- | :--- | :--- | :--- |
| `public` | `protected` / `private` | `using` | 是 | 谨慎使用。可能违反里氏替换原则 (LSP)。 |
| `protected`| `public` | `using` | 是 | 非常常见的用法，用于有选择地暴露基类为派生类提供的实现细节。 |
| `private` | 任何权限 | `using` 或重写 | **否** | **绝对不可行**。派生类无法访问基类的 `private` 成员。 |
| 任何可见权限 | 任何权限 | 重写虚函数 | 是 | 派生类可以在 `override` 时自由指定全新的访问权限，常用于实现设计模式。 |

掌握这些机制，不仅能帮助理解复杂的 C++ 代码，更能作为一种强大的设计工具，用于构建灵活、健壮且封装良好的类层次结构。然而，与所有强大的工具一样，它需要被审慎地使用，时刻关注其对面向对象设计原则（如封装和里氏替换原则）的影响。

---
