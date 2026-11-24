## 1. 采用的体系

仓库为 STM32CubeF1 HAL/LL C 驱动包，错误处理集中在 HAL 层：**同步 API 返回 `HAL_StatusTypeDef`（`HAL_OK/HAL_ERROR/HAL_BUSY/HAL_TIMEOUT`）**，参数合法性通过 `assert_param(expr)` 宏校验。CMSIS Core 与 Device 的 NMI、HardFault、BusFault、MemManage、UsageFault、DebugMon 等 CPU 异常 handler 在外部 Core/Device 目录中以 `__weak` 空函数形式提供，允许应用覆写——属于裸机环境下的不可恢复故障兜底入口。

不存在 C++ 异常、全局 errno、结构化错误对象、panic/recover 或中间件式错误传播框架。

## 2. 关键文件

- **`Inc/stm32f1xx_hal_def.h`**：定义 `HAL_StatusTypeDef`、`HAL_LockTypeDef`（`HAL_UNLOCKED/HAL_LOCKED`），以及 `__HAL_LOCK/__HAL_UNLOCK`；强制编译期约束 `USE_RTOS == 0`。
- **`Inc/stm32_assert_template.h`**：`assert_param` 模板实现，注释明确应复制到应用目录并重命名为 `stm32_assert.h`，由用户自己的 `stm32f1xx_hal_conf.h` 包含覆盖默认行为。
- **`Inc/stm32f1xx_hal_conf_template.h`**：保留 `USE_FULL_ASSERT` 开关样板。
- **外设头/源文件（40+）**：所有 `Inc/stm32f1xx_hal_*.h` 与 `Src/stm32f1xx_hal_*.c`、`Inc/stm32f1xx_ll_*.h`、`Src/stm32f1xx_ll_*.c` 统一采用该模式（GPIO、I2C、SPI、UART、TIM、RCC、RTC、DMA、Flash、SDMMC、USB、ADC、DAC、CAN、ETH 等）。
- **CMSIS Core / Device（外部）**：NMI、HardFault 等的弱符号声明位于未被此仓库打包的 `Core/Include/core_cm3.h` 及 `Device/ST/STM32F1xx/Include/*.h`，当前 STM32CubeF1 源码树不包含这些路径，仅引用其符号。

## 3. 架构与约定

### 3.1 `HAL_StatusTypeDef` 四值语义
```c
typedef enum {
    HAL_OK       = 0x00U,
    HAL_ERROR    = 0x01U,
    HAL_BUSY     = 0x02U,
    HAL_TIMEOUT  = 0x03U
} HAL_StatusTypeDef;
```
调用方依据返回值分支判断。

### 3.2 资源锁定与并发约束
`USE_RTOS != 1` 时，`__HAL_LOCK(__HANDLE__)` 在句柄已 `HAL_LOCKED` 时直接返回 `HAL_BUSY`；否则置锁；`__HAL_UNLOCK(__HANDLE__)` 清空 Lock。RTOS 路径被禁用：`#error "USE_RTOS should be 0 in the current HAL release"`。

### 3.3 `assert_param` 断言链
启用 `USE_FULL_ASSERT`：
```
assert_param(expr)
→ ((expr) ? (void)0U : assert_failed((uint8_t *)__FILE__, __LINE__))
```
关闭时为 `(void)0U`，零运行时开销。

### 3.4 异常/故障底层（弱符号钩子）
Core 头文件中 NMI、HardFault、BusFault、MemManage、UsageFault、DebugMon 以 `__weak` 空函数声明，应用覆写可实现硬件故障诊断、调试器断入或安全停机。设备头文件列出对应向量符号。这是不可恢复错误的最低层入口，而非库层面的返回值机制。

## 4. 约定与约束

- **约定（观察到的行为）**：所有 HAL/LL API 返回 `HAL_StatusTypeDef`；函数入口对传入指针与枚举参数使用 `assert_param`；跨实例互斥通过 `Handle->Lock` + `__HAL_LOCK/__HAL_UNLOCK` 实现；CPU 异常钩子以弱符号暴露给应用。
- **约束（编译期强制）**：
  - `stm32f1xx_hal_def.h` 中 `#if (USE_RTOS == 1U)` 分支触发 `#error "USE_RTOS should be 0 in the current HAL release"`，当前版本禁止启用 RTOS 集成。
  - `stm32_assert_template.h` 根据 `USE_FULL_ASSERT` 选择展开为带行号与文件名的 `assert_failed` 调用或空操作，二者之一必然生效。
  - `stm32f1xx_hal_conf_template.h` 要求将模板复制为实际配置头并设置 `USE_FULL_ASSERT` 等宏后才有效。
- **无以下机制**：未发现 C++ 异常、全局 errno、结构化错误对象、panic/recover 或任何中间件风格错误传播框架。
- **LL 层行为差异**：`stm32f1xx_ll_*.c` 广泛使用 `assert_param`，但通常不返回 `HAL_StatusTypeDef`，而是直接寄存器访问，错误判断交由上层 HAL 完成。