---
创建者: WeiXing
类别:
  - 402 OS
---
# Sixth-Edition Unix (1975)

- 单体内核 (**Monolithic OS**)：一个完整的可执行文件，只有 ~64KB。
- 系统启动时直接加载到内存。
- 强调基本功能：**Processes, Address Spaces, Threads, Files**。
- 影响深远 → 现代 OS (Linux, macOS, Windows) 的设计灵感。

# Processor Modes

- **Privileged Mode (Kernel Mode)**
    - 权限最高，可以直接操作硬件。
    - 整个 **OS kernel** 运行在这个模式下。
- **User Mode**
    - 普通应用运行在这里，权限有限。
    - 必须通过 **system calls (via traps)** 请求 OS 服务。
- 有些系统甚至把部分子系统放在 **user mode** 下运行（灵活性 vs 性能）。

# OS 与事件处理

- **Traps**
    - 来自程序 → 通过 **system calls** 请求 OS（例如 `read()` / `write()`）。
    - CPU 切换到 **privileged mode**，进入内核处理。
- **Interrupts**
    - 来自外部设备 → 例如 **I/O completion interrupt**（磁盘完成写入、键盘输入）。
    - CPU 暂停当前执行 → 跳转执行 **Interrupt Service Routine (ISR)**。

# Hardware Support

- 复习计算机组成：
    - **Registers**：EIP, ESP, EBP, EAX, EBX 等。
    - **Bus architecture**：
        - Address Bus (A[0..31])
        - Data Bus (D[0..31])
        - Control lines (RD, WR, INT, LOCK)
- **Read Bus Cycle**：CPU → 地址总线 (A) → RD → 数据总线 (D) → CPU 取数据。
- **Write Bus Cycle**：CPU → 地址总线 (A) + 数据总线 (D) → WR → 数据写入内存/设备控制器。