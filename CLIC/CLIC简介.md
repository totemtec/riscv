## 1. CLIC 是什么？

**CLIC** 全称是 **Core-Local Interrupt Controller**，即“核本地中断控制器”。

它是 RISC-V 为每个 hart（硬件线程）设计的一套低延迟中断控制机制，目标是解决传统 RISC-V 中断方案在以下方面的不足：

- 中断响应延迟较高；
- 中断优先级和抢占能力比较简单；
- 很难高效支持中断嵌套；
- 所有中断往往共用一个入口；
- 实时系统中需要软件完成较多分发工作。

可以把 CLIC 理解为：

> 一个更接近实时 MCU/实时 CPU 需求的、支持优先级、抢占和硬件向量化的 RISC-V 本地中断控制器。

---

## 2. 它位于 RISC-V 中断体系的什么位置？

传统 RISC-V 系统中常见的中断结构是：

```text
外设
  │
  ├── PLIC ──> hart 0
  ├── PLIC ──> hart 1
  └── ...
```

PLIC，即 **Platform-Level Interrupt Controller**，主要负责平台级、外部设备产生的中断，例如：

- UART；
- GPIO；
- SPI；
- 网卡；
- DMA；
- 定时器等。

而 CLIC 更偏向于“每个 hart 本地的中断入口和中断调度”：

```text
本地中断源
  │
  └── CLIC ──> 当前 hart
```

实际芯片中可能是：

```text
外设 ──> PLIC/AIA ──> hart
本地外设 ────────────> CLIC ──> hart
```

CLIC 并不一定要完全替换 PLIC。很多系统可以同时使用：

- CLIC：负责本地低延迟中断；
- PLIC 或 AIA：负责平台级外部中断。

---

## 3. CLIC 主要解决哪些问题？

### 3.1 更低的中断延迟

传统中断模式通常是：

1. CPU 收到中断；
2. 跳到统一的 `mtvec` 入口；
3. 软件读取 `mcause`；
4. 软件判断是哪一个中断；
5. 软件跳转到对应的 ISR。

CLIC 可以让硬件更直接地确定目标中断处理入口，减少软件分发步骤。

---

### 3.2 支持更多优先级

传统 RISC-V 的中断优先级相对简单，通常依赖：

- 中断类型；
- 当前特权级；
- 全局中断使能；
- 外部中断控制器中的优先级。

CLIC 可以为每个中断配置：

- enable；
- pending；
- level；
- priority；
- trigger 类型；
- 是否采用硬件向量化等。

因此，不同中断之间可以有更细粒度的优先级关系。

---

### 3.3 支持中断抢占和嵌套

例如：

```text
低优先级 UART 中断正在执行
        │
        └── 高优先级定时器中断到来
                │
                └── 抢占 UART ISR
```

CLIC 可以根据中断的 level、priority 以及当前阈值，决定新中断是否能够抢占当前正在运行的中断。

这对于以下场景很重要：

- 电机控制；
- 音频处理；
- 实时通信；
- 汽车控制；
- 高精度定时；
- RTOS 内核。

---

## 4. CLIC 的基本工作流程

一个典型的 CLIC 中断流程大致如下：

```text
中断源产生请求
      │
      ▼
CLIC 设置 pending
      │
      ▼
检查中断是否 enable
      │
      ▼
比较 level / priority / threshold
      │
      ▼
判断是否可以被当前 hart 接收
      │
      ▼
进入中断处理入口
      │
      ▼
执行 ISR
      │
      ▼
清除 pending 或完成 claim/complete
```

具体行为取决于中断是：

- 电平触发还是边沿触发；
- 普通模式还是硬件向量模式；
- 当前正在执行什么特权级；
- 当前中断阈值是多少；
- 是否允许嵌套和抢占。

---

## 5. CLIC 中常见的寄存器

不同版本的 CLIC 草案以及不同厂商实现可能存在差异，但通常会看到下面这类寄存器。

### 5.1 中断 Pending

表示某个中断是否处于待处理状态。

常见形式类似：

```text
clicintip[i]
```

其中 `i` 表示第 `i` 个中断源。

---

### 5.2 中断 Enable

表示某个中断是否允许被响应。

常见形式类似：

```text
clicintie[i]
```

---

### 5.3 中断属性

配置中断的触发类型、向量化属性等。

常见形式类似：

```text
clicintattr[i]
```

可能涉及：

- 电平触发；
- 上升沿触发；
- 下降沿触发；
- 是否使用选择性硬件向量化；
- 其他中断属性。

不同版本中字段定义可能有所变化。

---

### 5.4 中断控制字段

配置中断的 level 和 priority。

常见形式类似：

```text
clicintctl[i]
```

通常可理解为：

```text
+------------------+----------------+
| interrupt level  | interrupt prio |
+------------------+----------------+
```

具体分配多少位给 level、多少位给 priority，通常由 CLIC 配置决定。

---

### 5.5 CLIC 全局配置

通常有类似：

```text
cliccfg
```

用于设置整个 CLIC 的一些全局属性，例如：

- level 所占位数；
- 某些中断模式；
- 全局行为配置。

---

### 5.6 中断向量表基址

CLIC 引入了用于硬件向量化的向量表基址寄存器，常见形式为：

```text
mtvt
stvt
```

分别对应 Machine 模式和 Supervisor 模式。

当硬件向量化开启时，CPU 可以根据中断 ID 计算或查找对应的处理入口，而不是所有中断都跳到同一个统一入口。

---

## 6. CLIC 的中断向量化

传统的 `mtvec` 有两种常见模式：

### Direct 模式

所有中断都进入同一个地址：

```text
mtvec -> common_interrupt_handler
```

处理函数需要读取 `mcause`，再判断具体中断。

---

### Vectored 模式

不同中断根据 cause 计算不同入口：

```text
base + cause × offset
```

但传统 RISC-V 的 vectored 模式主要是按中断 cause 分发，能力相对有限。

---

### CLIC 硬件向量化

CLIC 可以进一步根据具体中断 ID 直接进入对应的处理入口：

```text
CLIC 中断 ID = 5
        │
        ▼
mtvt + 5 × entry_size
        │
        ▼
中断 5 的入口
```

这样可以减少类似下面的公共分发代码：

```c
void common_handler(void)
{
    int cause = read_mcause();

    if (cause == UART_IRQ)
        uart_isr();
    else if (cause == TIMER_IRQ)
        timer_isr();
    else if (cause == GPIO_IRQ)
        gpio_isr();
}
```

对于高频中断和硬实时系统，这可以明显降低延迟。

---

## 7. level 和 priority 有什么区别？

可以粗略地这样理解：

- **priority**：多个同级别中断同时到来时，谁先处理；
- **level**：某个中断能否抢占当前正在处理的中断。

举例：

```text
中断 A：level = 2, priority = 7
中断 B：level = 1, priority = 3
```

如果 A 正在执行，B 通常不能抢占 A。

如果 B 正在执行，A 由于 level 更高，可能可以抢占 B。

在同一个 level 内，如果多个中断同时可处理，则 priority 用于决定处理顺序。

需要注意，具体抢占规则还受到以下因素影响：

- 当前特权级；
- 全局中断使能；
- 中断阈值；
- 当前处理上下文；
- CLIC 具体版本的规范定义。

---

## 8. CLIC 和 PLIC 的区别

| 特性 | PLIC | CLIC |
|---|---|---|
| 主要定位 | 平台级外部中断控制器 | 每个 hart 的本地中断控制器 |
| 典型用途 | 外部设备中断 | 低延迟、实时、本地中断 |
| 中断抢占 | 能力较有限 | 支持更细粒度抢占 |
| 优先级 | 通常是简单优先级 | 支持 level + priority |
| 向量化 | 软件或传统 vectored 模式 | 支持更直接的硬件向量化 |
| 中断嵌套 | 需要较多软件配合 | 硬件支持更完善 |
| 低延迟能力 | 一般 | 更强 |
| 适合场景 | 通用 Linux 类系统 | MCU、RTOS、实时控制 |

简单说：

> PLIC 更像“平台外部中断汇聚器”，CLIC 更像“面向实时处理的本地中断调度器”。

---

## 9. CLIC 和 AIA 的关系

RISC-V 还发展了 **AIA**，即 Advanced Interrupt Architecture。

AIA 主要解决：

- 多 hart 系统的高级中断管理；
- MSI，即 Message-Signaled Interrupt；
- 虚拟机和虚拟中断；
- 高级平台级中断控制；
- APlic、IMSIC 等机制。

粗略区分：

```text
CLIC：偏向 hart 本地、低延迟、实时中断
AIA ：偏向平台级、多 hart、MSI、虚拟化
```

它们并不是简单的互相替代关系，实际系统可以根据需求组合使用。

---

## 10. 一个简化例子

假设系统有三个中断：

```text
IRQ 0：UART，低优先级
IRQ 1：GPIO，中优先级
IRQ 2：Timer，高优先级
```

可以配置成：

```text
UART  : level = 1, priority = 2
GPIO  : level = 2, priority = 1
Timer : level = 3, priority = 0
```

运行时：

```text
UART ISR 正在运行
        │
        └── GPIO 到来
                │
                └── GPIO level 更高，抢占 UART ISR
                        │
                        └── Timer 到来
                                │
                                └── Timer 再抢占 GPIO ISR
```

处理完成后：

```text
Timer ISR 返回
    ↓
恢复 GPIO ISR
    ↓
GPIO ISR 返回
    ↓
恢复 UART ISR
```

这就是 CLIC 面向实时中断嵌套的典型使用方式。

---

## 11. 使用 CLIC 时需要注意的地方

### 11.1 CLIC 规范长期处于演进状态

CLIC 曾经长期以 draft 规范形式发展，不同版本之间寄存器字段和细节可能不完全一致。

因此开发时要同时确认：

- RISC-V CLIC 规范版本；
- CPU IP 核实现版本；
- 芯片手册；
- 启动代码和编译器支持；
- 操作系统移植层。

不能只根据某一份早期草案写驱动。

---

### 11.2 不同实现支持程度不同

有的实现可能支持：

- 多级优先级；
- 抢占；
- 硬件向量化；
- 多种触发方式。

有的实现可能只实现其中一部分，甚至提供厂商自定义扩展。

---

### 11.3 编译器和 ABI 也要配合

中断嵌套、快速中断入口、特殊寄存器保存恢复，通常需要：

- 特定的启动代码；
- 特定的 ISR 属性；
- 正确保存寄存器；
- 正确设置 `mstatus`、阈值和特权状态；
- 有时还需要汇编入口包装。

不能只改几个 CLIC 寄存器就自动获得完整的嵌套中断能力。

---

## 12. 一句话总结

**CLIC 是 RISC-V 面向低延迟和实时场景设计的每核本地中断控制器，它在传统中断机制之上增加了更细粒度的优先级、level、抢占、嵌套和硬件向量化能力。**

如果把传统中断比作：

> “所有电话先打到总机，再由软件转接”

那么 CLIC 更像：

> “硬件根据号码、优先级和当前状态，直接把电话接到对应处理函数，并决定是否打断当前通话。”
