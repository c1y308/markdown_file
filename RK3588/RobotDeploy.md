# 重构内核

## 卸载原内核

``` bash
dpkg --list | grep -E "linux-image-[0-9]|linux-headers-[0-9]|linux-modules-[0-9]"

hi  linux-headers-5.10.209-rk3588   5.10.209-rk3588-4    arm64        Linux kernel headers for 5.10.209-rk3588 on arm64
hi  linux-image-5.10.209-rk3588     5.10.209-rk3588-4    arm64        Linux kernel, version 5.10.209-rk3588

sudo apt remove linux-headers-6.1.99-rk3588 linux-image-6.1.99-rk3588
```

## 构建新内核

 [野火\]嵌入式Linux镜像构建与部署——基于LubanCat-RK系列板卡 文档 (embedfire.com)](https://doc.embedfire.com/linux/rk356x/build_and_deploy/zh/latest/building_image/lubancat_sdk/lubancat_gen_sdk.html)

![image-20260327174328726](./assets/SDK.png)

### 添加实时补丁

RT-Linux 的核心是 Linux 内核的一个实时扩展，它为实时任务提供了必要的调度机制和时间管理。通过采用抢占式调度策略，高优先级的实时任务可以打断低优先级的任务，确保实时任务能够及时响应。RT-Linux 对任务的调度和中断处理进行了改进，使得任务能够按照预定的时间要求执行。

与传统的 Linux 内核相比，RT-Linux 在实时性能方面有所提升，但它**并不是一个硬实时系统**，无法保证任务的执行时间绝对精确。对于对时间要求极高的应用，可能需要采用更专门的实时操作系统。

#### 获取纯净内核与补丁

[LubanCat/kernel: LubanCat-RK BSP kernel source (github.com)](https://github.com/LubanCat/kernel/tree/lbc-develop-6.1)

![image-20260328141625172](./assets/SDK仓库地址.png)

``` shell
# 打好补丁的内核
git clone --depth=1 -b lbc-develop-6.1-rt36 https://github.com/LubanCat/kernel.git kernel-6.1

# 纯净内核
git clone --depth=1 -b lbc-develop-6.1 https://github.com/LubanCat/kernel.git kernel-6.1
```

---

查看内核版本：

``` shell
make kernelversion
```

下载实时补丁：[Index of /pub/linux/kernel/projects/rt/6.1/older/](https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.1/older/)

``` shell
# 进入纯净内核目录
cd 野火SDK路径/linux-sdk/kernel-6.1/
# 解压补丁
gzip -d patch-6.1.99-rt36.patch.gz
```

打补丁：

```shell
# 在纯净内核根目录执行（必须加-p1）
patch -p1 < patch-6.1.xx-rt36.patch

--- a/kernel/sched/core.c	2024-01-01 00:00:00
+++ b/kernel/sched/core.c	2024-01-01 00:00:00
```

#### `.rej`文件

查找冲突文件：

``` shell
find . -name "*.rej"
```

冲突文件示例：

``` c
--- drivers/tty/serial/8250/8250.h
+++ drivers/tty/serial/8250/8250.h
// 从176行开始的12行->改为49行
@@ -176,12 +176,49 @@ static inline void serial_dl_write(struct uart_8250_port *up, int value)
 	up->dl_write(up, value);
 }
 
+static inline int serial8250_in_IER(struct uart_8250_port *up)
+{
+	struct uart_port *port = &up->port;
+	unsigned long flags;
+	bool is_console;
+	int ier;
+
+	is_console = uart_console(port);
+
+	if (is_console)
+		printk_cpu_sync_get_irqsave(flags);
+
+	ier = serial_in(up, UART_IER);
+
+	if (is_console)
+		printk_cpu_sync_put_irqrestore(flags);
+
+	return ier;
+}
+
+static inline void serial8250_set_IER(struct uart_8250_port *up, int ier)
+{
+	struct uart_port *port = &up->port;
+	unsigned long flags;
+	bool is_console;
+
+	is_console = uart_console(port);
+
+	if (is_console)
+		printk_cpu_sync_get_irqsave(flags);
+
+	serial_out(up, UART_IER, ier);
+
+	if (is_console)
+		printk_cpu_sync_put_irqrestore(flags);
+}
+
 static inline bool serial8250_set_THRI(struct uart_8250_port *up)
 {
 	if (up->ier & UART_IER_THRI)
 		return false;
 	up->ier |= UART_IER_THRI;
-	serial_out(up, UART_IER, up->ier);
+	serial8250_set_IER(up, up->ier);
 	return true;
 }
 
@@ -190,7 +227,7 @@ static inline bool serial8250_clear_THRI(struct uart_8250_port *up)
 	if (!(up->ier & UART_IER_THRI))
 		return false;
 	up->ier &= ~UART_IER_THRI;
-	serial_out(up, UART_IER, up->ier);
+	serial8250_set_IER(up, up->ier);
 	return true;
 }
 

```



### 支持EtherCAT

#### ighEtherCAT

``` shell
# 进入 ethercat_igh 目录
cd external/ethercat_igh

# 导出编译器路径到环境变量
 export PATH=/home/dev/LubanCat_Linux_SDK/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin:$PATH

 # 初始化构建环境
./bootstrap
./configure --prefix=/home/dev/LubanCat_Linux_SDK/external/ethercat_igh/output --host=aarch64-none-linux-gnu --with-linux-dir=/home/dev/LubanCat_Linux_SDK/kernel-6.1 --enable-8139too=no --enable-stmmac=yes --enable-generic=no --enable-wildcards=yes

# 编译
make -j8

# 将编译生成的内容安装到 output 目录
make install systemdsystemunitdir=/home/dev/LubanCat_Linux_SDK/external/ethercat_igh/output

# 编译内核外部模块
make modules ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- -C /home/dev/LubanCat_Linux_SDK/kernel-6.1 M=$PWD -j3
```

编译生成的用户态文件保存在 ethercat_igh 的 `output` 目录下，生成的 `ko` 模块文件保存在 `master/ec_master.ko` 和`devices/stmmac/ec_stmmac.ko`。

#### 内核优化



## 安装新内核

``` shell
sudo dpkg -i linux-image-6.1.99-rt36-rk3588_*_arm64.deb
sudo dpkg -i linux-headers-6.1.99-rt36-rk3588_*_arm64.deb
```

# 脉塔电机

## EtherCAT

### 加载驱动

这两个模块均来自**IgH EtherCAT Master**（Linux 系统下最主流的开源 EtherCAT 主站实现，广泛用于工业自动化、机器人运动控制等实时控制场景）：

1. `ec_master.ko`：EtherCAT 主站的**核心内核模块**

   是整个 EtherCAT 协议栈的核心载体，负责 EtherCAT 总线拓扑管理、从站设备配置、PDO 过程数据 / SDO 邮箱通信、分布式时钟 DC 同步、数据帧封装与解析、总线状态机管理等核心功能，同时为用户态应用提供标准的控制接口（/dev/EtherCATx）。

2. `ec_stmmac.ko`：EtherCAT 主站针对 stmmac 网卡的**专用实时驱动模块**

   stmmac 是 Synopsys DesignWare 系列以太网 MAC 控制器的标准驱动，广泛用于瑞芯微 RK 系列、STM32MP1、NXP i.MX 等 ARM 嵌入式平台。该模块是为 EtherCAT 深度优化的专用驱动，替代 Linux 标准网络栈，直接对接网卡硬件，为 ec_master 提供低延迟、低抖动的帧收发能力，相比通用驱动能大幅提升 EtherCAT 通信的实时性。

#### 手动加载

<img src="./assets/网卡IP.png" alt="c5d8d1705f6c2ba3eaddff4b8fffaa82" style="zoom:80%;" />

``` shell
# 完好铜柱
sudo insmod ec_master.ko main_devices=fa:fd:53:a0:a5:55

sudo insmod ec_master.ko main_devices=46:ff:1d:47:0f:c7
sudo insmod ec_stmmac.ko
```

#### 自动加载

首先**安装模块到系统内核目录，更新依赖数据库**：

``` shell
# 创建专用的模块存放目录
sudo mkdir -p /lib/modules/$(uname -r)/extra/ethercat/

# 将两个.ko模块文件复制到系统内核模块目录
sudo cp ec_master.ko ec_stmmac.ko /lib/modules/$(uname -r)/extra/ethercat/

# 更新内核模块依赖数据库，让系统识别模块和依赖关系
sudo depmod -a
```

---

然后配置模块固定启动参数

**创建模块参数配置文件**，让系统每次加载 ec_master 时，自动带上指定的 MAC 地址参数：

``` shell
# 创建并编辑参数配置文件
sudo nano /etc/modprobe.d/ethercat.conf
```

在文件中写入以下内容，保存退出：

``` shell
options ec_master main_devices=fa:fd:53:a0:a5:55
```

---

其次配置开机自动加载，固定加载顺序

**创建 systemd 开机加载配置文件，按正确顺序**指定要自动加载的模块：

``` shell
# 创建并编辑开机加载配置文件
sudo nano /etc/modules-load.d/ethercat.conf
```

在文件中按顺序写入模块名，保存退出：

``` shell
ec_master
ec_stmmac
```

---

验证配置有效性：

``` shell
# 手动触发systemd模块加载服务，测试配置是否正常
sudo systemctl restart systemd-modules-load.service

# 验证模块是否成功加载
lsmod | grep ec_

# 查看模块加载日志，确认无报错
dmesg | grep EtherCAT
```

验证无报错后，重启系统，再次执行`lsmod | grep ec_`，即可确认模块已开机自动加载。

### 设备权限管理

![img](./assets/ethercat权限错误.png)

Linux 的 `/dev` 目录是**临时文件系统（tmpfs）**，**关机 / 重启后会清空**，所有设备节点（包括 `/dev/ethercat0`）会重新生成。

存放系统中所有的**设备文件**，全称是 Device Nodes，/dev 是用来传递**真实业务数据流**和发送**复杂控制指令（ioctl）**的地方。

**驱动视角**：在 Linux 驱动中，硬件通常被抽象为三种：字符设备（按字节流访问，如串口、鼠标）、块设备（按数据块访问，如硬盘、U盘）和网络设备（不通过 /dev，走 Socket）。
**当驱动程序向内核注册一个字符或块设备时，系统会在 /dev 下创建一个对应的文件**（例如 /dev/ttyS0 代表串口，/dev/sda 代表硬盘），应用程序通过标准的 C 语言文件 I/O 函数（open、read、write、ioctl）来操作这些文件。

---

查看命令权限：

``` shell
ls -l $(which ethercat)
```

修改权限：

``` shell
sudo nano /etc/udev/rules.d/99-ethercat.rules
# 输入：
SUBSYSTEM=="ethercat", GROUP="users", MODE="0666"

# 重载udev规则
sudo udevadm control --reload-rules
# 触发规则生效
sudo udevadm trigger
```

### 实现EtherCAT主站

#### IGH

 IgH EtherCAT Master，是一个**内核态**的开源 EtherCAT 主站栈，专门为 Linux 设计，是工业领域最成熟的硬实时 EtherCAT 主站实现之一。

**运行层级**：运行在 Linux 内核空间，**直接接管网卡驱动**， bypass 操作系统网络协议栈，延迟极低、抖动极小，适合硬实时运动控制（比如机器人、伺服电机）。

**部署要求**：需要编译安装内核模块（`ec_master`），并将专用网卡绑定到 EtherCAT 主站。

**性能**：实时性极强，抖动可控制在微秒级，是工业级场景的首选。

安装`ecrt.h`：

``` shell
# 安装依赖
sudo apt update && sudo apt install linux-headers-$(uname -r) build-essential automake autoconf libtool

# 下载源码（替换为最新版本）
git clone https://gitlab.com/etherlab.org/ethercat.git
cd ethercat
./bootstrap
./configure --prefix=/opt/etherlab --enable-generic --enable-8139too
make -j$(nproc)
sudo make install
```

安装后，头文件默认路径为 `/opt/etherlab/include`，库文件为 `/opt/etherlab/lib`。

``` shell
ethercat slaves -v
```

## 电机

### 代码框架

#### 架构

``` c
                ┌─────────────────────────────────────────────────────────────────────┐
                │                    用户层 / 业务层（非实时）                         │
                │  RobotInterface / motors_test                                       │
                │  - initial_and_start_motors()                                       │
                │  - apply_action()/send_command()                                    │
                │  - get_joint_q/get_status()                                         │
                └──────────────────────────────┬──────────────────────────────────────┘
                                               │
                                               │ ThreadSafeQueue<ControlCommand>
                                               ▼
                ┌─────────────────────────────────────────────────────────────────────┐
                │               MYACTUA 控制线程（实时，SCHED_FIFO，1ms）              │
                │  while (running_) {                                                  │
                │    process_commands();        // 处理异步命令队列                     │
                │    update();                  // 读通信状态+收PDO+状态机+写PDO         │
                │    update_status_snapshot();  // 发布快照（mutex保护）                │
                │    status_callback_(...);     // 可选回调                             │
                │    clock_nanosleep(ABSTIME);  // 1ms绝对时间周期                     │
                │  }                                                                    │
                └──────────────────────────────┬──────────────────────────────────────┘
                                               │ 通过 adapter->receive/send 访问PDO镜像
                                               ▼
                ┌─────────────────────────────────────────────────────────────────────┐
                │           EthercatAdapterIGH 线程（实时，SCHED_FIFO，1ms）           │
                │  rt_loop() {                                                         │
                │    ecrt_master_receive();                                            │
                │    ecrt_domain_process();                                            │
                │    更新 slave_configured = online && operational                     │
                │    DC时钟同步                                                        │
                │    ecrt_domain_queue();                                              │
                │    ecrt_master_send();                                               │
                │  }                                                                    │
                └──────────────────────────────┬──────────────────────────────────────┘
                                               ▼
                                         EtherCAT 总线 + 从站驱动器

```

| 改进点     | 实现方式                                  | 效果                     |
| ---------- | ----------------------------------------- | ------------------------ |
| 实时性保证 | 独立线程 + SCHED_FIFO 调度 + 绝对时间睡眠 | 控制周期抖动 < 100μs     |
| 解耦设计   | 生产者 - 消费者模式，线程安全队列         | 用户逻辑与控制完全分离   |
| 异步指令   | send_command () 非阻塞调用                | 用户无需关心控制周期     |
| 状态发布   | 状态快照 + 回调机制                       | 支持监控、日志等扩展功能 |

#### 数据流图

``` c
用户线程                           实时控制线程
   │                                    │
   │  send_command()                    │
   ├──────────────────────────────────► │ process_commands()
   │     ThreadSafeQueue                │      │
   │                                    │      ▼
   │                                    │  update() ──► EtherCAT
   │                                    │      │
   │                                    │      ▼
   │  get_status()                      │ update_status_snapshot()
   │ ◄──────────────────────────────────┤      │
   │     status_snapshot_               │      │
   │     (mutex protected)              │      ▼
   │                                    │  status_callback_()
   │                                    │
```

#### 执行流

``` c
用户线程调用：controller.start()
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ MYACTUA::start()                                            │
│ 1) if (running_) return;                                    │
│ 2) running_ = true;                                         │
│ 3) rt_thread_ = std::thread(&MYACTUA::rt_thread_func, this) │
│ 4) pthread_setschedparam(rt_thread_, SCHED_FIFO, prio=80)   │
│ 5) 打印“实时控制线程已启动”                                    │
└────────────────────────────────────────────────────────────┘
        │
        ├────────────► start() 立即返回（用户线程继续）
        │
        ▼
┌──────────────────────── 后台实时控制线程 ────────────────────────┐
│ rt_thread_func()                                                 │
│ - clock_gettime() 初始化 next_period                              │
│ - period_ns = 1,000,000 (1ms)                                    │
│ - while (running_) {                                              │
│     a) process_commands()                                         │
│        - 从 ThreadSafeQueue 取命令                                │
│        - SET_SETPOINTS: 更新 setpoint                             │
│        - STOP/RESTART/SET_MODE: 入离散命令队列                    │
│                                                                   │
│     b) update({})                                                 │
│        - 每电机：isConfigured() 判通信                             │
│        - 离线：offline计数+1，跳过收发                             │
│        - 在线：receive(PDO) -> 状态机 -> send(PDO)                │
│        - service_discrete_commands() 做离散命令重试/验收           │
│                                                                   │
│     c) update_status_snapshot()                                   │
│        - 加锁写 status_snapshot_                                   │
│                                                                   │
│     d) 可选 status_callback_(status_snapshot_)                    │
│                                                                   │
│     e) 绝对时间睡眠到下一个 1ms 周期                               │
│        next_period += 1ms; clock_nanosleep(TIMER_ABSTIME, ...)    │
│   }                                                               │
└───────────────────────────────────────────────────────────────────┘
        │
        ▼
用户线程任意时刻可调用：
- send_command(...) -> 入队（异步）
- get_status()/get_joint_*() -> 读快照（互斥锁保护）
- shutdown() -> running_=false + join，等待实时线程退出

```



### 线程安全队列

支持阻塞/非阻塞/超时三种模式，无锁设计不适用（指令需要可靠传递）

``` c++
#pragma once

#include <queue>
#include <mutex>
#include <condition_variable>
#include <chrono>

namespace myactua {

/* 线程安全队列 */
template<typename T>
class ThreadSafeQueue {
public:
    ThreadSafeQueue() = default;
    ~ThreadSafeQueue() = default;

    void push(const T& value) {
        std::lock_guard<std::mutex> lock(mutex_);
        queue_.push(value);
        cond_.notify_one();
    }

    void push(T&& value) {
        std::lock_guard<std::mutex> lock(mutex_);
        queue_.push(std::move(value));
        cond_.notify_one();
    }

    bool pop(T& value, int timeout_ms = -1) {
        std::unique_lock<std::mutex> lock(mutex_);
        if (timeout_ms > 0) {
            if (!cond_.wait_for(lock, std::chrono::milliseconds(timeout_ms),
                               [this] { return !queue_.empty(); })) {
                return false;
            }
        } else if (timeout_ms == 0) {
            if (queue_.empty()) {
                return false;
            }
        } else {
            cond_.wait(lock, [this] { return !queue_.empty(); });
        }
        value = std::move(queue_.front());
        queue_.pop();
        return true;
    }

    bool empty() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return queue_.empty();
    }

    size_t size() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return queue_.size();
    }

    void clear() {
        std::lock_guard<std::mutex> lock(mutex_);
        std::queue<T> empty;
        std::swap(queue_, empty);
    }

private:
    mutable std::queue<T> queue_;
    mutable std::mutex mutex_;
    std::condition_variable cond_;
};

} // namespace myactua

```

### 离散队列命令



### 运行模式切换

``` c++
void MYACTUA::handle_mode_switching(MotorState& motor)
{
    uint16_t sw = motor.rx.status_word;

    switch (motor.mode_switch_step)
    {
        case ModeSwitchStep::IDLE:
            motor.mode_switch_step = ModeSwitchStep::SET_MODE_CLEAR_DISABLE;
            break;

        case ModeSwitchStep::SET_MODE_CLEAR_DISABLE:
            motor.tx.op_mode = motor.target_mode;
            motor.tx.target_pos = motor.rx.pos;
            motor.tx.target_vel = 0;
            motor.tx.target_torque = 0;
            motor.tx.control_word = CMD_SHUTDOWN;
            motor.mode_switch_step = ModeSwitchStep::ENABLE;
            break;

        case ModeSwitchStep::ENABLE:
            if (!is_switched_on(sw) && !is_operation_enabled(sw)) {
                if (is_ready_to_switch_on(sw)) {
                    motor.tx.control_word = CMD_SWITCH_ON;
                    motor.mode_switch_step = ModeSwitchStep::OPERATING;
                } else {
                    motor.tx.control_word = CMD_SHUTDOWN;
                }
            } else
                motor.tx.control_word = CMD_SHUTDOWN;
            break;

        case ModeSwitchStep::OPERATING:
            if(is_switched_on(sw)){
                if (is_operation_enabled(sw)) {
                    motor.mode_switch_step = ModeSwitchStep::DONE;
                } else {
                    motor.tx.control_word = CMD_ENABLE_OPERATION;
                }
            }else
                motor.tx.control_word = CMD_SWITCH_ON;
            break;

        case ModeSwitchStep::DONE:
            if (motor.rx.op_mode == motor.target_mode) {
                motor.mode_switch_step = ModeSwitchStep::IDLE;
                motor.step = MotorStep::RUNNING;
            } else {
                motor.mode_switch_step = ModeSwitchStep::SET_MODE_CLEAR_DISABLE;
            }
            break;
    }
}
```

### 避免通信波动

> 负责 EtherCAT 多电机实时控制链路设计，针对总线抖动与瞬时失联，将离散指令设计为**事务化状态机**、连续指令设计为 **freshest-wins** 实时流，落地重试确认、**超时降级**与平滑恢复机制。

实时线程是 1kHz 周期，先处理上层命令，再执行一次 EtherCAT 周期，再更新状态快照，motor_control.cpp (line 74)。我会把它解释成：

- 控制面和数据面分离。
- 上层线程只投递命令，不直接碰实时 PDO。
- 实时线程统一仲裁命令、状态机和通信状态，避免多线程直接打控制字。

这点很加分，因为它体现了实时系统里“单写者模型”的意识。

**离散指令**
STOP / RESTART / SET_MODE 被建模成离散命令。每个电机各自有一个离散命令队列，并且命令不是“发出即完成”，而是经过：

`QUEUED -> APPLY_PENDING -> VERIFYING -> DONE / FAILED`。

> 对离散指令，我把它当成一个小事务。通信抖动时，不能因为主站发过一次就认为动作已经完成，而是要以从站状态字和模式反馈为准做闭环确认。

然后补上这几个硬指标，很像资深工程师：

- 掉线时不推进命令阶段，只冻结等待，motor_control.cpp (line 439)
- 每 10 个周期重试一次，motor_control.cpp (line 22)，最多重试 100 次
- 总超时 5 秒
- 必须连续 3 个周期满足判据才算成功，防止抖动时“误判成功”，motor_control.cpp (line 472)

> 离散命令的关键不是低延迟，而是抗抖动条件下的最终一致性。

---

**连续指令**
SET_SETPOINTS 在这套代码里**没有走事务状态机**，而是**只更新 setpoint 缓存**，motor_control.cpp (line 348)。上层 `apply_action()` 只是做限位和单位转换，再异步下发连续目标，robot_interface.cpp (line 245)。

然后在实时线程里，**每周期拿“当前最新 setpoint”生成目标值**；如果某个电机这一拍通信不正常，就直接跳过，不接收、不发送，也不覆写控制状态，motor_control.cpp (line 126)。

你可以把它总结成一句非常像专家的话：

> 连续指令不做补发语义，只做最新值语义。因为过期的速度、位置、力矩命令即使最终送达，也可能已经不再安全。

---

**为什么不能统一都做重试？**

> 因为两类命令的失效模型不一样。
> 离散命令是状态迁移，比如停机、重启、切模式，重试是合理的，因为目标状态在未来仍然有效。
> 连续命令是时序敏感量，比如某一时刻的速度或力矩，过了时效窗口再送达，物理意义就变了，盲目重传反而可能造成控制突变。

这段很关键，很多人答不到这个层次。

---

**进一步工程化时，会怎么增强**
这里要明确：下面是我基于当前代码的“工程升级思路”，不是说现在已经全做了。

1. 对连续指令加`freshness watchdog`

   如果最近 N 个周期没收到新的上层目标，或者总线连续异常超过阈值，就不要一直无限期保持旧值。

   更合理的是分层降级：

   - 短时抖动：保持上一拍命令
   - 中等时长：做斜坡减速或力矩衰减到 0
   - 长时失联：切 STOP 或 Quick Stop
   - 恢复后：不要瞬间恢复到最新大步长命令，而是重新对齐状态后再平滑接管

2. 对离散指令加优先级和抢占
   现在的队列设计已经不错，但如果工程再往前走，我会把 STOP / E-STOP 设成最高优先级，可以打断普通 SET_MODE。
   因为安全动作不应该排队等。

3. 对通信健康度做更细监控

   当前主要使用`isConfigured()` 判断链路是否正常。如果做量产级方案，还会加：

   - 周期级丢包统计
   - 连续丢包长度
   - 恢复次数
   - 状态字异常频率
   - 模式切换失败率
   - 控制环 jitter 统计

这样不只是“知道掉线了”，而是能知道“掉线模式是什么”。

---

**一段 1 分钟回答稿**
你可以直接说这段：

> 我处理通信波动时，首先会把命令分成离散指令和连续指令，因为两者语义完全不同。
> 对离散指令，比如停机、重启、模式切换，我按事务处理，核心是状态机、重试、超时和设备侧闭环确认。发出去不算成功，必须连续多个周期读到目标状态才算完成。
> 对连续指令，比如位置、速度、力矩目标，我按实时流处理，核心是最新值优先、过期值丢弃、短时保持、超时降级。因为连续命令关心的是时效性，不是必达性，盲目补发历史命令可能更危险。
> 这样做的结果是，系统在通信抖动时不会出现不可预测行为，而是能保持安全、平滑，并且恢复逻辑清晰。

**一段 3 分钟深挖版**
如果对方愿意继续听，你再补：

> 在实现上，我倾向于把上层命令和实时控制线程解耦。上层只投递命令，实时线程统一做仲裁，避免多线程直接操作控制字。
> 离散指令我会做 per-motor command queue，并给每条命令建模生命周期，比如 queued、apply、verifying、done、failed。遇到短时通信抖动时，不推进状态机；链路恢复后继续验证。这样能防止因为瞬时波动把一个本来应该成功的模式切换误判成失败。
> 连续指令我不会设计成 ACK 型协议，而是设计成 freshest-wins。因为控制目标是一个流，真正重要的是当前时刻的控制意图，而不是历史每一帧都必须执行。短抖动可以 hold，上层如果持续断流，则逐步衰减到安全状态，超过阈值再进入 stop 或 quick stop。
> 本质上，这是把“可靠性”和“实时性”拆开处理：离散指令要可靠，连续指令要实时。

**一个很加分的收尾**
最后你可以补一句：

> 我一般不把通信波动简单理解成“网络不好”，而是把它看成控制系统的一部分设计输入。真正好的方案不是尽量把包发出去，而是让系统在包丢了、延迟了、乱序了的时候，仍然表现出可解释的物理行为。



## 调试记录

### 模式匹配

``` c++
// 检查与驱动器当前的模式(6061h)是否与用户设定的目标模式一致
if (motor.rx.op_mode != motor.target_mode) 
{   
    motor.step = MotorStep::MODE_SWITCHING; // 标记状态为切换中
    motor.tx.control_word = CMD_SHUTDOWN;   // 先失能
    motor.tx.op_mode = motor.target_mode;
    motor.tx.target_pos = motor.rx.pos;     // 将当前实际位置设为目标位置
    motor.tx.target_vel = 0;                // 速度清零
    motor.tx.target_torque = 0;             // 力矩清零
    return;
}
```

电机上电后`6061h`中的模式并非和三种模式一致，导致直接进入错误判断。

---

### 串行走线

ethercat走线不能并行走线，只能串行走线，ethercat依据物理连接顺序确定ID；电机供电线断裂。

### 通信波动

如果`STOP`状态一直保持，应停在`STEP=STOPPED, MODE_SWITCH_STEP=IDLE`。 这次“小概率异常”主要是因为`STOP`不是锁存态，会被运行期状态覆盖。

1.`motors_test`只做了初始化，然后监控，没有再发控制命令 ；初始化里是“启动线程后，异步入队一次STOP” ；`STOP`执行时会把每个电机设成`STOPPED/IDLE`。

2.但在每个控制周期里，只要`isConfigured==false`，代码会直接把`step`改成`IDLE`：

 ```c++
   /* 设置电机目标值 */
   for (size_t i = 0; i < _motors.size(); i++)
   {
       if (!_adapter->isConfigured(_motors[i].slave_index)) {
           _motors[i].step = MotorStep::IDLE;
           continue;
       }
       double val = (i < setvalues.size()) ? setvalues[i] : _motors[i].setpoint;
       process_single_motor(_motors[i], val);
   }
 ```

3.`isConfigured`来自`online && operational`的1ms刷新，存在瞬时抖动可能一旦`STOPPED`被覆盖成`IDLE`，后续重新`configured`时就会重新进入使能/运行/模式切换流程，不再保证停在`STOPPED/IDLE`。

所以现象本质上是：运行期从站状态瞬时波动 + 当前状态机覆盖策略，导致看到“偶发不在STOPPED/IDLE”。

---

| 方案                                                    | 改动范围              | 效果（保持STOPPED/IDLE） | 风险/代价                       | 复杂度 |
| :------------------------------------------------------ | :-------------------- | :----------------------- | :------------------------------ | :----- |
| STOP锁存（推荐）                                        | MYACTUA状态机         | 高，最直接               | 需定义“谁来解锁”（通常RESTART） | 中     |
| isConfigured去抖（如连续N周期才判离线）                 | EtherCAT适配层        | 中高，能降“小概率抖动”   | 离线检测变慢一点                | 中     |
| 控制状态与通信状态解耦（掉线不改step，单独标comm_lost） | MYACTUA快照与打印逻辑 | 高，语义最清晰           | 需要调整监控显示/诊断习惯       | 中高   |
| 上层motors_test周期性重发STOP心跳                       | 仅测试程序            | 中，见效快               | 治标不治本，命令冗余            | 低     |
| 只优化实时环境（调度、CPU隔离、IRQ绑定）                | 部署层                | 中，减少触发概率         | 运维成本高，不保证根治          | 中高   |

---

**方案一：**通信波动通常不会导致“队列层面”的命令丢失，但会导致**执行层面的“等价丢失”**（尤其是轨迹点）。

- 命令入队不会丢：`send_command -> ThreadSafeQueue::push` 是互斥队列，无主动丢弃逻辑。
- 命令处理不依赖通信状态：实时线程每周期先`process_commands()`，把队列清空。
- 但通信断时不下发PDO：`comm_ok=false`就跳过`process_single_motor`和`send`。

按命令类型评估：

- `STOP/RESTART/SET_MODE`：大多不会“永久丢失”，因为它们写入的是持久状态（`step/target_mode`），通信恢复后会继续生效。
- `SET_SETPOINTS`：是“最后值覆盖”语义，断链期间的中间轨迹点会被后续值覆盖，恢复后只会执行最新 `setpoint`。

高风险边界：

- 若通信波动期间你执行`deinit`，`STOP`虽入队，但50ms后线程就停，可能来不及在链路恢复后发到从站。

当前设计是**state-based / last-write-wins**，不是“每条命令必达且可确认”的transaction模型。对状态命令较稳，对高频轨迹命令会有时序丢点。

**方案二**：`comm_ok`可以记录每个电机从站是否在线，如果此轮`update({})`中发现为离线状态，则进行不下发这`process_commands()`中的指令，在下轮周期再进行发送（**store-and-forward 重试**、语义上接近 **at-least-once（尽力不丢）**）。

**优点**

- 能吸收短时链路抖动，避免“一次离线就把这周期命令丢掉”。
- 如果按FIFO延迟发送，可保持命令顺序。
- 主站侧可实现，不一定要立刻改从站协议。

**缺点**

- 只能保证“最终发送”，不能保证“最终执行”，没有ACK就无法闭环确认。
- 离线恢复后会积压回放，带来时延抖动和突发下发，并且长时间离线会导致队列增长，需要上限与丢弃策略。
- 可能执行“过期命令”（比如轨迹点、旧姿态目标），有安全风险。
- 若是全局门控，一个从站掉线会阻塞全部电机（头阻塞）。

### SAFEOP错误

`ethercat slave`可以扫到从站，但是无法进入`OP(online && operational)`状态。

查看内核侧提示：

``` shell
dmesg -T | grep -Ei "EtherCAT|SAFEOP|watchdog|AL"
```

最常见是两类：

- Sync manager watchdog（经常表现为卡 SAFEOP）
- PDO/SM 配置不匹配（第二个从站对象或映射与第一个不完全一致）

![image-20260416101505233](./assets/image-20260416101505233.png)

---

排查方法：

1. 用实时调度运行程序，先排除主站周期抖动：

   ````shell
   sudo chrt -f 90 ./stop_read_status
   
   sudo ./stop_read_status
   ````

2. 固定网卡参数并关 EEE：
   ```shell
   sudo ethtool --set-eee enp8s0 eee off
   sudo ethtool -s enp8s0 speed 100 duplex full autoneg off
   ```

3. 再看状态是否仍报 0x001A：
   ``` shell
   sudo ethercat slaves -v
   dmesg -T | grep -Ei "EtherCAT|AL status|SAFEOP|Synchronization"
   ```

4. 做 A/B 交换测试（不改代码）：交换两台从站物理位置/网线，看错误是“跟设备走”还是“跟位置走”。

现在这组现象里，PDO 已匹配 + 0x001A，最可能就是 **DC 同步时序/实时性/物理链路质量** 这条线。

---

解决方法：

1. 推荐做法：用 `systemd` 固定实时调度

   service里设置：

   - CPUSchedulingPolicy = fifo
   - CPUSchedulingPriority = 90
   - LimitRTPRIO = 95

这样以后只要 systemctl start xxx，就自动是实时优先级。

   2.轻量做法：给可执行文件 `CAP_SYS_NICE`

```shell
sudo setcap cap_sys_nice+ep /home/cat/Myactua_Ethercat/src/inference/build/motors_test 
```

之后直接运行：

```shell
/home/cat/Myactua_Ethercat/src/inference/build/motors_test 
```

你代码里本来就会 pthread_setschedparam，有这个能力后通常就不需要 chrt 了。
注意：每次重新编译后，setcap 需要重新执行一次。

### 数据并发错误

**问题闭环总结（嵌入式实时控制视角）**

**问题定义与影响**

1. 在 EtherCAT 实时控制中，应用层已计算出正确控制字（`0x07/0x0F`），但电机状态随机无法进入 `SW_ON/OP_EN`。

2. 该问题表现为“偶发成功、偶发失败”，对启停一致性和安全性影响大，属于典型实时并发缺陷。

**现场现象与关键证据**

​	1.诊断显示 `wc_state=COMPLETE`，说明链路层基本健康，不是主因。

​	2.关键矛盾是“应用想发”和“总线实际待发”不一致：
`send_cw=0x000F`，但 `pd_pre_queue=0x0007`，状态停在 `0x1233`（未进入 OP_EN）。

​	3.通过 `--tx-phase-us` 相位扫描后，结果对延时敏感，进一步指向时序竞争而非固定逻辑错误。  

**分层排障思路**

​	1.第一层（通信层）排除：WKC 多数完整，非典型掉线问题。

​	2.第二层（状态机层）排除：控制字策略正确，但状态推进不稳定。

​	3.第三层（并发时序层）定位：应用线程与 EtherCAT 线程同时访问 `domain1_pd`，存在竞态窗口。

**根因建模**

7. 原结构中，`send()` 在应用线程直接写 `domain1_pd`。

8. 同时 EtherCAT 线程在 `receive/process/queue/send` 周期内也读写同一域内存。

9. 在特定相位下，应用写入会错过“有效发送窗口”或被后续周期内容覆盖，导致“写了但本周期没发出去”。

10. **修复策略（架构级）**

11. 采用“单写者原则”：应用线程不再直接写域内存。

12. 引入 `tx_shadow` 作为线程间缓冲，应用线程仅写缓冲。

13. 仅 EtherCAT 线程在 `ecrt_domain_process()` 后、`ecrt_domain_queue()` 前，将 `tx_shadow` 统一落盘到 `domain1_pd`。

14. 该策略把“控制决策时机”和“总线发送时机”解耦，消除跨线程竞态。

15. **实现要点**

16. 缓冲与同步新增于 [EthercatAdapterIGH.hpp](/home/cat/Myactua_Ethercat/src/motors/src/protocol/ethercat/EthercatAdapterIGH.hpp)。

17. 周期内统一落盘与发送路径在 [EthercatAdapterIGH.cpp](/home/cat/Myactua_Ethercat/src/motors/src/protocol/ethercat/EthercatAdapterIGH.cpp)。

18. `send()` 改为仅写 `tx_shadow`，不再直接写 `domain1_pd`。

19. **验证与验收标准**

20. 编译通过并可运行：`stop_read_status`。

21. 验收核心指标从“随机”转为“确定性”：
       `send_cw == pd_pre_queue` 应稳定成立。

22. 状态机推进应稳定复现：
       `0x07 -> SW_ON`，`0x0F -> OP_EN`。

23. 相位扫描下成功率不再对微小延时高度敏感，说明竞态被消除。

24. **工程化结论**

25. 此次缺陷本质是“实时系统中共享过程映像的多线程写冲突”。

26. 解决关键不在微调控制字，而在重构写入责任边界（Single Writer + Shadow Buffer）。

27. 该修复具备可迁移性，可作为后续 EtherCAT/现场总线驱动并发访问的标准范式。

# IMU

查看是否连接：

``` shell
dmesg | tail -n 20
```

# 推理

如果你要把这套框架重写成适配你自己的人形机器人，最稳妥的顺序不是从策略推理开始，而是从“硬件抽象和坐标定义”开始。因为推理层只是吃 observation、吐 action，真正最难、最容易埋坑的是底层机器人接口。

**推荐构建顺序**

先定义机器人抽象，不写推理

先把你机器人的“最小控制闭环”想清楚：

- 一共多少关节
- 每个关节的控制模式是什么
- 电机 ID、总线、IMU、编码器分别怎么连
- 你的统一关节顺序是什么

先固定两套顺序：

- 硬件顺序
- 算法顺序

这一步最重要。后面所有 obs/action、零位、限位、符号、模型输入输出都依赖它。

## 配置文件

### `robot.yaml`

先重写 robot.yaml 对应的配置模型

你可以参考现在的三层拆法：

- imu
- motors
- robot

但建议一开始就按你的机器人重新设计字段，不要机械照搬。至少要能描述：

- 传感器连接方式
- 所有关节/电机拓扑
- 零位偏置
- 正负方向
- 关节限位
- PD 参数
- 是否有闭链/并联机构

目标是：配置文件能完整描述你的机器人硬件和控制参数，而不是代码里写死。

## RobotInterface

实现你自己的 RobotInterface

这是第一优先级，先不用碰 ONNX。

你需要先做到这些基础能力：

- init_motors()
- deinit_motors()
- get_joint_q()
- get_joint_vel()
- get_joint_tau()
- get_quat()
- get_ang_vel()
- apply_action()
- reset_joints()

先让它能跑一个最简单的测试程序：

- 初始化硬件
- 读取关节和 IMU
- 发一个固定站立姿态
- 机器人能稳定站住或进入安全姿态

只要这一步不稳，后面接策略一定会出问题。

把“关节定义”彻底校准

这是上线前必须单独完成的一步：

- 每个关节零位是否正确
- 正方向是否正确
- 限位是否合理
- 左右腿/左右臂是否镜像一致
- IMU 朝向和机体坐标系是否一致

建议做一个单独校准脚本或小节点，逐关节验证：

- 给正的小角度命令
- 看反馈方向是否符合预期
- 检查 motor_sign、zero_offset、joint_limits

这一层没校准好，模型再好也没法用。

如果有闭链/并联机构，单独实现映射层

如果你的机器人像这里一样有踝关节闭链，那就要尽早把这层单独抽出来，像现在的`Decouple`一样。

顺序建议是：

- 先只做“读状态映射”
- 再做“写命令映射”
- 最后再把 q/vel/tau 三者统一起来

如果你的机器人没有闭链，这层可以直接省掉，先用串联关节版本跑通。

做一个“无模型”的控制节点

在接强化学习策略前，先写一个简化版节点，只做：

- 读 IMU
- 读关节
- 发布 joint_states / imu
- 接收一个固定姿态或外部 joint target
- 调用 RobotInterface::apply_action()

目标是验证：

- 线程模型
- ROS 通信
- 实时性
- 电机控制链路
- 安全停机逻辑

这一步其实就是把现在的 InferenceNode 删掉推理部分，只保留机器人控制骨架。

## 推理层

定义 observation 和 action 语义

现在才开始碰策略接口。

你要明确：

- 模型输出的是“目标关节角”还是“增量”还是“力矩”
- observation 里有哪些量
- 各量的顺序、缩放、裁剪方式
- 是否需要 last_action
- 是否需要 frame_stack
- 是否需要外部命令速度
- 是否需要感知输入

建议先把 observation/action 的定义写成文档或注释表，再写代码。
因为这一步其实是在定义“训练侧”和“部署侧”的契约。

重写 InferenceNode 的 obs 拼装逻辑

在硬件和关节定义稳定后，再实现：

- load_config()
- setup_model()
- inference()
- control()
- apply_action()

建议按最小版本开始：

- 单模型
- 单帧 obs
- 无 interrupt
- 无 attn_enc
- 无 beyondmimic

先让最普通的 locomotion policy 跑通，再逐渐加高级分支。

先跑“离线对齐”，再上真机

在真机跑之前，先验证：

- 部署端 obs 顺序是否和训练端一致
- 单位是否一致：rad / deg，机体系 / 世界系
- action 反归一化是否一致
- 默认姿态偏置是否一致
- 输出关节顺序是否一致

这一层如果能拿训练日志或仿真数据做 replay 对齐，会省很多时间。

## 模式扩展

基础 locomotion 跑稳之后，再考虑：

- frame_stack
- interrupt
- attn_enc
- beyondmimic
- motion prior
- Python binding

这些都应该是第二阶段，不要一开始全做。
