# 关键词

## `constexpr`

`constexpr` 是 **C++11** 引入的核心关键词，意为 “常量表达式”（constant expression），用于**强制变量、函数或对象在编译期完成计算或初始化**，而非运行期。它是 C++ 实现 “编译期编程”（Compile-time Programming）的关键工具，核心目标是**提升性能**（将运行期计算移至编译期）和**增强类型安全**。

---

- **`const`**：表示 “只读”，值可以在**运行期**确定（如 `const int x = rand();` 合法）。
- **`constexpr`**：表示 “编译期常量”，值必须在**编译期**就能计算出来（如 `constexpr int x = rand();` 非法，因为 `rand()` 是运行期函数）。

简单来说：**`constexpr` 一定是 `const`，但 `const` 不一定是 `constexpr`**。

---

与define：核心差异体现在**类型安全、作用域、内存占用、调试性**等维度。以下是详细对比：

 1. 类型安全

- **变量**：有明确的类型（如 `int`、`double`），编译器会进行**类型检查**，避免类型不匹配错误。
- **`#define`**：是**预处理器的文本替换**，无类型概念，仅做字符串替换，容易引发隐式类型转换或逻辑错误。

 2. 作用域

- **变量**：有严格的作用域（块级作用域、函数作用域、文件作用域等），可通过 `static`/ 命名空间限制访问范围。
- **`#define`**：作用域是**从定义点到文件结束**，或通过 `#undef` 提前取消，无块级作用域，容易污染命名空间。

 3. 内存占用与符号

- **变量**：（非优化情况下）会占用内存，有**内存地址**，可通过 `&` 取地址；`const`/`constexpr` 变量可能被优化为 “编译期常量”，但本质仍有类型和符号。
- **`#define`**：仅做文本替换，**不占用内存**，无内存地址，也不会出现在符号表中。

---
`constexpr` 变量：必须用**常量表达式**初始化（即编译期能确定值的表达式，如字面量、其他 `constexpr` 变量 / 函数的返回值）。

**`constexpr` 函数**：编译期可计算的函数：`constexpr` 函数的特点是：若传入的参数是常量表达式，则函数在编译期执行；若参数是运行期值，则退化为普通函数（C++14 及以后支持）。

## `explicit`

`explicit` 是 C++ 中专门用来**修饰类的构造函数**的关键字，核心作用只有一个：

**禁止单参数构造函数的「隐式类型转换」，强制构造函数只能「显式调用」**，从根源避免代码因编译器自动转换产生意外 bug。

如果一个类的构造函数**只有一个参数**（或者有多个参数，但除了第一个都有默认值），编译器会默认允许：把「构造函数的参数类型」**自动隐式转换成当前类的对象**，不需要手动写构造语法；加了`explicit`关键词之后**仍然可以进行显式转换**。

无 explicit 的危险示例（隐式转换）:

``` c++
#include <iostream>
class Number {
public:
    // 单参数构造函数（无 explicit）
    Number(int value) : m_val(value) {
        std::cout << "构造函数调用: " << value << "\n";
    }

private:
    int m_val;
};

// 接收 Number 对象的函数
void printNum(Number num) {}

int main() {
    // ✖️ 危险：编译器偷偷做了隐式转换！
    // 本意是赋值 int，实际自动调用 Number(10) 构造了对象
    Number n1 = 10; 

    // ✖️ 更危险：直接传 int 给需要 Number 的函数
    printNum(20); 

    return 0;
}
```

几乎所有企业 C++ 编码规范：**所有单参数构造函数，默认必须加 explicit**。

## `const`与`mutable`

关键词`const`多才多艺：

- 面对**变量**：

  - 既可以修饰**`namespace`**或类外部的**全局常量**；

  - 也可以修饰在**文件、函数、区块作用域**中被声明为`static`的对象；

  - 同样也可也修饰**`class`内部**的`static`和`non-static`**成员变量**；

---

- 面对**函数**（最具有威力）在一个函数声明式内`const`可以和函数返回值、参数、函数自身（如果是成员函数）产生关联：

  - 修饰**返回值**：

    - 场景一：按 **return by reference(pointer)** 时，保护类的内部状态，打破封装

      这是现代 C++ 中最常见、也最重要的 `const` 返回值用法。当你想让外部读取类内部的一个大型对象（如 `std::string` 或 `std::vector`），为了提高性能你会选择**按引用返回**（避免拷贝）。但如果不加 `const`，外部就可以轻易修改类的内部数据，破坏了面向对象的封装性。

      ``` c++
      class Student {
      private:
          std::string name;
      public:
          Student(const std::string& n) : name(n) {}
          
          // 危险：返回了内部成员的普通引用
          std::string& getName() { 
              return name; 
          }
      };
      
      int main() {
          Student student("Alice");
          
          // 灾难发生：外部不小心修改了 student 内部的名字！
          student.getName() = "Bob"; 
          
          return 0;
      }
      ```

      ---

    - 场景二：按值返回自定义对象时，防止“无意义的赋值”导致的隐藏 bug.

      这种用法在**自定义对象的操作符重载**中特别常见。假设我们自己实现了一个有理数类（`Rational`），并且重载了乘法操作符 `*`。如果程序员在写条件判断时，不小心把等于运算符 `==` 错写成了赋值运算符 `=`，会发生什么？

      ``` c++
      class Rational {
          // ... 省略具体实现 ...
      };
      
      // 返回一个普通的 Rational 对象
      Rational operator*(const Rational& lhs, const Rational& rhs);
      
      int main() {
          Rational a, b, c;
          
          // 程序员的本意是判断 a*b 是否等于 c： if (a * b == c)
          // 但不小心漏打了一个 '='：
          if (a * b = c) {  
              // ...
          }
          return 0;
      }
      ```

      在这个例子中，`a * b` 会产生一个临时的右值对象。如果不加 `const`，C++ 允许你对这个临时对象进行赋值操作（`a * b = c`）。这行代码**可以正常编译运行**，但它只是修改了一个马上就会被销毁的临时变量，不仅毫无意义，而且会导致 `if` 语句永远计算为 `true`（或者根据重载情况引发其他怪异行为），这种 bug 在庞大的代码库中极难排查。

      虽然上面提到的按值返回 `const` 对象在 C++98 时代被奉为经典准则，但在 C++11 引入了**移动语义（Move Semantics）**之后，情况发生了变化：在现代 C++ 中，**极度不推荐按值返回 `const` 对象**。因为返回 `const` 对象会**阻止编译器使用移动构造函数或移动赋值运算符**。`const` 意味着不可修改，而“移动”本质上需要“窃取”（修改）右值内部的资源。如果返回值是 `const`，编译器就只能退化去调用昂贵的拷贝构造函数，从而拖慢程序性能，现在很多编译器已经可以进行警告⚠。

      ---

  - 修饰**参数**：

    修饰参数没什么好说的，应该在必要使用它们的时候进行使用。

    ---

  - 修饰**函数自身（如果是成员函数）**：

    在 C++ 中，在类的成员函数声明末尾加上 `const`（例如 `void print() const;`），被称为**常量成员函数**。它向编译器和代码的阅读者做出了一个极其重要的承诺：“这个函数绝对不会修改该对象（`this`指针的指向）内部的**任何成员变量**（非 `static` 变量，因为 `static` 修饰的不属于该对象）”。这在 C++ 的设计哲学中被称为 **“常量正确性（Const Correctness）”**。

    ---

    - 场景一：配合`pass by reference to const`，解决“只能看不能摸”的编译报错

      这是 `const` 成员函数存在的最核心原因。在 C++ 中，为了避免对象拷贝带来性能损耗，我们通常会将对象通过**常量引用（`const ClassName&`）**传递给函数。一旦成员变量被 `const`修饰就变为常量对象，**编译器就规定：这个常量对象，只能调用被 `const` 修饰的成员函数**。如果调用了普通的非 `const` 函数，编译器会害怕那个函数悄悄修改了对象，从而破坏了 `const` 的承诺，发生编译器报错。

      ``` c++
      #include <iostream>
      #include <string>
      
      class BankAccount {
      private:
          double balance;
      public:
          BankAccount(double b) : balance(b) {}
      
          // 逻辑上只是读取余额，但忘记加 const 修饰
          double getBalance() { 
              return balance; 
          }
      };
      
      // 审计函数：为了不拷贝账户对象，且保证不篡改数据，使用 const 引用传参
      void auditAccount(const BankAccount& account) {
          // 💥 编译报错！
          // 错误信息类似：不能将 "this" 指针从 "const BankAccount" 转换为 "BankAccount &"
          std::cout << "当前余额: " << account.getBalance() << std::endl; 
      }
      
      int main() {
          BankAccount myAccount(1000.0);
          auditAccount(myAccount);
          return 0;
      }
      ```

      ---

    - 场景二：逻辑常量 vs 物理常量（`mutable` 关键字的用武之地）

      有时候我们会遇到一种很纠结的场景：从外部使用者的“逻辑”上看，调用这个函数并没有修改对象的状态，应该加上`const`；但在类内部的“物理”实现上，为了性能或记录，我们又确实需要修改某个隐藏变量。

      举个例子：我们有一个计算特别耗时的数学对象。为了优化，我们想加一个“缓存（Cache）”。当第一次请求数据时，计算并存入缓存；后续请求直接返回缓存。

      **矛盾点：** 获取数据的方法 `getData()` 在逻辑上绝对应该是个 `const` 函数（因为它不改变数学对象的本质）。但如果把它标记为 `const`，它内部就无法给缓存变量赋值了！解决方案：`const` 成员函数配合 `mutable` 关键字。

      ``` c++
      #include <iostream>
      
      class HeavyMathObject {
      private:
          int baseValue;
          
          // mutable 关键字：允许该变量即使在 const 成员函数中也能被修改！
          mutable bool isCached;
          mutable int cachedResult;
      
      public:
          HeavyMathObject(int val) : baseValue(val), isCached(false), cachedResult(0) {}
      
          // 逻辑上，获取结果不会改变对象的外部状态，所以必须是 const
          int getResult() const {
              if (!isCached) {
                  std::cout << "正在进行极其耗时的计算..." << std::endl;
                  // 正常情况下 const 函数里不能修改成员变量
                  // 但因为 isCached 和 cachedResult 被 mutable 修饰，这里被允许了！
                  cachedResult = baseValue * baseValue * baseValue; // 假装很耗时
                  isCached = true;
              } else {
                  std::cout << "直接返回缓存..." << std::endl;
              }
              return cachedResult;
          }
      };
      
      int main() {
          // 即使对象是 const 的，也能完美运作
          const HeavyMathObject obj(10);
          
          std::cout << obj.getResult() << std::endl; // 触发计算
          std::cout << obj.getResult() << std::endl; // 直接返回缓存
          
          return 0;
      }
      ```

      **解析：** 这个场景展示了 C++ 设计的精妙之处。`const` 成员函数不仅是对外的一份“契约”（我不改变状态），在内部实现遇到特殊情况时，C++ 也给了你 `mutable` 这样一个小后门，让你可以完美兼顾**“外部接口的常量安全性”**和**“内部实现的灵活性”**。

      ---

      物理常量性 vs 逻辑常量性

      编译器是一个极其死板的机器，它只懂得**“物理常量性”**：

      - **编译器的视角**：只要你在这个函数里修改了对象所在内存的哪怕一个比特（Bit）的数据，我就认为你改变了对象，我就不准你加 `const`。

      但是，作为写代码的人类，我们关注的是**“逻辑常量性”**：

      - **程序员的视角**：从外部调用者的角度看，调用这个函数之后，对象的**业务逻辑状态和外部表现**有没有发生变化？如果没有，那它在逻辑上就是 `const` 的。
      
      `mutable` 的出现，**绝不仅仅是为了自洽或打补丁，而是为了在“死板的编译器”和“灵活的业务逻辑”之间搭建一座桥梁。** 它的真正含义是告诉编译器：“这个变量不属于对象的逻辑状态（它是底层的脏活累活/基础设施），请你对它网开一面。”
      
      ---
      
      既然改了值，为什么还要硬声明为 `const`？
      
      因为 `const` 是一份**对外的契约**，而不是**对内的枷锁**。
      
      假设你写了一个类库给别人用，如果你的获取数据函数 `getData()` 没有加 `const`，会导致一个致命的连锁反应：
      
      1. 别人无法将你的对象作为 `const Type&` 传参。
      2. 别人无法把你的对象放进某些要求 `const` 的标准库容器中。
      3. 别人在看代码时，不敢确定调用 `getData()` 会不会产生破坏性的副作用（比如把数据清空了）。
      
      声明为 `const`，是向所有的使用者保证：**“放心调用吧，对于你关心的核心数据，我绝对原封不动。”** 至于内部偷偷摸摸做了什么（比如写了条日志、更新了下缓存），调用者根本不需要，也不应该关心。这就是面向对象中“封装”的精髓。
      
      ---
      
      mutable 是不可或缺的：一个最硬核的实战证明
      
      **如果没有 `mutable`，整个面向对象的多线程编程就彻底崩溃了。**那就是**互斥锁（Mutex）**。
      
      假设我们有一个账户类，在多线程环境下运行。我们要提供一个查询余额的接口：
      
      ``` c++
      #include <mutex>
      
      class ThreadSafeAccount {
      private:
          double balance;
          // 注意这里的 mutable
          mutable std::mutex mtx; 
      
      public:
          ThreadSafeAccount(double b) : balance(b) {}
      
          // 查询余额，逻辑上绝对应该是一个 const 操作！
          double getBalance() const {
              // 锁定互斥锁（注意：lock() 操作会修改 mtx 的内部状态！）
              std::lock_guard<std::mutex> lock(mtx); 
              
              return balance; // 安全地读取
          }
      };
      ```
      
      1. 外部调用 `getBalance()`，纯粹是为了“看一眼”余额，这个函数在业务逻辑上 **必须是 `const`**。
      2. 为了保证多线程安全，读取前必须对 `std::mutex` 加锁（`lock()`）。
      3. **加锁这个动作，本质上是在修改 `mtx` 对象的内部状态！**
      
      如果 C++ 没有 `mutable` 关键字，将陷入死局：
      
      - 你要么把 `getBalance()` 的 `const` 去掉。但这样一来，所有传进来的 `const ThreadSafeAccount&` 都无法查询余额了，这极其荒谬。
      - 你要么强行用指针强转等黑魔法绕过编译器的检查，这会导致未定义行为。
      
      正因为有了 `mutable`，你可以把 `mtx` 标记为“可变的”。这完美地表达了语义：**`mtx` 只是为了保证线程安全的底层管道设施，它不属于账户的“业务数据（balance）”。因此，修改管道设施，并不违背账户对象作为 `const` 的承诺。**
    
    ---
    
    在 C++ 开发中，有一个几乎所有资深程序员都会遵守的铁律： **只要一个成员函数没有修改对象状态的意图，就请毫不犹豫地在它末尾加上 `const`。**
    
    这不仅仅是为了避免上述的编译错误，更是为了让代码自我记录（Self-documenting）。当你看到一个 `const` 函数时，你不用去阅读它的源码，就能放心地在任何安全级别要求高的地方（如并发读取）调用它。

---

- 面对**指针**：也可以指出指针自身（指针常量`char* const p;`）、指针所指之物（常量指针`const char *p`）或者二者都是`const char* const p;`（都不是）常量。

---

- 面对**迭代器**：声明一个迭代器为`const` (例如`const std::vector<int>::iterator iter = vec.begin())`则和声明一个**指针常量**一样，`iter`本身不可改变：`*iter = 10;(合法)  iter++;(非法)`；如果是希望**常量指针的效果**（迭代器指向的值不可以改变）则应该使用`std::vector<int>::const_iterator`类型。

# 常用库

## `<chrono>`

`<chrono>` 是 C++11 引入的标准库头文件，提供了**类型安全、高精度**的时间处理功能，核心围绕 “时长”“时间点”“时钟” 三个概念设计，解决了传统 C 风格时间函数（如 `time()`、`clock()`）精度低、类型不安全的问题。

### `duration`

表示 “一段时间间隔”，如 3 秒、50 毫秒。定义为：

``` c++
template <class Rep, class Period = ratio<1>>
class duration;

// Rep：数值类型（如 int、double），存储时长的 “数量”。
// Period：时间单位，用 std::ratio 表示（如 ratio<1> 是秒，ratio<1, 1000> 是毫秒）。
```

**预定义时长**（为方便使用，标准库定义了常用类型）：

``` c++
using nanoseconds  = duration<long long, nano>;   // 纳秒
using microseconds = duration<long long, micro>;  // 微秒
using milliseconds = duration<long long, milli>;  // 毫秒
using seconds      = duration<long long>;          // 秒
using minutes      = duration<int, ratio<60>>;     // 分钟
using hours        = duration<int, ratio<3600>>;   // 小时
```

### `time_point`

表示 “某个时钟下的具体时刻”，定义为：

``` c++
template <class Clock, class Duration = typename Clock::duration>
class time_point;

// Clock：关联的时钟类型（见下文）。
// Duration：时间精度，默认为时钟的 native 精度。
```

### 时钟

提供 “获取当前时间点” 的接口，标准库定义了三种时钟：

| 时钟类型                | 特性                                                         | 适用场景                       |
| ----------------------- | ------------------------------------------------------------ | ------------------------------ |
| `system_clock`          | 系统范围的实时时钟（可手动调整，如 NTP 同步），关联 Unix 时间戳。 | 获取日历时间、跨进程同步。     |
| `steady_clock`          | 单调时钟（不可调整，时间点严格递增），精度通常高于 `system_clock`。 | 性能计时、测量时间间隔。       |
| `high_resolution_clock` | 实现定义的 “最高精度时钟”（通常是 `system_clock` 或 `steady_clock` 的别名）。 | 高精度计时（需结合平台验证）。 |

**典型用法：**

``` c++
#include <chrono>
#include <iostream>

void expensive_function() {
    // 模拟耗时操作
    for (volatile int i = 0; i < 100000000; ++i);
}

int main() {
    // 1. 获取开始时间点
    auto start = std::chrono::steady_clock::now();
    
    // 2. 执行待测量代码
    expensive_function();
    
    // 3. 获取结束时间点，计算时长
    auto end = std::chrono::steady_clock::now();
    auto duration = end - start; // 类型为 steady_clock::duration（通常是纳秒级）
    
    // 4. 时长转换为毫秒输出
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(duration);
    std::cout << "耗时: " << ms.count() << " 毫秒\n";
    
    return 0;
}
```



# 命名空间

`namespace`（命名空间作用域）是 C++ 为解决命名冲突、规范化代码作用域设计的核心语法，它是**纯编译期特性**（运行时完全不存在），底层依靠编译器的**名字修饰（Name Mangling）** 实现唯一标识。

---

## `using namespace`

`using namespace` 是**调用函数 / 类 **时使用的：把**命名空间**里的所有名字**导入**到当前作用域**（不改变作用域）**，编译器自动匹配命名空间。

**定义函数 / 类**时，必须手动写在命名空间`namespace my_string{}`内，或者加空间名`my_string::strlen`；定义类的成员函数需要`::`声明成员函数属于哪个类。

在命名空间`namespace my_string{}`内调用函数时首先查找**当前命名空间作用域**（有`{}`就代表定义了作用域）里的函数，然后查找上一层作用域，最后查找全局作用域。

---

``` c++
/* 自定义函数 */
my_string::my_strlen
/* 标准库函数 */ 
std::strlen
/* 全局函数(::指定调用全局函数) */
::my_strlen
```

这三个是完全独立、互不干扰的函数，编译器靠**命名空间前缀**区分它们。

**全局函数** = 不属于任何命名空间、也不属于任何类的函数，它直接暴露在 全局作用域（global scope） 里，整个程序都能访问到。

## 匿名命名空间

可以在**嵌套命名空间内部**再写**匿名命名空间**，用于封装**仅内部使用的工具函数**，外部完全无法访问：

``` c++
namespace myactua{

#define NSEC_PER_SEC (1000000000L)
#define CLOCK_TO_USE CLOCK_MONOTONIC

/* namespace后无空间名，为匿名命名空间 */
namespace {
constexpr double kPi = 3.14159265358979323846;
constexpr double kPosPlusPerRev = 131072.0;
constexpr double kRawPosToRad = (2.0 * kPi) / kPosPlusPerRev;
constexpr double kRawVelToRpm = 60.0 / kPosPlusPerRev;
constexpr double kRpmToRadPerSec = (2.0 * kPi) / 60.0;
constexpr double kRadToDeg = 180.0 / kPi;


constexpr uint64_t kDiscreteRetryCycles = 10;       // 重试间隔：10 ms @ 1kHz
constexpr int kDiscreteSuccessStableCycles = 3;     // 连续 3 个周期满足判据视为成功
constexpr int kDiscreteMaxRetries = 100;            // 最多重试 100 次
constexpr uint64_t kDiscreteTimeoutCycles = 5000;   // 命令超时：5 s @ 1kHz

enum DiscreteFailReason {
    kDiscreteFailNone = 0,
    kDiscreteFailFault = 1,
    kDiscreteFailTimeout = 2,
    kDiscreteFailMaxRetry = 3,
};
}

 ...
     
}
```



## std库的函数/类

**Linux 的 `open` 不是 `std` 函数，** 它是**操作系统提供的系统调用**，和 C++ 标准库无关，`std` 是 C++ **跨平台标准库**，里面全是语言自带的、所有系统（Windows/Linux/Mac）都能用的函数 / 类。

`std` 是 C++ 标准库的命名空间，**所有函数 / 类都跨平台**，包含这几大类：

1. 输入/输出类：

   ``` c++
   std::cout    // 打印输出
   std::cin     // 键盘输入
   std::endl    // 换行
   ```

2. 字符串相关（**注意：在`cstring`中也定义了全局函数**）：

   ``` c++
   std::string()   // 字符串类（你自己写MyString就是模仿它）
   std::strlen()   // 字符串长度（C标准库移植到std里）
   std::strcmp()   // 字符串比较
   ```

3. **容器**：

   ``` c++
   std::vector    // 动态数组
   std::map       // 键值对
   std::list      // 链表
   ```

4. **算法工具**：

   ``` c++
   std::sort      // 排序
   std::find      // 查找
   std::swap      // 交换
   ```

5. **文件操作（跨平台替代 open）**：

   ``` c++
   std::fstream   // 文件读写
   std::ifstream  // 读文件
   std::ofstream  // 写文件
   ```

6. **其他工具函数**：

   ``` c++
   std::move      // 移动语义
   std::shared_ptr// 智能指针
   std::thread    // 线程
   ```

# 设计类

## 基本准则

<img src="pic/面向对象.png" alt="面向对象" style="zoom: 25%;" />

- 数据一定放在`private`里面。

- 构造函数的` :`一定要记得用。

---

- **成员函数**不修改类的非`mutable`的成员变量**立刻**在成员函数名后面加上`const`。

``` cpp
Complex operator+(const Complex &c) const{
    return Complex(this->real + c.real, this->imag + c.imag);
}
```

- 参数传递与返回尽量可能用引用`(reference)`来传递，不希望对方改加`const`。

- 构造函数只有一个参数，加上`explicit`。

---

- `return by reference` 时`return`的不能是函数内部的局部变量（`local`），必须返回全局变量或者不会死亡的。

- 临时对象：`typename()`是创建临时对象，到下一行就死亡，没有变量名都无所谓。

## 成员函数

成员函数推荐设计为在类中进行声明，在类外进行定义，类的成员函数**可以在声明时指定默认参数**，在调用这个函数的时候就可以不传入有默认值的函数；**默认参数都在最右侧**。

``` c++
void set_print_info(bool enable, int slave_index = -1);
```



## 操作符重载

### 重载规则

在 C++ 中，**操作符重载**的本质是**函数重载**，通过定义名为`operator@`（`@`表示要重载的运算符）的函数，让用户自定义类型（如类、结构体）支持内置运算符的语法。以下是操作符重载的详细规则、实现方式和示例。

1. **只能重载已有运算符**：不能创造新运算符（如`operator**`表示幂运算是不允许的）。

2. 不改变运算符的基本特性：

   - 优先级和结合性不能改（比如`+`始终比`*`优先级低）；

   - 操作数个数不能改（比如`+`是二元运算符，不能重载成一元）。`+=`是二元操作符：

     ``` c++
     MyInt& operator+=(const MyInt& other) {
         this->val += other.val;
         return *this;
     }
     // 单独使用+=，返回值被忽略（最常见）
     MyInt a(10), b(20);
     a += b; // 执行a.operator+=(b)，返回a的引用，但我们没有接收这个返回值
     
     // 链式调用，返回值被下一个+=接收
     int x = 1, y = 2, z = 3;
     x += y += z; // 先执行y += z（y变成5），再执行x += 5（x变成6）
     ```
    一元操作符（比如`++a、!a`）：成员函数版没有显式参数（只有隐式`this`）,二元操作符（比如`+`、`-`、`+=`）：成员函数版只有一个显式参数（右操作数）;如果是类外定义则第一个传入的参数作为左操作符。
3. **必须包含用户自定义类型**：重载的运算符至少有一个操作数是**用户定义的类型**（防止重载`int + int`这类内置类型的运算）。

4. 在C++中，**仅仅重载`+`操作符并不能直接使用`+=`语法**，因为`+`和`+=`是两个完全独立的操作符，它们对应不同的重载函数，编译器不会自动从`operator+`推导出`operator+=`的实现。
---
### 定义方式
- 运算符重载函数为成员函数实现时（**类内定义**）：隐含的`this`**指针**指向当前对象，作为运算符的**第一个操作数**。
- **类中声明类外定义**：如果运算符重载是类的成员函数，类内仅声明、类外定义时，**必须**用`类名::`限定作用域，类的本质是一个**独立的作用域**—— 类内的所有成员（数据、函数、重载的操作符）都被 “包裹” 在这个作用域内，类内的声明（比如操作符重载的声明、成员函数的声明）只是告诉编译器：“这个类有一个叫 X 的成员”，但并没有给 X 分配具体的实现 / 内存（函数体、变量初始化）。而类外定义时，编译器需要知道：“这个 X 属于哪个类的作用域？”——`类名::` 就是用来完成这个 “绑定” 的，否则编译器会把 X 当成全局作用域的标识符，导致编译错误。
- **全局函数使用友元**：没有`this`指针，所有操作数都显式作为参数；**若需要访问类的私有成员**，需在类中将其声明为类的**友元函数**，需要注意的是使用**友元**`operator-`**不是`Complex`类的成员函数**（没有`this`指针），它是全局函数，但因为类内的友元声明，它与`Complex`类建立了关联，并且获得了私有成员的访问权限。这也是为什么它能作为`Complex`类的运算符重载的原因。

``` c++
#ifndef COMPLEX_H
#define COMPLEX_H
#include <iostream>
using namespace std;
class Complex
{
private:
    double real;
    double imag;
public:
    Complex(double r = 0, double i = 0) : real(r), imag(i) {}

    double get_real() const { return real; }
    double get_imag() const { return imag; }

    // 1.友元操作符重载
    friend Complex operator*(const Complex &c1, const Complex &c2);


    // 2.类中定义操作符重载
    Complex operator+(const Complex &c) const{
        return Complex(this->real + c.real, this->imag + c.imag);
    }

    // 3.类中声明类外定义
    Complex operator-(const Complex &c) const;

    
    void print() const{
        cout << "(" << get_real() << ", " << get_imag() << ")" << endl;
    }
};
#endif
/----------------------------------------------------------------------------------------------------------/
#include "complex.h"

Complex operator*(const Complex &c1, const Complex &c2)
{
    return Complex(c1.real * c2.real - c1.imag * c2.imag, c1.real * c2.imag + c1.imag * c2.real);
}


Complex Complex::operator-(const Complex &c) const{
    return Complex(this->real - c.real, this->imag - c.imag);
}


int main()
{
    Complex c1(3, 8);
    Complex c2(3, 4);

    Complex c3 = c1 + c2;
    Complex c4 = c1 - c2;
    Complex c5 = c1 * c2;
    c3.print();
    c4.print();
    c5.print();

    return 0;
}
```

---

## 类带指针

### 三大件

类带指针一定要写出**三大件**：**拷贝构造函数（定义一个对象如何`pass by value`）**，**拷贝赋值函数**（`=`的操作符重载）和**析构函数**；如果定义了拷贝赋值函数，使用`=`也可能调用拷贝构造函数：

``` c++
Weight w1;
/* 调用拷贝构造函数 */
Weight w2(w1);

/* 调用拷贝赋值函数（w1已经被定义，不会调用构造函数） */
w1 = w2;
/* w3未被定义，调用拷贝构造函数 */
Weight w3 = w1;
```

---

编译器默认生成的版本是字符串类**浅拷贝**：只复制指针的值（地址），而不复制指针指向的内存内容，即多个对象共享同一块内存。

会导致：

- **重复释放（Double Free）**：多个对象指向同一内存，析构时每个对象都会尝试释放同一块内存。
- **悬垂指针（Dangling Pointer）**：一个对象删除内存后，其他对象的指针变成野指针。
- **内存泄漏**：对象间赋值时，旧的内存可能丢失，无法被释放。

**深拷贝**：不仅复制指针，还复制指针指向的内存内容。每个对象拥有独立的内存副本，因此需要实现拷贝构造。

---

拷贝构造函数定义一个对象如何`pass by value`：把一个类通过`pass by value`的方式传递给一个函数，这个类对象会在函数的栈上进行复制，而这个复制操作就是通过调用这个类对象的拷贝构造函数实现，而`pass by reference to const`往往是比较好的选择。

### 实现string类

`.hpp`文件：

``` c++
#ifndef MY_STRING
#define MY_STRING
#include <iostream>
#include <cstring>
#include <cassert>
namespace my_string{


size_t strlen(const char *str);


class MyString{

public:
    MyString(const char *str = nullptr);
    MyString(const MyString& other);
    MyString& operator=(const MyString &other);

    ~MyString();


private:
    char *ch = nullptr;
};


}



#endif
```

`.cpp`文件：

``` c++
#include "my_string.hpp"

using namespace my_string;
size_t my_string::strlen(const char *str){
    if(str == nullptr)
        return 0;
    
    size_t len = 0;
    while(str[len] != '\0'){len++;}

    return len;

}


MyString::MyString(const char *str){
    size_t len = strlen(str);
    char *arr = new char[len + 1];
    
    if(str){
        strcpy(arr, str);
    }

    arr[len] = '\0';
    ch = arr;
}


MyString::MyString(const MyString &other){
    size_t len = strlen(other.ch);
    ch = new char[len + 1];
    strcpy(ch, other.ch);
}


MyString& MyString::operator=(const MyString &other){
    if(this == &other)
        return *this;

    delete[] ch;

    ch = new char[strlen(other.ch) + 1];
    strcpy(ch, other.ch);

    return *this;
}


MyString::~MyString(){
    delete[] ch;
}


void test_strlen() {
    assert(my_string::strlen(nullptr) == 0);
    assert(my_string::strlen("") == 0);
    assert(my_string::strlen("hello") == 5);
    assert(my_string::strlen("hello world") == 11);
    std::cout << "test_strlen passed!" << std::endl;
}


void test_default_constructor() {
    MyString s1;
    MyString s2(nullptr);
    std::cout << "test_default_constructor passed!" << std::endl;
}


void test_constructor_with_string() {
    MyString s1("hello");
    MyString s2("hello world");
    MyString s3("");
    std::cout << "test_constructor_with_string passed!" << std::endl;
}


void test_copy_constructor() {
    MyString s1("hello");
    MyString s2(s1);
    std::cout << "test_copy_constructor passed!" << std::endl;
}


void test_assignment_operator() {
    MyString s1("hello");
    MyString s2("world");
    s2 = s1;
    
    MyString s3("test");
    s3 = s3;
    std::cout << "test_assignment_operator passed!" << std::endl;
}


void test_self_assignment() {
    MyString s1("hello");
    s1 = s1;
    std::cout << "test_self_assignment passed!" << std::endl;
}


void test_chained_assignment() {
    MyString s1("hello");
    MyString s2("world");
    MyString s3("test");
    
    s3 = s2 = s1;
    std::cout << "test_chained_assignment passed!" << std::endl;
}


void run_all_tests() {
    std::cout << "=== Running MyString Tests ===" << std::endl;
    
    test_strlen();
    test_default_constructor();
    test_constructor_with_string();
    test_copy_constructor();
    test_assignment_operator();
    test_self_assignment();
    test_chained_assignment();
    
    std::cout << "=== All Tests Passed! ===" << std::endl;
}


int main(void){
    run_all_tests();
    return 0;
}

```

## 静态函数与静态变量

在 C++ 中，类的**静态成员**（包括静态数据成员和静态成员函数）是属于**整个类**的成员，而非类的某个具体`object` —— 所有该类的`object`共享同一份静态成员，这是核心特征。

子类也会**共享**父类的静态成员函数和静态成员变量。

---

`static`函数没有`this`指针，访问数据只能访问`static`数据（非静态成员函数本质是通过`this`指针来访问具体`object`的数据）。

调用`static`函数的方式有两种：（1）通过`class name`调用：`类名::静态函数名()`。（2）通过`object`调用：`对象名.静态函数名()`；

---

`static`数据对所有`object`都是一样的，`static`数据在类中只进行声明，在类外进行定义。

``` c++
double Account::m_rate = 8.0;
```

访问`static`成员变量也有两种方式：（1）`类名::静态变量`、（2）`子类对象.静态变量` 访问。

## 模板

### 函数模板

可以指定类型，也可以让编译器进行参数推导：

``` c++
template <typename T>
T add(T a, T b) {
    return a + b;
}

int main() {
    cout << add<int>(5, 3) << endl;      // 显式指定int
    cout << add<double>(5.5, 3.2) << endl; // 显式指定double
}
```
使用多参数模板并且采用编译器自动推导：

默认情况下，**不可以** 只使用一个参数 —— 多参数模板的所有参数在实例化时都必须显式指定；但你可以通过给模板参数设置 **默认值**，实现 “只传一个参数” 的效果（未传的参数会使用默认值）。

``` c++
template <typename T1, typename T2>
void printPair(T1 first, T2 second) {
    cout << first << ", " << second << endl;
}

// 使用
printPair(10, "Hello");  // T1=int, T2=const char*
printPair(3.14, true);   // T1=double, T2=bool
```

### 类模板

类模板是 C++ 中实现**泛型编程**的核心机制。它允许定义一个 “通用的类”，这个类不绑定具体的数据类型（比如 int、float、string 等），而是用一个**类型参数**（比如 T）来占位；当实际使用这个类时再指定具体的类型，编译器会自动根据指定的类型，生成对应版本的类。

如果要指定类型的话在使用模板类创建`object`的时候就要指定类型。

`template <typename T>` **只针对紧接着的类、函数或成员有效**，**作用一次后就失效**。

``` c++
// 定义一个简单的栈类模板
template <typename T>
class Stack {
private:
    vector<T> elements;
    
public:
    void push(const T& element) {
        elements.push_back(element);
    }
    
    T pop() {
        if (empty()) throw runtime_error("Stack is empty");
        T element = elements.back();
        elements.pop_back();
        return element;
    }
    
    bool empty() const {
        return elements.empty();
    }
};

// 使用
int main() {
    Stack<int> intStack;      // 创建int类型的栈
    intStack.push(1);
    intStack.push(2);
    
    Stack<string> strStack;   // 创建string类型的栈
    strStack.push("Hello");
    strStack.push("World");
}
```

### 成员模板

在类内部定义的**模板成员**（函数或嵌套类），类本身**不一定**是模板。

如果要指定类型的话在调用这个模板函数的时候指定类型。

``` c++
class SmartPtr {
    void* ptr;  // 类本身不是模板
    
public:
    // 成员函数模板 - 在普通类中的模板函数
    template <typename T>
    T* getAs() {
        return static_cast<T*>(ptr);
    }
    
    // 嵌套类模板
    template <typename U>
    class Helper {  // 在类内部的模板类
        U value;
    };
};
```

### 模板特化



## 转换函数

转换函数没有返回类型，通常会加上`const`。如果写了转换函数之后就可以让这个类的`object`参与转换后的类型运算。（`double`进行`+ - * /`运算）

`non-explicit ctor`编译器会尝试将非此类的`object`转换为当前类再执行重载的操作符，如果在构造函数前加上`explicit`就不会让编译器来尝试将其他量构造为此类的`object`来执行操作符重载。

## 智能指针

也就是把一个类设计成`pointer-like classses`，这个类里面一定有一个真正的指针，希望比指针多一些功能。

肯定要写出`* ->`的操作符重载。

``` c++
T* operator->() const{
    return px;
}
//sp->method() == px->method()  ->会继续作用下去。

T* operator*() const{
    return *px;
}
//*sp 会用掉*，来触发调用 * 的操作符重载，因此需要返回 *px。
```

迭代器写法不一样。

## function-like classes

相当于能够接受对`()`进行操作符重载。

``` c++
const T& operator() (const T &x) const{
    return x;
}
```

# 堆、栈与内存管理

<img src="pic/堆栈.png" alt="堆栈" style="zoom:50%;" />

栈是存在于某一个作用域的一块内存空间，例如函数本身会形成一个栈来放置它接受的参数、返回地址、`local object`，只要离开作用域就会消失。

## new/delete

<img src="./assets/new一个类的过程.png" alt="new一个类的过程" style="zoom: 33%;" />



`new` 是专门用来**在堆上创建对象、并返回对象指针**的关键字，如果不用`new`则是编译器**自动分配内存**（全局区 / 栈，由变量位置决定）。

`new`一个类是**先调用`malloc()`**分配内存再调用构造函数；`delete`一个类是**先调用析构函数**再调用`operator delete(ps)`（本质是调用`free()`）释放内存。

---

分配出来的内存大小一定是 16 的倍数字节；

记录分配的内存大小的`cookie`（红色部分）的字节大小取决于机器的位数：如果为 32 位则是 4 字节，如果为 64 位机器则是 8 字节大小（取决于CPU 一次性能读取多少个字节）；

灰色部分为`debug`模式添加的。

<img src="pic/分配的内存.png" alt="分配的内存" style="zoom:33%;" />

`delete[]`的工作流程正是依赖这个信息：

1. **读取数组元素个数**：知道需要处理多少个元素；
2. **调用析构函数**（仅对类类型有效）：从最后一个元素到第一个元素，依次调用每个对象的析构函数；

而`delete`（无`[]`）的工作流程是：

- 直接将指针视为**单个对象的地址**，只会调用**一次**析构函数（类类型），并释放**单个对象的内存**；
- 它**不知道**数组的元素个数，也不会处理数组的额外信息，导致内存释放不完整。

### array delete

<img src="pic/delete[].png" alt="delete[]" style="zoom:50%;" />

当用`new[]`分配数组时，编译器会在实际数组内存的前方额外分配一小块内存来存储数组的元素个数（这个信息对程序员不可见，但编译器会用到）。

`array new`要搭配`array delete`也就是`delete[]`，让编译器知道是要删除一个数组，会**多次调用析构函数**，不然会导致析构函数未调用发生内存泄漏。

# 类的关系

## Composition（组合）

**将其他类的`object`作为当前类的成员变量**，生命周期一致。

其他类的`object`的初始化通过当前类的初始化列表来进行初始化。

``` c++
class HisString : public MyString{
    private:
        Complex c1;
    public:
        HisString(double real = 0, double imag = 0) : c1(real, imag){
            cout<< "执行子类的构造函数" << endl;
        }

        ~HisString(){
            cout << "执行子类的析构函数" <<endl;
        }
};
#endif
```

构造由内而外：先调用内部类的构造函数再调用自己的构造函数；析构由外而内：先调用自己的析构函数再调用内部类的析构函数。

## Delegation（委托）

生命周期不同步。

composition by reference。一个类**用一个指针变量指向另外一个类**。

**point 2 implementation（pimpl）**对外的接口可以指向不同的类，也是编译防火墙。

## Inheritance（继承）

有三种继承方式，最重要的是`:public`继承，表示`is-a`，子类是父类的一种，但是有额外的东西。

``` c++
class HisString : public MyString{
    private:
        Complex c1;
    public:
        HisString(double real = 0, double imag = 0) : c1(real, imag){
            cout<< "执行子类的构造函数" << endl;
        }

        ~HisString(){
            cout << "执行子类的析构函数" <<endl;
        }
};
```

**父类的析构函数需要为虚函数**，否则会出现未定义行为。最有价值的地方是和虚函数搭配，函数是继承的父类的调用权，虚函数是希望子类对函数进行重新定义，纯虚函数是让子类必须进行重新定义`virtual void func() const = 0;`。

**子类的`object`有父类的成分**，可以通过子类的`object`来调用父类的函数。

---

一个父类做很多相同的动作函数，把一个关键的步奏函数延缓实现，写为虚函数，这种做法叫做`Template Method`。在框架设计中十分常见！

---

如果一个类又是一个类的子类又是另外一个类的组合，构造函数和析构函数谁先调用？

既有父类又有组合，先执行父类的构造函数，再执行组合的构造函数，再执行子类的构造函数，而析构则反过来。

# 设计模式

## 单例模式

当前`class`只产生一个`object`，把构造函数和拷贝构造函数都放到`private`里面。

## Composite

委托+继承。

Composite（组合）设计模式是一种**结构型设计模式**，它允许将对象组合成树形结构来表示"部分-整体"的层次结构。Composite模式使得客户端可以统一地处理单个`object`和组合`object`。

- 核心思想：Composite模式的核心是创建一个**统一接口**，让客户端不必区分单个对象（叶子节点）和组合对象（容器节点），从而简化客户端代码。

<img src="pic/composite.png" alt="composite" style="zoom:50%;" />

## Prototype





# 数组与vector

## 定义数组

在C/C++中，以下情况需要显式指明数组大小：

- **不提供初始化列表时（无enum/数组内容）**：如果定义数组时不初始化，必须指定大小。例如：

  ```c
  int arr[10]; // 必须指明大小
  ```

- **数组作为函数参数（且以数组形式声明）**：可以省略第一维的大小（比如`int arr[]`），但其他维度必须指明。例如：

  ```c
  void func(int arr[]);    // 合法，一维数组可以省略大小
  void func(int arr[][10]); // 多维数组必须指明其他维度
  ```

- **数组是局部变量且未初始化**：如果数组是局部变量且未初始化，必须指定大小：

  ```c
  void func() {
      int arr[10]; // 必须指明大小
  }
  ```

- **数组是头文件中的声明**：如果在头文件中声明一个数组（非定义），通常需要指定大小：

  ```c
  // header.h
  extern int arr[10];
  ```

- **C++的堆数组（new[]）**：在C++中用`new`分配数组时必须指明大小：

  ```c
  int *arr = new int[10]; // 必须指明大小
  ```

- 字符串数组的大小会多包含一个`\0`，C++中对字符串处理的函数如`strlen()`，`strcmp()`等在`cstring`库中。

  ```c
  char src[] = "Hello, World!";
  char dest[20] = {0};
  memcopy(dest, src, strlen(src) + 1); // +1 包含 '\0'
  ```

- **字符串数组存储的是指向字符串的指针！**修改字符串数组某个下标的字符串本质是修改这个下标存储的指针指向的地址。

## vector容器

### 本质特性

- **连续内存存储**：和普通数组一样，元素在内存中紧挨着，支持**随机访问**（下标 O (1) 访问）
- **动态扩容**：元素超出当前容量时，自动申请更大内存，迁移元素，释放旧内存

​	当 `size == capacity` 时插入元素，vector 会自动扩容：扩容会重新分配内存→拷贝 / 移动元素→释放旧内存，**迭代器会全部失效**

​	GCC：**1.5 倍扩容**；MSVC：**2 倍扩容**

- **模板类**：可存储任意类型（int、char、结构体、类对象、指针等），**不能直接存引用**
- **尾部操作高效**：尾插 / 尾删 O (1)，中间插入 / 删除 O (n)（需移动元素）

vector 内部通过**三个指针**管理内存：

1. `_Start`：指向数组起始地址
2. `_Finish`：指向**最后一个有效元素的下一个位置**（对应 size）
3. `_EndOfStorage`：指向分配内存的末尾（对应 capacity）

> `size`：实际存储的元素个数
>
> `capacity`：当前已分配的总容量（≥size）

---

 1. 迭代器失效问题（高频坑点）

以下操作会导致迭代器失效：

1. **扩容操作**：push_back、emplace_back、resize、reserve 等（重新分配内存）
2. **中间插入 / 删除**：insert、erase（元素移动，迭代器位置偏移）

> 解决：操作后重新获取迭代器，不要保存旧迭代器

 2. 连续内存特性

- vector 元素一定是**连续内存**，可通过`data()`或`&v[0]`传给 C 语言接口
- 不同于 list（链表，非连续）、deque（分段连续）

 3. 存储对象的要求

- 存储自定义对象时，类需支持**拷贝构造 / 移动构造**（vector 扩容 / 拷贝会用到）
- 不能存储**引用**（引用无独立内存，无法拷贝），可存指针

### 构造函数

| 构造方式      | 语法                          | 说明                                                    |
| ------------- | ----------------------------- | ------------------------------------------------------- |
| 默认构造      | `vector<T> v;`                | 创建空 vector                                           |
| 带大小构造    | `vector<T> v(n);`             | 创建 n 个元素，值为类型默认值（int=0，对象 = 默认构造） |
| 大小 + 初始值 | `vector<T> v(n, val);`        | 创建 n 个值为 val 的元素                                |
| 拷贝构造      | `vector<T> v1(v2);`           | 拷贝 v2 所有元素                                        |
| 移动构造      | `vector<T> v(std::move(v2));` | 转移 v2 资源，v2 变空                                   |
| 迭代器区间    | `vector<T> v(arr, arr+5);`    | 用数组 / 其他容器迭代器初始化                           |
| 初始化列表    | `vector<T> v{1,2,3,4};`       | C++11 直接初始化                                        |

示例：

``` c++
vector<int> v1;                  // 空
vector<int> v2(5);               // 5个0
vector<int> v3(5, 10);           // 5个10
vector<int> v4(v3);              // 拷贝v3
vector<int> v5{1,2,3,4,5};       // 初始化列表
int arr[] = {1,2,3};
vector<int> v6(arr, arr+3);      // 数组初始化
```

### 迭代器（Iterator）

vector 支持**随机访问迭代器**，用于遍历 / 操作元素，常用迭代器函数：

| 函数                    | 作用                                         |
| ----------------------- | -------------------------------------------- |
| `v.begin()`             | 指向**第一个元素**的迭代器                   |
| `v.end()`               | 指向**最后一个元素的下一个位置**（左闭右开） |
| `v.rbegin()`            | 反向迭代器，指向最后一个元素                 |
| `v.rend()`              | 反向迭代器，指向第一个元素前一位置           |
| `v.cbegin()`/`v.cend()` | 常量迭代器（不能修改元素）                   |

``` c++
vector<int> v{1,2,3};
// 1. 迭代器遍历
for(vector<int>::iterator it = v.begin(); it != v.end(); ++it){
    cout << *it << " ";
}
// 2. C++11 范围for
for(auto x : v) cout << x << " ";
// 3. 下标遍历（最常用）
for(int i=0; i<v.size(); ++i) cout << v[i] << " ";
```

### 操作函数

#### 核心容量函数

| 函数                | 作用                                 | 时间复杂度 |
| ------------------- | ------------------------------------ | ---------- |
| `v.size()`          | 获取实际元素个数                     | O(1)       |
| `v.capacity()`      | 获取当前总容量                       | O(1)       |
| `v.empty()`         | 判断是否为空（size=0）               | O(1)       |
| `v.resize(n)`       | 调整**元素个数**为 n，多删少补默认值 | O(n)       |
| `v.resize(n, val)`  | 调整大小为 n，新增元素值为 val       | O(n)       |
| `v.reserve(n)`      | 预分配**容量**为 n，不改变 size      | O(n)       |
| `v.shrink_to_fit()` | 释放多余容量，capacity=size          | O(n)       |

#### 随机访问

vector 支持**随机访问**，效率 O (1)：

| 函数        | 作用               | 注意                                   |
| ----------- | ------------------ | -------------------------------------- |
| `v[i]`      | 访问第 i 个元素    | **不检查越界**，越界行为未定义         |
| `v.at(i)`   | 访问第 i 个元素    | **检查越界**，越界抛`out_of_range`异常 |
| `v.front()` | 获取第一个元素     | 空 vector 调用崩溃                     |
| `v.back()`  | 获取最后一个元素   | 空 vector 调用崩溃                     |
| `v.data()`  | 返回底层数组首地址 | 等价于 & v [0]，用于兼容 C 接口        |

#### 增删改

**尾部：**

| 函数                   | 作用                                     |
| ---------------------- | ---------------------------------------- |
| `v.push_back(val)`     | 尾部插入元素 val                         |
| `v.pop_back()`         | 删除尾部元素，无返回值                   |
| `v.emplace_back(args)` | C++11**原位构造**元素，比 push_back 高效 |

`emplace_back` vs `push_back`：

- push_back：先构造临时对象，再拷贝 / 移动到 vector
- emplace_back：直接在 vector 内存中构造对象，**少一次拷贝 / 移动**

---

**任意位置：**

| 函数                    | 作用                                | 时间复杂度          |
| ----------------------- | ----------------------------------- | ------------------- |
| `v.insert(pos, val)`    | 在迭代器 pos 处插入 val             | O(n)                |
| `v.insert(pos, n, val)` | 在 pos 处插入 n 个 val              | O(n)                |
| `v.erase(pos)`          | 删除迭代器 pos 处元素               | O(n)                |
| `v.erase(beg, end)`     | 删除区间 [beg,end) 元素             | O(n)                |
| `v.clear()`             | 清空所有元素，size=0，capacity 不变 | O(n)                |
| `v.swap(v2)`            | 交换两个 vector 的所有内容          | O (1)（仅交换指针） |
| `v.assign(n, val)`      | 赋值 n 个 val，覆盖原有元素         | O(n)                |
| `v.assign(beg, end)`    | 用迭代器区间赋值                    | O(n)                |

示例：

``` c++
vector<int> v{1,2,3};
v.push_back(4);      // {1,2,3,4}
v.pop_back();        // {1,2,3}
v.insert(v.begin()+1, 9); // {1,9,2,3}
v.erase(v.begin());  // {9,2,3}
v.clear();           // 空
```



# 字符串类

`std::string` 是 C++ 标准库提供的字符串类，本质是一个**动态字符数组**，**自动管理内存**（无需手动 malloc/free 或 new/delete），支持丰富的字符串操作函数。

 核心优势（对比 C 语言 `char*`）：

- 自动内存管理，不会内存泄漏 / 越界；

- 长度可变，直接拼接、修改；

- 内置几十种操作函数，无需手写逻辑；

- 类型安全，支持运算符重载（`+`/`==`/`[]` 等）。

---

## 声明

``` c++
#include <iostream>
#include <string>
using namespace std;

int main() {
    // 1. 默认初始化：空字符串
    string s1;               

    // 2. 直接赋值初始化
    string s2 = "Hello C++"; 

    // 3. 构造函数初始化
    string s3("Hello World");

    // 4. 拷贝初始化：用已有字符串创建新字符串
    string s4 = s3;         

    // 5. 重复字符初始化：n个相同字符
    string s5(5, 'a');       // 结果："aaaaa"

    
    // 6. 子串初始化：从s2的第2个字符开始，取4个字符
    string s6(s2, 2, 4);     // 结果："llo "

    
    // 7. 字符数组初始化
    char arr[] = "test";
    string s7(arr);          // 结果："test"
    // 8. 部分字符数组初始化
    string s8(arr, 2);       // 结果："te"

    // 9. 移动初始化（C++11，高效）
    string s9 = move(s2);    

    return 0;
}
```

## 成员函数

### 基础属性

| 函数 / 属性           | 作用                       | 返回值                            |
| --------------------- | -------------------------- | --------------------------------- |
| `size()` / `length()` | 获取字符串长度（字符个数） | `size_t`（无符号整数）            |
| `empty()`             | 判断是否为空字符串         | `bool`（空返回 true，非空 false） |
| `max_size()`          | 获取字符串最大支持长度     | `size_t`                          |

---

访问方式：支持两种访问方式，**安全性不同**：

| 方式   | 用法          | 特点                               |
| ------ | ------------- | ---------------------------------- |
| `[]`   | `s[index]`    | 越界**不抛异常**，直接崩溃（高效） |
| `at()` | `s.at(index)` | 越界**抛异常**，安全（调试推荐）   |

---

### 拼接

支持**操作符重载**和**成员函数**，两种方式等价：

``` c++
string s1 = "Hello";
string s2 = " C++";

// 1. 运算符 += （推荐，原地拼接，高效）
s1 += s2;        // s1 = "Hello C++"
s1 += "!!!";     // 拼接常量字符串
s1 += 'A';       // 拼接单个字符

// 2. 运算符 + （生成新字符串，不修改原串）
string s3 = s1 + " World";

// 3. append() 成员函数
s1.append(s2);   // 等价于 +=
s1.append("123", 2); // 拼接前2个字符："12"
```

---

### 比较

支持所有**比较运算符**，也可使用 `compare()` 函数：

- 规则：按**ASCII 码字典序**比较

- 返回值：`true`/`false`（运算符），`0`/`正数`/`负数`（compare）

``` c++
string a = "Apple";
string b = "Banana";
string c = "Apple";

// 运算符比较（推荐）
cout << (a == c);  // 1（相等）
cout << (a < b);   // 1（A < B）
cout << (a != b);  // 1

// compare() 函数
a.compare(c);      // 0（相等）
a.compare(b);      // 负数（a < b）
b.compare(a);      // 正数（b > a）
```

### 查找

查找是字符串最常用操作，**返回找到的第一个字符下标**，未找到返回 `string::npos`（C++ 字符串常量，代表「无效位置」值为 `size_t` 类型的最大值（`-1`））。

| 函数                 | 作用                    |
| -------------------- | ----------------------- |
| `find(str)`          | 从左往右查找子串 / 字符 |
| `rfind(str)`         | 从右往左查找子串 / 字符 |
| `find_first_of(str)` | 查找第一个匹配的字符    |
| `find_last_of(str)`  | 查找最后一个匹配的字符  |

``` c++
string s = "Hello C++ C++";

// 1. find：查找子串
size_t pos = s.find("C++");  // 找到，返回下标：6
pos = s.find("Java");        // 未找到，返回 string::npos

// 2. rfind：反向查找
pos = s.rfind("C++");       // 返回最后一个位置：10

// 3. 查找单个字符
pos = s.find('o');          // 返回：4

// 关键：判断是否找到（必须和npos比较）
if (s.find("Java") == string::npos) {
    cout << "未找到目标字符串";
}
```

### 截取

**语法**：`s.substr(起始下标, 截取长度)`，省略长度：截取到字符串末尾。注意：**不是「结束下标」，是「长度」**。

``` c++
string s = "Hello World";
string sub1 = s.substr(6, 5);  // 从下标6开始，截5个："World"
string sub2 = s.substr(6);     // 从下标6到末尾："World"
```

### 修改

插入、删除、替换、清空。

| 函数                     | 作用                   |
| ------------------------ | ---------------------- |
| `insert(pos, str)`       | 在指定位置插入字符串   |
| `erase(pos, len)`        | 删除指定位置的字符     |
| `replace(pos, len, str)` | 替换指定位置的字符     |
| `clear()`                | 清空字符串（变为空串） |

### 与C风格字符串互转

C++ 兼容 C 语言，`string` 提供函数转换为 `const char*`（C 风格字符串）：

`string → const char*`：

``` c++
string s = "Hello";
// 两个函数等价，返回C风格字符串指针
const char* cstr1 = s.c_str();  // 推荐
const char* cstr2 = s.data();
```

`char* → string`：直接赋值即可。

``` c++
char *arr = "C String";
string s = arr;  // 自动转换
```

## 非成员函数

### 字符串 ↔ 数字

C++11 提供了**内置转换函数**，无需手写算法，完美支持数字与字符串互转。

| 函数       | 作用               |
| ---------- | ------------------ |
| `stoi(s)`  | string → int       |
| `stol(s)`  | string → long      |
| `stoll(s)` | string → long long |
| `stof(s)`  | string → float     |
| `stod(s)`  | string → double    |

``` c++
string s1 = "123";
string s2 = "3.14";
int a = stoi(s1);       // a = 123
double b = stod(s2);    // b = 3.14
```

### 数字 → 字符串

**函数**：`to_string(数字)`

``` c++
int a = 100;
double b = 3.1415;
string s1 = to_string(a);  // "100"
string s2 = to_string(b);  // "3.141500"
```

### 读取完整字符串

`cin` 读取字符串会**自动截断空格 / 换行**，`getline` 可以读取**包含空格的整行**：

``` c++
string s;
// 读取一整行（直到回车）
getline(cin, s);  
// 输入：Hello World
// 输出：Hello World
```

## 遍历方式

1.普通 for 循环（最通用）：

``` c++
string s = "ABC";
for (int i = 0; i < s.size(); i++) {
    cout << s[i] << " ";
}
// 输出：A B C
```

2.范围 for 循环（C++11，简洁)：

``` c++
for (char ch : s) {
    cout << ch << " ";
}
```

3.迭代器方式：

``` c++
for (string::iterator it = s.begin(); it != s.end(); it++) {
    cout << *it << " ";
}
```



# lambda

## 定义

在C++中，Lambda 表达式是一种匿名函数对象，它可以在需要时定义并使用（**就地定义、使用，拉满封装性**）。

``` c++
/* 最基本的lambda定义 */
int main() {
    auto lambda = []() { std::cout << "Hello, Lambda!" << std::endl; };
    lambda(); // 输出：Hello, Lambda!
    return 0;
}

auto dfs = [&] (this auto&& dfs, int i, int j) ->void{};  // c++23
/*******************************************************/
#include<functional>
function<void<int, int>> dfs = [&](int i, int j){};  // c++17
```

​	在C++的Lambda表达式中，“捕获”（Capture）是指将Lambda表达式定义时所在作用域中的变量“捕获”到Lambda表达式内部，以便在Lambda表达式中使用这些变量。捕获的作用主要是解决Lambda表达式如何访问外部变量的问题。

``` c++
 // 按值捕获当前作用域中所有变量
    auto lambda = [=]() {
        std::cout << "a = " << a << ", b = " << b << std::endl;
    };

// 按引用捕获当前作用域中所有变量
    auto lambda = [&]() {
        std::cout << "a = " << a << ", b = " << b << std::endl;
    };

// 按引用捕获所有变量，但变量a按值捕获
    auto lambda = [&, a]() {
        std::cout << "a = " << a << ", b = " << b << std::endl;
    };
```

## 例程

``` c++
void MYACTUA::enqueue_discrete_command(const ControlCommand& cmd)
{
    /* 就地定义、使用，拉满封装性 */
    auto enqueue_one = [this, &cmd](int idx) {
        if (idx < 0 || idx >= static_cast<int>(_motors.size())) {
            return;
        }
        DiscreteCommand pending(cmd.type, cmd.mode);
        pending.next_retry_cycle = control_cycle_;
        discrete_cmd_queues_[idx].push_back(pending);
    };


    /* 小于 0 表示对所有电机执行命令 */
    if (cmd.slave_index < 0) {
        for (int i = 0; i < static_cast<int>(_motors.size()); ++i) {
            enqueue_one(i);
        }
    } else {
        enqueue_one(cmd.slave_index);
    }
}
```



# 函数指针

函数指针本质上是一个**指针。**使得可以通过函数指针来调用函数。主要用于**回调函数**。

- **通过函数指针来调用原函数**：

``` c
int add(int a, int b)
{ 
    return a + b; 
}

返回类型 (*指针变量名)(参数列表);
int (*func_ptr) (int, int); // 声明一个指向函数的指针变量

func_ptr = add;  // 或 funcPtr = &add;

int result = funcPtr(3, 4);  // 或 (*funcPtr)(3, 4);
```
-  回调函数允许**运行时动态决定行为**，而无需修改原有代码。**因此函数指针通常用于回调函数的实现：**
``` c
typedef void (*TimeoutCallback)(void);

void setTimer(int seconds, TimeoutCallback callback) {
    // 等待seconds秒后...
    callback();  // 调用用户提供的函数
}

// 用户自定义回调
void onTimeout() {
    printf("Time's up!\n");
}

setTimer(5, onTimeout);  // 5秒后打印 "Time's up!"
```

# 指针与地址

**指针的本质**：指针是一种**变量**，它的值是**内存地址**。

指针类型的不同只会对指针解引用时读取到的数据产生影响，对指针本身指向的地址无影响，但影响指针指向地址的运算步长。

``` c
int num = 0x12345678;
char* p_char = (char*)&num;
int*  p_int  = &num;

printf("char视角: %x\n", *p_char);  // 输出 78（仅读取最低1字节）
printf("int视角: %x\n", *p_int);    // 输出 12345678（读取4字节）
```

内存采用小端布局：

``` c
地址     0x1000 0x1001 0x1002 0x1003
数据       78    56    34    12
```

指针类型影响指针的运算步长：

``` c
p_int++;   // 地址增加 sizeof(int)（如 +4）
p_char++;  // 地址增加 sizeof(char)（如 +1）
```

# Effective C++

## 尽可能用替代#define

使用`#define`定义的符号名称从未被编译器看见，在编译器开始处理源码之前就被预处理器一走了，因此使用`#define`定义的符号名称有可能从未进入符号表内，因此如果使用`#define`定义的符号但获得了一个错误的信息，可能会难以调试。

解决之道是使用一个常量来替换宏：

``` c++
const double AspectRatio = 1.653;
```

作为一个语言常量这个符号肯定会被编译器看到，当然就会进入符号表内；而对于浮点常量而言使用常量可能会比使用`#define`更小的代码量，因为预处理器的盲目替换可能导致目标码出现多份`1.653`。

---

如果是替换指针则要声明为**指针常量**：

``` c++
char* const p;
```

---

如果是 class 的专属常量，为了将常量的作用域限制在 class 中，必须让它成为 class 的一个成员，而为了**确保只有一个实体**，需要加上`static`：

``` c++
class GamePlayer{
private:
    // 只有整型静态常量（int/char/bool 等）可以在类内直接写 =5；
	// 如果是 static const double、static std::string，不能类内初始化，必须类外定义时赋值：
    static const int NumTurns = 5;  // 仍然为声明式
    ...
}
```

如果需要取地址或者编译器坚持要看到定义式，则需要另外提供定义式：

``` c++
const int GamePlayer::NumTurns;
```

`static` 是**声明关键字**（和`extern`一样），不是**定义关键字**（`const`、`volatile`）

- 类内写 `static`：是给编译器看的声明修饰符作用只有一个：✅ 告诉编译器：这个成员不是普通成员变量，是类级别的静态成员（全局唯一、不依附对象）。

- **类外不写 `static`**：编译器**已经通过类内的声明知道它是静态的**，不需要重复声明这个属性。

C++ 标准明确要求：**类的静态成员变量，在类外进行定义时，绝对不能加 `static` 关键字！**加了会直接**编译报错**，因为这是语法冲突。


全局作用域的 `static` 含义是：**内部链接（只在当前文件可见）**；而**类的静态成员**默认是**外部链接**（整个程序共享）。

如果类外定义写 `static const int GamePlayer::NumTurns;`，会让编译器混淆：你到底是要「类静态成员」，还是「全局静态变量」？

---

并且`#define`无视作用域，因此无法通过`#define`创建一个 class 的专属常量，一旦宏被定义，它在之后的处理都有效，除非在某处被`#undef`。如果编译器不允许完成 in class 初值设定，可以改用`enum`：

``` c++
class GamePlayer{
private:
    enum {NumTurns = 5};
    ...
}
```

而`enum`比较像`#define`，因为取一个`enum`的地址是非法的，但是取一个`const`的地址是合法的。

