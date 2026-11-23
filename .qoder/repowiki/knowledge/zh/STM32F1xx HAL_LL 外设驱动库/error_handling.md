## 1. 采用的体系

该仓库包含 STM32CubeF1 的 HAL/LL C 驱动以及新增的 CMSIS Core/Device 基础设施。错误处理分两层：

- **HAL 层**：同步 API 通过 `HAL_StatusTypeDef` 枚举返回状态；参数合法性通过 `assert_param(expr)` 宏校验。
- **CMSIS Core/Device 层**：CPU 异常（NMI、HardFault、MemManage、BusFault、UsageFault、DebugMon）均以 ARM Compiler / GCC 的 `__weak` 空函数形式声明，允许应用覆写——这是裸机环境下的“最低层错误/故障”兜底机制。

不存在 C++ 异常、errno、结构化错误对象或中间件式传播。

## 2. 关键文件

### HAL 公共定义与配置
- `Inc/stm32f1xx_hal_def.h`：定义 `HAL_StatusTypeDef`（HAL_OK/HAL_ERROR/HAL_BUSY/HAL_TIMEOUT）、`HAL_LockTypeDef`、`__HAL_LOCK/__HAL_UNLOCK`，并强制编译期约束 `USE_RTOS == 0`（见下方约定）。
- `Inc/stm32f1xx_hal_conf.h`（本次引入的新头文件）：`#ifdef USE_FULL_ASSERT` 分支直接定义 `assert_param` 和 `void assert_failed(uint8_t* file, uint32_t line);`。未启用时断言为空操作。
- `Inc/stm32_assert_template.h`：`assert_param` 模板版本，注释明确说明应复制到应用目录并重命名为 `stm32_assert.h`，以便 `stm32f1xx_hal_conf.h` 包含自定义实现。`stm32f1xx_hal_conf_template.h` 中保留对应的开关样板。

### CMSIS Core 与 Device 异常入口
- `CMSIS/Core/Include/core_cm3.h`：为 Cortex-M3 暴露 NMI、HardFault、MemManage、BusFault、UsageFault、DebugMon 等 handler 的弱符号原型，由系统启动代码链接到默认空实现。
- `CMSIS/Device/ST/STM32F1xx/Include/*.h`（stm32f105xc.h、stm32f107xc.h、stm32f103x{g,e,b,6}.h、stm32f101x{g,e,b,6}.h、stm32f100x{e,b,6}.h、stm32f102x{b,6}.h）：器件专属的中断向量表声明。

### 外设源文件
所有 HAL `.c` 与 LL `.c`（约 40 个）统一返回 `HAL_StatusTypeDef`，并以 `assert_param` 做边界检查（如 GPIO、I2C、SPI、UART、TIM、RCC、RTC、DMA、Flash、SDMMC、USB、ADC、DAC 等）。

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
调用方依据返回值分支。该类型被 `Src/` 下全部 HAL 源文件一致使用。

### 3.2 资源锁定
`USE_RTOS != 1` 时，`__HAL_LOCK(__HANDLE__)` 在句柄已被锁定时直接返回 `HAL_BUSY`；否则置 `Lock == HAL_LOCKED`。解锁仅清空该字段。

### 3.3 assert_param 断言链
启用 `USE_FULL_ASSERT`：
```
assert_param(expr)
→ ((expr) ? (void)0U : assert_failed((uint8_t *)__FILE__, __LINE__))
→ assert_failed(file, line)
```
关闭时为 `(void)0U`，零运行时开销。

### 3.4 异常/故障底层（CMSIS）
CMSIS Core 将 NMI、HardFault、BusFault、MemManage、UsageFault、DebugMon 以 `__weak` 空函数形式声明，应用可覆写以实现硬件故障诊断、进入调试器或安全停机；设备头文件则列出对应向量符号。这属于最底层的“不可恢复错误”入口，而非库层面的错误返回码。

## 4. 约定与约束

- **约定（观察到的行为）**：HAL 函数返回 `HAL_StatusTypeDef`；参数合法性经 `assert_param` 验证；跨实例互斥通过 `Handle->Lock` 配合 `__HAL_LOCK/__HAL_UNLOCK` 实现；CMSIS 层提供弱符号异常钩子供应用覆写。
- **约束（编译期强制）**：
  - `stm32f1xx_hal_def.h` 中包含 `#if (USE_RTOS == 1U) ... #error "USE_RTOS should be 0 in the current HAL release"`，当前 HAL 版本禁止启用 RTOS 集成，并发由应用自行保证。
  - `assert_param` 的行为完全由 `USE_FULL_ASSERT` 决定：启用则展开为带行号调用的 `assert_failed`，关闭则为空操作。
- **无以下机制**：未发现 C++ 异常、全局 errno 结构、结构化错误对象、panic/recover 或任何“中间件”风格错误传播框架。
- **LL 层**：`stm32f1xx_ll_*.c` 广泛使用 `assert_param`，但通常不暴露 `HAL_StatusTypeDef`，而是直接寄存器访问，错误判断交由上层完成。
