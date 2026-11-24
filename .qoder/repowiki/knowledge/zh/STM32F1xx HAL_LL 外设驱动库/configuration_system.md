## 概述

本仓库是 STMicroelectronics STM32CubeF1 包，为 F100/F101/F102/F103 xx6/xb/xe/xg（含 xxx5c/xxx7c）MCU 提供完整的 CMSIS Core 与 Device 基础设施，以及对应的 HAL/LL C 驱动。配置入口仍然是 **纯 C 预处理器宏**（无 YAML/TOML/env/properties 等运行时加载机制），并在本增量中继续携带 CMSIS Core/Device 完整实现。

仓库根目录不存在 `config/`、`.yaml`、`.toml`、`.env*`、`application.properties` 等运行时配置文件；仅在 `.qoder` 知识库索引中使用了两份 `.yaml`（`_module.yaml`、`_index.yaml`），属于元数据仓库索引，不参与构建与运行时的配置系统。

## 核心机制

### 1. HAL 层双配置文件

#### `Inc/stm32f1xx_hal_conf_template.h`（模板文档）
- 顶部注释明确声明："This file should be copied to the application folder and renamed to stm32f1xx_hal_conf.h"
- 作为用户定制的起点，模块默认全部注释禁用（`/* #define HAL_<X>_MODULE_ENABLED */`），供应用按需启用

#### `Inc/stm32f1xx_hal_conf.h`（新增——包内预设默认值）
- 所有外设的 `HAL_MODULE_ENABLED` / `HAL_<X>_MODULE_ENABLED` **全部启用**（ADC、CAN、CEC、CORTEX、CRC、DAC、DMA、ETH、EXTI、FLASH、GPIO、HCD、I2C、I2S、IRDA、IWDG、NAND、NOR、PCCARD、PCD、PWR、RCC、RTC、SD、SMARTCARD、SPI、SRAM、TIM、UART、USART、WWDG、MMC）
- 所有 `USE_HAL_<X>_REGISTER_CALLBACKS` **默认开启**（不再默认为 `0U`），允许调用方注册回调
- 振荡器、系统参数、断言开关、以太网 PHY 等区块结构与模板一致，但处于可用状态
- 新文件被 `stm32f1xx_hal_def.h` 直接 include；模板文件不参与正常构建流程，仅用于指导用户生成自定义版本

这种“模板 + 包内默认”的双份结构，使现有应用继续使用覆盖的 `stm32f1xx_hal_conf.h`，而直接从库源码构建的工程获得全功能默认配置。

### 2. 芯片系列选择：`CMSIS/Device/ST/STM32F1xx/Include/stm32f1xx.h`

通过编译器预处理宏（如 `STM32F101x6`、`STM32F103xE`、`STM32F105xC`）展开为对应具体器件头：

| 系列 | 变体 |
|---|---|
| STM32F100 | stm32f100xb.h / stm32f100xe.h |
| STM32F101 | stm32f101x6.h / stm32f101xb.h / stm32f101xe.h / stm32f101xg.h |
| STM32F102 | stm32f102x6.h / stm32f102xb.h |
| STM32F103 | stm32f103x6.h / stm32f103xb.h / stm32f103xe.h / stm32f103xg.h |
| STM32F105/107 | stm32f105xc.h / stm32f107xc.h |

每个具体头文件进一步导出 `__STM32F1XX_HAL` 与 `_STM32F1XX_` 家族宏，供 HAL/LL 源文件在内部进行条件编译。

### 3. System/时钟配置入口

- `CMSIS/Device/ST/STM32F1xx/Include/system_stm32f1xx.h` 声明 `SystemInit(void)`、`SystemCoreClockUpdate(void)`、全局变量 `SystemCoreClock` 及 AHB/APB 预分频表
- 该头文件中 `SYS_CLK_FREQ` 宏用于 RCC 初始频率计算（配合外部工具链 `-D` 参数覆盖 HSE/HSI/LSE_VALUE）
- 所有 OSC_*_VALUE 常量均以 `#if !defined(...)` 保护默认值，可在工具链命令行覆盖，无需修改模板

### 4. CMSIS Core 与编译器适配

| 组件 | 路径 | 作用 |
|---|---|---|
| Core API | `CMSIS/Core/Include/core_cm3.h` | Cortex-M3 内核寄存器访问、异常处理、Systick 等 |
| 编译器分发 | `CMSIS/Core/Include/cmsis_compiler.h` | 按 ARMCLANG/GCC/ARMCC/IAR 分发到对应 `<cmsis_<compiler>.h>` |
| GCC 适配 | `cmsis_gcc.h` | `__weak`、`__packed`、`__ALIGNED()`、`FORCE_INLINE` 等属性宏 |
| ARM Compiler | `cmsis_armcc.h` | 同目标，针对 Keil MDK v5 (ARMCC) 和 v6 (armclang) |
| IAR 适配 | `cmsis_iar.h` | IAR EWARM 对应属性映射 |
| MPU | `mpu_armv7.h` | Cortex-M7/M3 MPU 抽象（M3 下为空壳） |

`stm32f1xx_hal_def.h` 中的 `#if defined(__ICCARM__)` / `__GNUC__` / `__CC_ARM` / `__ARMCC_VERSION` / `__clang__` 分支选择编译器头，确保 `__weak`、`__packed`、`__RAM_FUNC`、`__NOINLINE` 在所有支持的工具链上一致。

### 5. 包含顺序与依赖拓扑

```
C 源文件
→ stm32f1xx_hal.h
→ stm32f1xx_hal_conf.h（包内默认）← 可被用户 stm32f1xx_hal_conf.h 覆盖
→ stm32f1xx_hal_def.h
→ CMSIS/Device/.../stm32f1xx.h → stm32f103xe.h（示例）→ system_stm32f1xx.h
→ CMSIS/Core/Include/cmsis_compiler.h → cmsis_gcc.h / cmsis_armcc.h / ...
```

HAL/LL 源文件不直接 include 芯片头或 CMSIS Core，而是统一通过 `stm32f1xx_hal.h` 间接引入，保证所有模块看到同一组配置宏。

### 6. 运行时零开销

所有行为由 `#define` 宏决定：未启用的模块代码被预处理器彻底裁剪，不产生任何运行时解析开销。不存在配置文件读取函数、环境变量枚举或运行时特性探测。

## 约定与约束

- **模板复制到应用目录并重命名为 `stm32f1xx_hal_conf.h`**——由 `stm32f1xx_hal_conf_template.h` 顶部注释明确要求；`stm32f1xx_hal_conf.h` 是包内置入的版本，`stm32f1xx_hal_conf_template.h` 仅供编辑参考。
- **当前 HAL 版本禁止启用 RTOS**——`stm32f1xx_hal_def.h` 中 `USE_RTOS == 1U` 触发 `#error "USE_RTOS should be 0 in the current HAL release"`。
- **模块控制宏对独立存在**——`HAL_<X>_MODULE_ENABLED` 控制编译，`USE_HAL_<X>_REGISTER_CALLBACKS` 控制回调接口生成，二者互不影响。
- **OSC_*_VALUE 常量可通过 `-D` 覆盖**——模板使用 `#if !defined(HSE_VALUE)` 包裹默认值，遵循工具链命令行的优先级覆盖规则。
- **断言功能默认关闭**——模板中 `USE_FULL_ASSERT` 默认注释掉，需显式取消注释以启用。
- **LL 层不经过 HAL 配置层**——`stm32f1xx_ll_*.h` 直接暴露寄存器级 API，不受 `HAL_*_MODULE_ENABLED` 控制。
- **编译期必须定义一个 `STM32F10X_...` 系列宏**——`stm32f1xx.h` 用 `#ifndef __STM32F1XX_HAL` 校验，若未定义会触发 `#error "Please select first the target STM32F1xx device used in your application"`（该错误位于 stm32f1xx.h 本身）。