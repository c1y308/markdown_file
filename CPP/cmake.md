# RCom 项目 CMake 使用教程

---

## 目录

1. [项目结构概览](#1-项目结构概览)
2. [CMake 基本概念与核心语法](#2-cmake-基本概念与核心语法)
3. [项目 CMakeLists.txt 组织结构详解](#3-项目-cmakeliststxt-组织结构详解)
4. [常用 CMake 命令实战案例](#4-常用-cmake-命令实战案例)
5. [构建过程完整步骤](#5-构建过程完整步骤)
6. [常见问题与解决方案](#6-常见问题与解决方案)
7. [最佳实践总结](#7-最佳实践总结)

---

## 1. 项目结构概览

> **模块依赖关系**：`base` → `common` → `log` / `transport_commom`，`time` 和 `serialize` 相对独立。

---

## 2. CMake 基本概念与核心语法

### 2.1 三个核心概念

| 概念                 | 说明                        | 项目中对应                                 |
| -------------------- | --------------------------- | ------------------------------------------ |
| **Target（目标）**   | 构建产物：可执行文件或库    | `RCom_base`、`RCom_log`、`log_test` 等     |
| **Property（属性）** | 目标的元数据                | include 路径、编译选项、链接库             |
| **Command（命令）**  | CMakeLists.txt 中的函数调用 | `add_library()`、`target_link_libraries()` |

**关键思想**：CMake 是"面向目标"的。不要全局修改编译标志，而是通过 `target_*()` 命令精确作用到每个目标。这样做的好处是依赖关系自动传递。

### 2.2 三种库类型（项目全部使用到）

```cmake
# 1. INTERFACE — 只有头文件，不需要编译 .cpp（如 base, common）
add_library(RCom_base INTERFACE)

# 2. SHARED — 编译为 .so 动态库（如 log, time, serialize）
add_library(RCom_log SHARED logger.cpp)

# 3. STATIC — 编译为 .a 静态库（项目目前未使用，但可选）
add_library(mylib STATIC foo.cpp)
```

### 2.3 三种链接可见性

这个库的`target_include_directories` 是否传递给链接这个库的目标？

```cmake
# PUBLIC   — 自己和依赖者都能使用（最常见的传递方式）
target_link_libraries(RCom_log PUBLIC RCom_common)

# PRIVATE  — 只有自己能使用，不传递给依赖者
target_link_libraries(log_test PRIVATE RCom_log)

# INTERFACE — 自己不使用，仅传递给依赖者（纯头文件库用这个）
target_link_libraries(RCom_common INTERFACE RCom_base)
```

> **本项目中的典型应用**：
> - `RCom_common` → `RCom_base` 用 `INTERFACE`，因为两者都是纯头文件
> - `RCom_log` → `RCom_common` 用 `PUBLIC`，因为 `logger.hpp` 公开包含了 `common/macros.hpp`
> - 测试程序 → 被测模块用 `PRIVATE`，测试依赖不对外暴露

### 2.4 变量、选项与条件判断

```cmake
# 设置变量
set(CMAKE_CXX_STANDARD 17)

# 定义布尔选项，默认为 OFF，可通过 -D 覆盖
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)

# 条件判断
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(-Wall -Wextra -Wpedantic)
endif()
```

---

## 3. 项目 CMakeLists.txt 组织结构详解

### 3.1 顶层 CMakeLists.txt —— 项目全局配置

> 文件位置：[CMakeLists.txt](file:///home/cat/RCom/CMakeLists.txt)

```cmake
cmake_minimum_required(VERSION 3.14)

# 定义项目名称、版本和使用的语言。
project(RCom VERSION 1.0.0 LANGUAGES CXX)

# 给 clangd / IDE 生成准确的编译数据库。
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 统一设置整个项目的 C++ 标准。
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# GCC/Clang 编译时打开常见警告，帮助尽早发现代码问题。
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# 可选的运行时检查工具，默认关闭，需要时通过 -DENABLE_ASAN=ON 打开。
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
option(ENABLE_TSAN "Enable ThreadSanitizer" OFF)
option(BUILD_GTESTS "Build GoogleTest test programs" ON)

if(ENABLE_ASAN)
    add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address)
endif()
if(ENABLE_TSAN)
    add_compile_options(-fsanitize=thread -fno-omit-frame-pointer)
    add_link_options(-fsanitize=thread)
endif()

# 添加各个功能模块。每个子目录负责定义自己的库目标。
add_subdirectory(base)
add_subdirectory(common)
add_subdirectory(log)
add_subdirectory(time)
add_subdirectory(serialize)
add_subdirectory(transport)

# 按需构建 GoogleTest 测试程序。
if(BUILD_GTESTS)
    add_subdirectory(test)
endif()
```

**逐段解读：**

| 行                                          | 作用                         | 为何这样设置                                                |
| ------------------------------------------- | ---------------------------- | ----------------------------------------------------------- |
| `cmake_minimum_required(VERSION 3.14)`      | 声明最低 CMake 版本          | FetchContent 需要 3.14+                                     |
| `project(RCom VERSION 1.0.0 LANGUAGES CXX)` | 项目元信息                   | CXX 表示 C++ 项目                                           |
| `set(CMAKE_EXPORT_COMPILE_COMMANDS ON)`     | 生成 `compile_commands.json` | IDE（VS Code + clangd）的代码补全、跳转依赖此文件           |
| `set(CMAKE_CXX_STANDARD 17)`                | 全局 C++17                   | 项目使用了 `if constexpr`、`std::enable_if_t` 等 C++17 特性 |
| `-Wall -Wextra -Wpedantic`                  | 编译警告级别                 | 尽早暴露潜在 bug                                            |
| `option(ENABLE_ASAN ...)`                   | AddressSanitizer 开关        | 默认关闭，调试内存问题时 `-DENABLE_ASAN=ON`                 |
| `option(BUILD_GTESTS ...)`                  | 测试开关                     | 默认开启，CI 中可以按需控制                                 |

### 3.2 base/CMakeLists.txt —— INTERFACE 库（纯头文件）

> 文件位置：[base/CMakeLists.txt](file:///home/cat/RCom/base/CMakeLists.txt)

```cmake
# base 目录目前都是头文件模板/内联实现，因此使用 INTERFACE 库。
add_library(RCom_base INTERFACE)

# INTERFACE include 路径会传递给链接 RCom_base 的目标。
target_include_directories(RCom_base INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})
```

**设计要点**：
- `base/` 下全部是 `.hpp` 文件（模板、内联函数），不需要编译，所以用 `INTERFACE`
- 任何链接 `RCom_base` 的目标都会自动获得 `base/` 目录到 include 路径
- 比如在测试代码中 `#include "unbounded_queue.hpp"` 能直接找到

### 3.3 common/CMakeLists.txt —— INTERFACE 库（依赖 base）

> 文件位置：[common/CMakeLists.txt](file:///home/cat/RCom/common/CMakeLists.txt)

```cmake
# common 目前也是头文件工具集合，因此使用 INTERFACE 库。
add_library(RCom_common INTERFACE)

# 暴露 common 目录
target_include_directories(RCom_common INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})

# common/macros.hpp 依赖 base/macros.hpp，所以把依赖关系传递出去。
target_link_libraries(RCom_common INTERFACE RCom_base)
```

**依赖传递链**：任何链接 `RCom_common` 的目标，自动获得：
1. `common/` 目录的 include 路径
2. `base/` 目录的 include 路径（通过 INTERFACE 传递）
3. 这就是为什么 [logger.hpp](file:///home/cat/RCom/log/logger.hpp#L4) 可以写 `#include "../common/macros.hpp"`

### 3.4 log/CMakeLists.txt —— SHARED 库（有 .cpp 编译单元）

> 文件位置：[log/CMakeLists.txt](file:///home/cat/RCom/log/CMakeLists.txt)

```cmake
# logger.cpp 有实际编译单元，因此创建动态库目标。
add_library(RCom_log SHARED
            logger.cpp)

# PUBLIC 表示 RCom_log 自己和链接它的目标都能使用该头文件目录。
target_include_directories(RCom_log PUBLIC
                            ${CMAKE_CURRENT_SOURCE_DIR})

# logger.hpp 依赖 common/macros.hpp，因此公开链接 RCom_common。
target_link_libraries(RCom_log PUBLIC
                      RCom_common)
```

**设计要点**：
- `logger.cpp` 包含实现代码，必须编译 → 用 `SHARED`
- `logstream.hpp` 没有列在 `add_library` 中，但它是 `logger.hpp` 内部引用的，通过 PUBLIC include 目录自然可见
- `PUBLIC` 链接确保依赖者也能用 `common` 的头文件（虽然不强制，但遵循"显式表达依赖"原则）

### 3.5 test/CMakeLists.txt —— 测试基础设施

> 文件位置：[test/CMakeLists.txt](file:///home/cat/RCom/test/CMakeLists.txt)

```cmake
# FetchContent 可以在配置阶段下载/引入第三方 CMake 项目。
include(FetchContent)

# 引入 GoogleTest，并把 gtest_main 链接到测试程序。
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0
)
FetchContent_MakeAvailable(googletest)

# 封装一个小函数，减少每个测试目标重复写 add_executable 和 gtest_main。
function(rcom_add_gtest target)
    add_executable(${target} ${ARGN})
    target_link_libraries(${target} PRIVATE gtest_main)
endfunction()

# 每个测试子目录对应一个被测试模块。
add_subdirectory(base_test)
add_subdirectory(common_test)
add_subdirectory(log_test)
add_subdirectory(time_test)
add_subdirectory(serialize_test)
```

**核心设计模式**：自定义函数 `rcom_add_gtest`

```cmake
# 使用前（冗余）：
add_executable(log_test log_test.cpp)
target_link_libraries(log_test PRIVATE gtest_main)
target_link_libraries(log_test PRIVATE RCom_log)

# 使用后（简洁）：
rcom_add_gtest(log_test log_test.cpp)
target_link_libraries(log_test PRIVATE RCom_log)
```

`${ARGN}` 捕获所有额外参数，所以 `log_test.cpp` 会传给 `add_executable`。

### 3.6 test/base_test/CMakeLists.txt —— 一对多测试

> 文件位置：[test/base_test/CMakeLists.txt](file:///home/cat/RCom/test/base_test/CMakeLists.txt)

```cmake
rcom_add_gtest(unbounded_queue_test unbounded_queue_test.cpp)
target_link_libraries(unbounded_queue_test PRIVATE RCom_base)

rcom_add_gtest(bounded_queue_test bounded_queue_test.cpp)
target_link_libraries(bounded_queue_test PRIVATE RCom_base)

rcom_add_gtest(macros_test macros_test.cpp)
target_link_libraries(macros_test PRIVATE RCom_base)

rcom_add_gtest(atomic_rw_lock_test atomic_rw_lock_test.cpp)
target_link_libraries(atomic_rw_lock_test PRIVATE RCom_base)
```

**设计要点**：base 模块有多个测试，每个都是独立的可执行文件。好处：
- 单个测试失败不影响其他测试的运行
- 可以单独运行：`./unbounded_queue_test`
- 并行测试：`ctest -j4`

---

## 4. 常用 CMake 命令实战案例

### 4.1 添加新模块

假设要添加一个 `crypto/` 加密模块：

**步骤 1**：在顶层 [CMakeLists.txt](file:///home/cat/RCom/CMakeLists.txt#L40) 中添加：

```cmake
add_subdirectory(crypto)
```

**步骤 2**：创建 `crypto/CMakeLists.txt`：

```cmake
add_library(RCom_crypto SHARED
    aes.cpp
    rsa.cpp
)

target_include_directories(RCom_crypto PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

# 如果依赖 common
target_link_libraries(RCom_crypto PUBLIC RCom_common)
```

### 4.2 添加新测试

**步骤 1**：在 `test/CMakeLists.txt` 中添加：

```cmake
add_subdirectory(crypto_test)
```

**步骤 2**：创建 `test/crypto_test/CMakeLists.txt`：

```cmake
rcom_add_gtest(aes_test aes_test.cpp)
target_link_libraries(aes_test PRIVATE RCom_crypto)
```

### 4.3 给模块添加源文件

如果 `log` 模块新增 `log_rotation.cpp`，修改 [log/CMakeLists.txt](file:///home/cat/RCom/log/CMakeLists.txt#L2)：

```cmake
add_library(RCom_log SHARED
            logger.cpp
            log_rotation.cpp)   # ← 新增
```

### 4.4 条件编译

如果某功能仅在 Linux 上可用：

```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    target_compile_definitions(RCom_log PRIVATE HAS_EPOLL)
    target_sources(RCom_log PRIVATE epoll_watcher.cpp)
endif()
```

---

## 5. 构建过程完整步骤

### 5.1 标准构建（Release 模式）

```bash
# 1. 进入项目根目录
cd /home/cat/RCom

# 2. 创建构建目录（与源码分离）
mkdir -p build && cd build

# 3. 配置（生成 Makefile）
cmake .. -DCMAKE_BUILD_TYPE=Release

# 4. 编译（-j$(nproc) 使用全部 CPU 核心）
cmake --build . -j$(nproc)
```

### 5.2 Debug + AddressSanitizer 构建

```bash
cd /home/cat/RCom
mkdir -p build/debug && cd build/debug

cmake ../.. \
    -DCMAKE_BUILD_TYPE=Debug \
    -DENABLE_ASAN=ON

cmake --build . -j$(nproc)
```

### 5.3 仅构建不测试（跳过 GoogleTest 下载）

```bash
cmake .. -DBUILD_GTESTS=OFF
cmake --build . -j$(nproc)
```

### 5.4 运行测试

```bash
cd /home/cat/RCom/build

# 运行所有测试
ctest --output-on-failure

# 运行特定测试
ctest -R log_test --output-on-failure

# 或直接执行
./test/log_test/log_test
```

### 5.5 完整开发流程一览

```bash
# === 首次配置 ===
cd /home/cat/RCom
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Debug -DENABLE_ASAN=ON

# === 日常开发迭代 ===
# 修改源码后，只需：
cmake --build . -j$(nproc)

# === 运行测试 ===
ctest --output-on-failure

# === 新增文件后 ===
# 如果新增了 .cpp 或 CMakeLists.txt，重新配置：
cmake ..
cmake --build . -j$(nproc)
```

> **提示**：修改 `.cpp` 文件不需要重新运行 `cmake ..`，只需 `cmake --build .`。只有修改 CMakeLists.txt 或新增源文件时才需要重新配置。

---

## 6. 常见问题与解决方案

### 6.1 找不到头文件（fatal error: xxx.hpp: No such file or directory）

**症状**：**原因**：目标没有链接正确的库，导致 include 路径不完整。

**排查**：对照依赖关系图检查 `target_link_libraries`：

| 如果代码中写了                           | 需要链接         |
| ---------------------------------------- | ---------------- |
| `#include "macros.hpp"` 且文件在 base/   | `RCom_base`      |
| `#include "macros.hpp"` 且文件在 common/ | `RCom_common`    |
| `#include "logger.hpp"`                  | `RCom_log`       |
| `#include "../common/macros.hpp"`        | 仅限模块内部使用 |

**解决**：在测试的 CMakeLists.txt 中确保有 `target_link_libraries(test_target PRIVATE RCom_xxx)`。

### 6.2 FetchContent 下载 GoogleTest 失败（网络问题）

**症状**：**解决方式 1**：跳过测试构建
```bash
cmake .. -DBUILD_GTESTS=OFF
```

**解决方式 2**：手动安装系统 gtest
```bash
sudo apt install libgtest-dev
# 然后修改 test/CMakeLists.txt，用 find_package 代替 FetchContent
```

### 6.3 未定义引用（undefined reference to ...）

**症状**：`ENABLE_ASAN=ON` 和 `ENABLE_TSAN=ON` 只能二选一。

### 6.5 新增 .cpp 文件后未重新配置

**症状**：新文件没有被编译。

**原因**：CMake 在配置阶段确定文件列表，新增文件后必须重新运行 `cmake ..`。

**解决**：
```bash
cmake ..
cmake --build . -j$(nproc)
```

### 6.6 IDE 中代码跳转/补全不工作

**症状**：VS Code 中 clangd 无法跳转、补全。

**原因**：缺少 `compile_commands.json`。

**解决**：
```bash
# 1. 确保顶层 CMakeLists.txt 中有此行（项目已配置）
#    set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 2. 重新生成
cd build && cmake ..

# 3. 如果 build/ 不在项目根目录，创建符号链接
cd /home/cat/RCom
ln -sf build/compile_commands.json .
```

### 6.7 transport 模块未导出到顶层

**现状**：[transport/CMakeLists.txt](file:///home/cat/RCom/transport/CMakeLists.txt) 只包含 `add_subdirectory(commom)`，没有创建 transport 层级的聚合目标。

**解决方法**（如需聚合）：
```cmake
add_library(RCom_transport INTERFACE)
target_link_libraries(RCom_transport INTERFACE RCom_transport_commom)
```

---

## 7. 最佳实践总结

### 7.1 本项目的 CMake 设计原则

| 原则                     | 具体做法                                | 收益                       |
| ------------------------ | --------------------------------------- | -------------------------- |
| **一个模块一个库目标**   | `RCom_log`、`RCom_time` 各为一个 target | 依赖关系清晰，可单独编译   |
| **头文件库用 INTERFACE** | `base/`、`common/` 不编译               | 零编译开销，纯头文件传递   |
| **依赖用 PUBLIC 表达**   | `RCom_log → RCom_common`                | 自动传递 include 路径      |
| **测试用自定义函数封装** | `rcom_add_gtest()`                      | 减少样板代码               |
| **源码与构建分离**       | `mkdir build && cd build`               | 不污染源码目录，多配置共存 |
| **选项开关控制可选功能** | `option(ENABLE_ASAN ...)`               | 按需启用，默认关闭         |

### 7.2 新增模块检查清单

- [ ] 创建 `module/CMakeLists.txt`
- [ ] 在顶层 [CMakeLists.txt](file:///home/cat/RCom/CMakeLists.txt#L37-L41) 添加 `add_subdirectory(module)`
- [ ] 确定库类型：纯头文件 → `INTERFACE`，有 `.cpp` → `SHARED`/`STATIC`
- [ ] 通过 `target_include_directories` 暴露头文件路径
- [ ] 通过 `target_link_libraries` 声明对其他模块的依赖
- [ ] 创建 `test/module_test/CMakeLists.txt` 并注册到 [test/CMakeLists.txt](file:///home/cat/RCom/test/CMakeLists.txt#L22-L26)
- [ ] 验证：`cmake .. && cmake --build .` 通过

### 7.3 命名约定

| 类型       | 命名规范                  | 示例                                |
| ---------- | ------------------------- | ----------------------------------- |
| 库目标名   | `RCom_<module>`           | `RCom_base`、`RCom_log`             |
| 测试目标名 | `<module>_<feature>_test` | `bounded_queue_test`、`macros_test` |
| CMake 选项 | `大写_下划线`             | `ENABLE_ASAN`、`BUILD_GTESTS`       |
| 自定义函数 | `rcom_<动作>`             | `rcom_add_gtest`                    |

### 7.4 依赖关系快速参考图

                RCom_base (INTERFACE)
                    ↑
             (INTERFACE link)
                    ↑
                RCom_common (INTERFACE)
               ↗               ↖
      (PUBLIC link)       (PRIVATE link)
           ↗                   ↖
     RCom_log (SHARED)    RCom_transport_commom (SHARED)
    
     RCom_time (SHARED)         RCom_serialize (SHARED)
     ↑        							↑    
     time_test (EXE)			serialize_test (EXE)
> 箭头方向：`→` 表示"链接了"，依赖自动向下传递。例如 `log_test` 链接 `RCom_log`，自动获得 `RCom_common` 和 `RCom_base` 的头文件路径。

---

## 附录：快速命令参考卡

```bash
# === 首次构建 ===
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . -j$(nproc)

# === Debug + ASan ===
cmake .. -DCMAKE_BUILD_TYPE=Debug -DENABLE_ASAN=ON

# === 跳过测试 ===
cmake .. -DBUILD_GTESTS=OFF

# === 增量编译 ===
cmake --build . -j$(nproc)

# === 运行所有测试 ===
ctest --output-on-failure

# === 运行单个测试 ===
ctest -R log_test --output-on-failure
# 或
./test/log_test/log_test

# === 清理重新构建 ===
cd .. && rm -rf build && mkdir build && cd build
cmake .. && cmake --build . -j$(nproc)

# === 重新配置（修改了 CMakeLists.txt 后） ===
cmake .. && cmake --build . -j$(nproc)

# === 查看所有可用目标 ===
cmake --build . --target help
```

---

> **版本信息**：本教程基于 RCom v1.0.0（CMake 3.14+，C++17），2026 年 7 月编制。