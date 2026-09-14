# RK3588 + PREEMPT_RT + EtherCAT 启动死锁：一次从 SSH 失联到内核锁死的完整排查记录

> 平台：野火 LubanCat-5-BTB / RK3588  
> 系统：Debian 12，Linux 6.1.99-rt36-rk3588，PREEMPT_RT  
> 关键组件：IgH EtherCAT Master、定制 `ec_stmmac`、NetworkManager、systemd、U-Boot、eMMC  
> 最终根因：NetworkManager 在冷启动时自动 bring-up EtherCAT 专用网口 `eth0`，进入 `ec_stmmac` 的 `stmmac_open()` 路径后触发 RTNL / RT-mutex 锁问题，随后演化为 `scheduling while atomic` 和 RCU stall，导致整机逐步失去响应。

这次问题最值得保留的不是某一条修复命令，而是完整的分析路径：

```text
SSH timeout
→ 判断是否是网络层
→ 串口确认系统启动状态
→ 从大量 kernel log 中找到“第一条破坏性异常”
→ 顺着 call trace 定位到 NetworkManager / ec_stmmac
→ U-Boot + initramfs 无损救援
→ A/B 验证
→ sysfs 映射物理网口
→ 将 EtherCAT NIC 永久设为 NetworkManager unmanaged
```

---

## 1. 从 SSH 失联开始：先判断故障在哪一层

前一天在开发板上正常执行：

```bash
sudo poweroff
```

系统关机后再物理断电。第二天重新上电后，VS Code Remote-SSH 无法连接：

```text
ssh: connect to host 192.168.5.237 port 22: Connection timed out
```

Windows 上执行：

```cmd
ping 192.168.5.237
```

原始结果：

```text
正在 Ping 192.168.5.237 具有 32 字节的数据:
来自 192.168.5.179 的回复: 无法访问目标主机。
来自 192.168.5.179 的回复: 无法访问目标主机。
来自 192.168.5.179 的回复: 无法访问目标主机。
来自 192.168.5.179 的回复: 无法访问目标主机。
```

随后：

```cmd
arp -a
```

在 `192.168.5.179` 对应的局域网接口下没有发现：

```text
192.168.5.237
```

### 如何分析

这里第一反应不能是“检查 sshd”或“重装 VS Code Remote-SSH”。

因为 SSH 的数据路径至少是：

```text
SSH
↓
TCP/22
↓
IP
↓
ARP / 邻居发现
↓
Ethernet / Wi-Fi
```

当本机连 `192.168.5.237` 的 MAC 地址都解析不到时，TCP 22 根本还没有机会建立。因此：

```text
SSH timeout 是症状，
故障一定发生在 SSH 之前。
```

ARP 表中当时存在：

```text
192.168.5.239    6c-1f-f7-8c-09-b4
```

而 `.239` 可以 ping，于是尝试：

```cmd
ssh cat@192.168.5.239
```

得到：

```text
cat@192.168.5.239's password:
Permission denied, please try again.
...
Permission denied (publickey,password).
```

这一步说明 `.239` 是一台在线且开放 SSH 的设备，但**不能说明它就是开发板**。后面系统恢复后确认，真正的 `192.168.5.237` 是开发板的 `wlan0` 地址，`.239` 只是同网段另一台设备。

这里的经验是：**ping 通只能证明“某台设备在线”，不能证明“它就是目标设备”。**

---

## 2. 串口日志：从“看起来卡在 PCIe”追到真正的内核死锁

开发板 HDMI 没有任何输出，SYS LED 先慢速双闪，之后常亮。于是接入 Debug UART。

最初看到的原始串口信息是：

```text
Debian GNU/Linux 12 lubancat ttyFIQ0

[username:password] root:root cat:temppwd

Modify information : /etc/issue

lubancat login: [    7.345331] rk_pcie_establish_link: 371 callbacks suppressed
[    7.345347] rk-pcie fe170000.pcie: PCIe Linking... LTSSM is 0x3
[    7.365555] rk-pcie fe170000.pcie: PCIe Linking... LTSSM is 0x3
[    7.386605] rk-pcie fe170000.pcie: PCIe Linking... LTSSM is 0x3
[    7.407644] rk-pcie fe170000.pcie: PCIe Linking... LTSSM is 0x3
[    7.428699] rk-pcie fe170000.pcie: PCIe Linking... LTSSM is 0x3
[    7.449730] rk-pcie fe170000.pcie: PCIe Linking... LTSSM is 0x3
[    7.470776] rk-pcie fe170000.pcie: PCIe Linking... LTSSM is 0x3
[    7.491818] rk-pcie fe170000.pcie: PCIe Linking... LTSSM is 0x3
[    7.512863] rk-pcie fe170000.pcie: PCIe Linking... LTSSM is 0x3
[    7.533899] rk-pcie fe170000.pcie: PCIe Linking... LTSSM is 0x3
[    7.744232] rk-pcie fe170000.pcie: PCIe Link Fail, LTSSM is 0x3, hw_retries=1
[    8.772430] rk-pcie fe170000.pcie: failed to initialize host
[   15.109993] platform mtd_vendor_storage: deferred probe pending
```

### 如何分析这段信息

最重要的不是：

```text
PCIe Link Fail
```

而是更前面的：

```text
Debian GNU/Linux 12 lubancat ttyFIQ0
lubancat login:
```

只要 `login:` 已经出现，就说明系统至少已经经过：

```text
BootROM
→ U-Boot
→ Linux kernel
→ rootfs
→ systemd/getty
```

所以此时不能下结论说“板子卡死在 PCIe 初始化”或“系统没有启动”。

这也是嵌入式 Linux 调试中非常重要的一点：

```text
某个 driver probe fail
≠ 整个系统 boot fail
```

PCIe 的这些报错可能来自某一路没有连接有效设备。真正要做的是继续向前寻找**第一条会破坏系统调度或锁状态的异常**。

继续分析完整串口日志后，看到在 `NetworkManager` 启动、EtherCAT 网口开始初始化附近出现：

```text
Starting NetworkManager-di…nager Script Dispatcher Service...
[    5.592335] rk_gmac-dwmac-ethercat fe1c0000.ethernet eth0: Register MEM_TYPE_PAGE_POOL RxQ-0
[    5.592820] rk_gmac-dwmac-ethercat fe1c0000.ethernet eth0: Register MEM_TYPE_PAGE_POOL RxQ-1

[  OK  ] Started NetworkManager-dis…Manager Script Dispatcher Service.

[    5.670196] rk_gmac-dwmac-ethercat fe1c0000.ethernet eth0: PHY [stmmac-1:00] driver [RTL8211F Gigabit Ethernet] (irq=POLL)
[    5.670736] dwmac4: Master AXI performs any burst length
[    5.670757] rk_gmac-dwmac-ethercat fe1c0000.ethernet eth0: No Safety Features support found
[    5.670774] rk_gmac-dwmac-ethercat fe1c0000.ethernet eth0: IEEE 1588-2008 Advanced Timestamp supported
[    5.671016] rk_gmac-dwmac-ethercat fe1c0000.ethernet eth0: registered PTP clock
[    5.682334] rk_gmac-dwmac-ethercat fe1c0000.ethernet eth0: FPE workqueue start
[    5.682344] ------------[ cut here ]------------

[    5.682345] rtmutex deadlock detected

[    5.682351] WARNING: CPU: 2 PID: 1110 at kernel/locking/rtmutex.c:1642 __rt_mutex_slowlock_locked.constprop.0+0x128/0x160
[    5.682395] CPU: 2 PID: 1110 Comm: NetworkManager Tainted: G           O       6.1.99-rt36-rk3588 #6
```

这里已经出现了本次故障的第一条强证据：

```text
Comm: NetworkManager
rtmutex deadlock detected
```

也就是说，发生死锁检测时当前进程就是 `NetworkManager`，并且时间点正好位于：

```text
EtherCAT eth0 被打开
```

的过程中。

接着看 call trace：

```text
Call trace:
 __rt_mutex_slowlock_locked.constprop.0+0x128/0x160
 mutex_lock+0x78/0x90
 rtnl_lock+0x18/0x20
 __stmmac_open+0x1c0/0x4c0 [ec_stmmac]
 stmmac_open+0x40/0xd0 [ec_stmmac]
 __dev_open+0xe4/0x1e0
 __dev_change_flags+0x198/0x220
 dev_change_flags+0x20/0x60
 do_setlink+0x5fc/0xdd0
 __rtnl_newlink+0x500/0x880
 rtnl_newlink+0x4c/0x74
 rtnetlink_rcv_msg+0x11c/0x384
 netlink_rcv_skb+0x58/0x124
 rtnetlink_rcv+0x14/0x20
 netlink_unicast+0x268/0x334
 netlink_sendmsg+0x198/0x3ec
 ____sys_sendmsg+0x214/0x27c
 ___sys_sendmsg+0x7c/0xd4
 __sys_sendmsg+0x64/0xc0
```

### 如何读这条调用链

调用栈应该**从下往上理解调用方向**：

```text
用户态 NetworkManager
↓
通过 netlink 请求修改网卡状态
↓
rtnetlink_rcv_msg
↓
do_setlink / dev_change_flags
↓
__dev_open
↓
stmmac_open [ec_stmmac]
↓
__stmmac_open [ec_stmmac]
↓
rtnl_lock
↓
RT-mutex slow path
↓
deadlock detected
```

这里最值得关注的是：

```text
__dev_open
→ stmmac_open
→ __stmmac_open
→ rtnl_lock
```

Linux 网络设备的 open/状态修改本身就是在 RTNL 体系下进行的。如果驱动的 open 路径中又不恰当地获取 RTNL，就存在递归锁获取或错误锁上下文的可能。

因此问题从“NetworkManager 网络配置错误”进一步缩小成：

```text
NetworkManager 是触发者；
ec_stmmac 的 open / locking path 是真正值得审查的底层代码。
```

几毫秒后，串口继续输出：

```text
[    5.689088] BUG: scheduling while atomic: NetworkManager/1110/0x00000002
[    5.689092] Modules linked in: ... stmmac ec_stmmac(O) ... ec_master(O)
[    5.689122] CPU: 2 PID: 1110 Comm: NetworkManager Tainted: G        W  O       6.1.99-rt36-rk3588 #6
[    5.689130] Call trace:
[    5.689159]  __schedule_bug+0x50/0x64
[    5.689166]  __schedule+0x470/0x64c
[    5.689173]  schedule+0x58/0xd0
[    5.689179]  __rt_mutex_slowlock_locked.constprop.0+0x13c/0x160
[    5.689187]  mutex_lock+0x78/0x90
[    5.689191]  rtnl_lock+0x18/0x20
[    5.689196]  __stmmac_open+0x1c0/0x4c0 [ec_stmmac]
[    5.689236]  stmmac_open+0x40/0xd0 [ec_stmmac]
[    5.689261]  __dev_open+0xe4/0x1e0
[    5.689266]  __dev_change_flags+0x198/0x220
[    5.689272]  dev_change_flags+0x20/0x60
[    5.689277]  do_setlink+0x5fc/0xdd0
```

这条信息的含义比普通 warning 严重得多：

```text
scheduling while atomic
```

表示当前执行上下文处于“不应该睡眠/调度”的状态，但代码却进入了会调度的路径。

结合前面的：

```text
rtnl_lock
→ mutex_lock
→ __rt_mutex_slowlock_locked
```

可以理解为：

```text
错误锁上下文
→ RT-mutex 进入慢路径
→ 尝试调度等待
→ 当前上下文又不允许这样调度
→ scheduling while atomic
```

此时系统虽然还能继续打印日志，但内核状态已经不健康了。

大约 60 秒后，串口出现：

```text
[   66.413914] rcu: INFO: rcu_preempt detected stalls on CPUs/tasks:
[   66.413941] rcu:     4-...!: (1 GPs behind) idle=b84c/1/0x4000000000000000 softirq=0/0 fqs=0 rcuc=60003 jiffies(starved)
[   66.413972]  (detected by 0, t=60002 jiffies, g=3213, q=103617 ncpus=8)
...
[   66.414383] rcu: rcu_preempt kthread starved for 60002 jiffies! g3213 f0x2 RCU_GP_WAIT_FQS(5) ->state=0x0 ->cpu=4
[   66.414404] rcu:     Unless rcu_preempt kthread gets sufficient CPU time, OOM is now expected behavior.
```

### 如何分析 RCU stall

这里不能把 RCU stall 当成一个独立的新问题。

RCU 需要 CPU 和 task 定期推进 grace period。如果内核因为前面的锁问题导致 CPU/task 长时间不能正常调度，就会出现：

```text
RCU grace period 无法推进
→ rcu_preempt kthread 长期得不到 CPU
→ RCU stall
```

因此因果顺序应该理解成：

```text
rtmutex deadlock
→ scheduling while atomic
→ 调度 / 锁状态恶化
→ RCU stall
```

而不是：

```text
RCU 配置错误
→ 导致前面的 NetworkManager 死锁
```

这也帮助排除了当时正在调试的：

```text
isolcpus
rcu_nocbs
irqaffinity
```

作为第一根因的可能性。

更晚，在约 205 秒时串口又出现：

```text
[  205.439575] rockchip-spi feb20000.spi: RK SPI transfer timed out
[  205.439591] rk806 spi2.0: SPI transfer failed: -110
[  205.439607] rockchip-spi feb20000.spi: state=0
[  205.439619] rockchip-spi feb20000.spi: tx_left=0
[  205.439630] rockchip-spi feb20000.spi: rx_left=3
...
[  205.439725] spi_master spi2: failed to transfer one message from queue
[  205.439735] spi_master spi2: noqueue transfer failed
[  205.439754] cpu cpu0: mem: failed to set voltage (850000 850000 950000 uV): -110

Error reading from serial device
```

看到：

```text
rk806
failed to set voltage
SPI transfer timed out
```

很容易怀疑供电、PMIC 或 RK806 硬件损坏。

但时间轴告诉我们：

```text
5.68 s   rtmutex deadlock
5.69 s   scheduling while atomic
66.41 s  RCU stall
205.44 s SPI / RK806 timeout
```

所以更加合理的判断是：

```text
RK806 timeout 是系统长时间失稳后的 secondary failure，
而不是整件事的第一根因。
```

这次排查中非常重要的一条经验就是：

> **内核日志不要看“哪一条报错最吓人”，而要看“哪一条最早改变了系统的正常控制流”。**

---

## 3. 系统很快死锁时，用 U-Boot / initramfs 无损救援

因为正常启动后很快进入内核异常，无法可靠地在完整系统里修改配置，于是进入 U-Boot。

上电时按 `Ctrl+C` 停止 autoboot，进入：

```text
=>
```

执行：

```text
ext4ls mmc 0:2 /
```

确认 boot 分区中存在：

```text
System.map-6.1.99-rt36-rk3588
boot.cmd
boot.scr
config-6.1.99-rt36-rk3588
dtb/
extlinux/
initrd-6.1
uEnv/
Image-6.1.99-rt36-rk3588
```

继续：

```text
ext4ls mmc 0:2 /dtb/
```

看到：

```text
rk3588-lubancat-5.dtb
rk3588-lubancat-5io.dtb
rk3588-lubancat-5-v2.dtb
...
```

为了做最小变量 A/B，思路是：

```text
保留 kernel
保留 PREEMPT_RT
保留 ec_stmmac
保留 EtherCAT
只阻止 NetworkManager
```

也就是尝试在这一次启动中加入：

```text
systemd.mask=NetworkManager.service
```

第一次手工启动没有进入正常 rootfs，而是出现：

```text
[  212.673179] mmcblk0: mmc0:0001 EG1061 117 GiB
[  212.678818]  mmcblk0: p1 p2 p3
...
Begin: Waiting for root file system ...
Gave up waiting for root file system device.  Common problems:
 - Boot args (cat /proc/cmdline)
   - Check rootdelay= (did the system wait long enough?)
 - Missing modules (cat /proc/modules; ls /dev)
ALERT!  PARTUUID=614e0000-0000 does not exist.  Dropping to a shell!

BusyBox v1.30.1 ...
(initramfs)
```

### 如何分析这段信息

这里首先看到：

```text
mmcblk0: p1 p2 p3
```

证明：

```text
eMMC 已经被内核识别
分区表也正常
```

但：

```text
PARTUUID=614e0000-0000 does not exist
```

说明问题是：

```text
kernel cmdline 指定了一个错误 / 不完整的 root PARTUUID
```

而不是 eMMC 坏了。

在 `(initramfs)` 中执行：

```sh
cat /proc/cmdline
```

实际看到：

```text
storagemedia=emmc androidboot.storagemedia=emmc androidboot.mode=normal root=PARTUUID=614e0000-0000 boot_part=2 earlyprintk console=ttyFIQ0 consoleblank=0 loglevel=7 rootwait rw rootfstype=ext4 isolcpus=7 rcu_nocbs=7 irqaffinity=0-5 systemd.mask=NetworkManager.service ...
```

继续：

```sh
ls -l /dev/mmcblk*
```

看到：

```text
/dev/mmcblk0
/dev/mmcblk0p1
/dev/mmcblk0p2
/dev/mmcblk0p3
/dev/mmcblk0boot0
/dev/mmcblk0boot1
/dev/mmcblk0rpmb
```

然后：

```sh
blkid /dev/mmcblk0p3
```

得到：

```text
/dev/mmcblk0p3: UUID="c97af8b3-8f31-46cf-9839-e857d41119aa" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="rootfs" PARTUUID="614e0000-0000-4b53-8000-1d28000054a9"
```

这一步把问题彻底分开了：

```text
真实 rootfs:
    /dev/mmcblk0p3
    PARTUUID=614e0000-0000-4b53-8000-1d28000054a9

手工启动使用:
    root=PARTUUID=614e0000-0000
```

所以：

```text
rootfs 没坏
eMMC 没坏
只是手工 bootarg 不正确
```

此时没有继续重刷系统，而是利用 initramfs 直接挂载真实 rootfs：

```sh
mkdir -p /mnt/root
mount -t ext4 /dev/mmcblk0p3 /mnt/root
ls /mnt/root
```

确认真实 Debian 根目录存在后，在 rootfs 上直接创建 systemd mask：

```sh
ln -s /dev/null /mnt/root/etc/systemd/system/NetworkManager.service
sync
umount /mnt/root
reboot -f
```

这等价于正常系统中的：

```bash
systemctl mask NetworkManager.service
```

`mask` 的本质就是：

```text
/etc/systemd/system/NetworkManager.service -> /dev/null
```

比 `disable` 更强，因为 systemd 即使被其他 unit 依赖，也无法再启动这个 service。

---

## 4. A/B 验证成功，再做永久修复

重新按照原来的正常 U-Boot / `boot.scr` / `uEnv.txt` 路径启动，只保留 NetworkManager 被 mask 这一项变化。

这次串口出现：

```text
[  OK  ] Reached target network.target - Network.
[  OK  ] Reached target network-online.target - Network is Online.
...
Starting ssh.service - OpenBSD Secure Shell server...
...
[  OK  ] Started ssh.service - OpenBSD Secure Shell server.
[  OK  ] Started gdm.service - GNOME Display Manager.
...
[  OK  ] Reached target multi-user.target - Multi-User System.
[  OK  ] Reached target graphical.target - Graphical Interface.

Debian GNU/Linux 12 lubancat ttyFIQ0

[username:password] root:root cat:temppwd

lubancat login:
```

最关键的不是“看到 login”，而是这一次**完整日志中没有再出现**：

```text
rtmutex deadlock detected
BUG: scheduling while atomic
rcu_preempt detected stalls
```

而 EtherCAT 相关模块仍然正常加载。

于是得到一个很强的 A/B 结论：

```text
原始状态：
NetworkManager + ec_stmmac
→ deadlock

实验状态：
保留 ec_stmmac / EtherCAT / PREEMPT_RT
只禁止 NetworkManager
→ 系统稳定
```

因此可以高置信度判断：

```text
NetworkManager 自动操作 EtherCAT NIC
是死锁的触发条件。
```

然后需要进一步确认：

```text
到底 eth0 / eth1 哪个物理口是 EtherCAT？
```

执行：

```bash
for i in eth0 eth1; do
    echo "===== $i ====="
    ethtool -i $i
    readlink -f /sys/class/net/$i/device
    cat /sys/class/net/$i/address
done
```

原始输出：

```text
===== eth0 =====
Cannot get driver information: Device or resource busy
/sys/devices/platform/fe1c0000.ethernet
fa:fd:53:a0:a5:55

===== eth1 =====
Cannot get driver information: Device or resource busy
/sys/devices/platform/fe1b0000.ethernet
f6:fd:53:a0:a5:55
```

### 如何分析

`ethtool -i` 虽然因为设备 busy 没拿到 driver name，但：

```text
/sys/class/net/eth0/device
→ fe1c0000.ethernet
```

已经足够建立 Linux netdev 与 SoC GMAC 的映射。

再结合启动日志：

```text
rk_gmac-dwmac-ethercat fe1c0000.ethernet
rk_gmac-dwmac fe1b0000.ethernet
```

可以确定：

```text
eth0 → fe1c0000 → EtherCAT 专用 GMAC
eth1 → fe1b0000 → 普通 Linux GMAC
```

于是永久配置 NetworkManager：

```bash
sudo mkdir -p /etc/NetworkManager/conf.d
sudo nano /etc/NetworkManager/conf.d/99-ethercat-unmanaged.conf
```

写入：

```ini
[keyfile]
unmanaged-devices=mac:fa:fd:53:a0:a5:55
```

这里使用 MAC，而不是：

```ini
interface-name:eth0
```

是为了避免未来驱动 probe 顺序变化导致 `eth0/eth1` 编号交换。MAC 更稳定地代表这一块 EtherCAT 物理 NIC。

恢复 NetworkManager 后执行：

```bash
nmcli device status
ip -br link
ip -br addr
```

最终原始状态为：

```text
DEVICE          TYPE      STATE        CONNECTION
wlan0           wifi      已连接       JuShenShiYanShi_5G
lo              loopback  连接（外部） lo
p2p-dev-wlan0   wifi-p2p  已断开       --
eth1            ethernet  不可用       --
usb0            ethernet  不可用       --
can0            can       未托管       --
dummy0          dummy     未托管       --
eth0            ethernet  未托管       --
```

以及：

```text
lo      UNKNOWN  127.0.0.1/8 ::1/128
wlan0   UP       192.168.5.237/24 fe80::1036:5103:d359:a72f/64
eth0    DOWN
eth1    DOWN
usb0    DOWN
```

这里最终确认：

```text
192.168.5.237 = 开发板 wlan0
```

而：

```text
eth0 = EtherCAT，NetworkManager 未托管
```

这正是希望得到的最终架构：

```text
Management Plane
├── wlan0 → Wi-Fi / SSH / VS Code
└── eth1  → 普通 Ethernet

Real-Time Fieldbus Plane
└── eth0  → IgH EtherCAT / ec_stmmac
             NetworkManager unmanaged
```

所以最终修复并不是：

```text
永远关闭 NetworkManager
```

而是：

```text
让 NetworkManager 正常管理管理网络，
但永远不要触碰 EtherCAT 专用网口。
```

---

## 5. 这次故障真正学到的 Linux 调试方法，以及简历怎么写

这次问题可以压缩成一个非常实用的故障分析框架：

```text
执行一个最小诊断命令
→ 看原始现象
→ 判断问题属于哪一层
→ 根据日志时间顺序找第一条破坏性异常
→ 顺着 call trace 找调用者和 driver
→ 做最小变量 A/B
→ 先恢复系统可维护性
→ 再做永久修复
```

几个关键知识点尤其值得记住。

**第一，SSH timeout 不等于 SSH 出错。**  
先用 `ping`、`arp` 判断 L2/L3 是否成立。如果 ARP 都不成立，就不要浪费时间检查 SSH key 和 VS Code。

**第二，Debug UART 是嵌入式 Linux 的生命线。**  
HDMI 没画面并不能说明 kernel 没启动，而 UART 可以判断系统到底在 BootROM、U-Boot、kernel、rootfs 还是 systemd 哪一层。

**第三，读 kernel log 时要看时间顺序。**  
这次最吓人的 `RK806 SPI transfer failed` 并不是根因。真正第一条破坏系统状态的是 5.68 秒的 `rtmutex deadlock detected`。

**第四，要学会读 call trace。**  
这次调用链不是抽象信息，而是直接告诉了问题路径：

```text
NetworkManager
→ netlink
→ rtnetlink
→ __dev_open
→ stmmac_open [ec_stmmac]
→ rtnl_lock
→ rtmutex deadlock
```

**第五，RCU stall 很多时候是“系统已经被前面的问题拖死”的表现。**  
不能看到 RCU stall 就立即归因于 `rcu_nocbs` 或 CPU isolation。

**第六，复杂问题最有效的方法是 A/B，不是同时改十个东西。**  
这次只改变 NetworkManager 一个变量，保留 EtherCAT、PREEMPT_RT、kernel、CPU isolation，从而建立了很强的因果证据。

**第七，U-Boot / initramfs 是恢复手段，而不是只用于刷机。**  
即使完整系统无法稳定进入 shell，只要 kernel 和 rootfs 还在，就可以从 initramfs 挂载真实 rootfs 修 systemd 配置，避免重刷系统。

简历中可以写成：

```text
RK3588 PREEMPT_RT / EtherCAT 系统级故障定位与恢复

- 在 RK3588 + Debian 12 + Linux PREEMPT_RT + IgH EtherCAT 平台上，
  定位冷启动后 SSH 失联及整机 stall 问题。
- 基于 ARP、Debug UART 与 kernel call trace，
  将问题从网络不可达下钻至
  NetworkManager → ec_stmmac → rtnl_lock 的 RT-mutex 锁异常，
  并分析 scheduling while atomic 与后续 RCU stall 的因果链。
- 使用 U-Boot / initramfs 挂载 eMMC rootfs 完成无重刷救援，
  通过最小变量 A/B 启动验证确认 NetworkManager 为 EtherCAT NIC 死锁触发条件。
- 基于 sysfs 建立 Linux netdev 与 RK3588 GMAC 映射，
  按 MAC 将 EtherCAT NIC 配置为 NetworkManager unmanaged，
  实现管理网络与实时 EtherCAT 数据平面隔离并恢复 Wi-Fi / SSH。
```

面试时最值得讲的不是“我会哪些 Linux 命令”，而是这一句话：

```text
我不是从 SSH 配置开始试，而是先判断故障层级；
不是看到最后一个 error 就猜根因，而是按时间轴找到第一条破坏性异常；
最后通过单变量 A/B 验证，把系统级现象收敛到了一个具体 driver locking path。
```

这才是这次排障最有价值的部分。
