## 概述

本仓库是 STMicroelectronics STM32CubeF1 的 HAL（硬件抽象层）与 LL（底层寄存器层）C 驱动库。它没有运行时的配置文件加载机制（无 YAML/TOML/env/properties 等），而是采用纯 **C 预处理器宏** 作为唯一的配置入口，全部在编译期生效。

## 核心机制

### 1. 主配置入口：`stm32f1xx_hal_conf_template.h`

该文件位于 `Inc/` 下，文档明确说明：

> "This file should be copied to the application folder and renamed to stm32f1xx_hal_conf.h."

应用工程应复制此模板为 `stm32f1xx_hal_conf.h`，并通过修改其中的 `#define` 来定制构建。顶层头文件 `Inc/stm32f1xx_hal.h` 通过 `#include "stm32f1xx_hal_conf.h"` 引入它，因此这是整个 HAL 配置树的根。

配置分为若干逻辑区块（均以注释标题分隔）：
- **模块选择**：`HAL_<PERIPHERAL>_MODULE_ENABLED`（如 `HAL_ADC_MODULE_ENABLED`、`HAL_UART_MODULE_ENABLED` 等），控制各外设驱动是否参与编译。
- **振荡器值**：`HSE_VALUE`、`HSI_VALUE`、`LSI_VALUE`、`LSE_VALUE` 及其 `_STARTUP_TIMEOUT`，用于 RCC 时钟树计算；支持通过工具链编译器预处理器覆盖（见注释 "Tip: To avoid modifying this file... define the HSE value in your toolchain compiler preprocessor"）。
- **系统配置**：`VDD_VALUE`、`TICK_INT_PRIORITY`、`USE_RTOS`、`PREFETCH_ENABLE`。
- **回调启用开关**：`USE_HAL_<PERIPHERAL>_REGISTER_CALLBACKS`，默认全部为 `0U`，按外设逐一开启可选的回调注册能力。
- **断言**：`USE_FULL_ASSERT` 控制 assert 宏展开。
- **以太网 PHY 配置**：MAC 地址、缓冲区大小、DP83848 地址等。

所有数值常量均使用 `#if !defined(...)` 包裹并提供默认值，允许外部覆盖。

### 2. 公共定义与行为开关：`stm32f1xx_hal_def.h`

该文件提供 HAL 通用的类型与宏，其中关键配置点：
- `USE_RTOS` 被检查：当等于 `1U` 时直接触发 `#error "USE_RTOS should be 0 in the current HAL release"`，强制当前版本不支持 RTOS 集成。
- `__HAL_LOCK` / `__HAL_UNLOCK` 宏在未启用 RTOS 时实现基于 handle->Lock 的简单互斥；若启用 RTOS，则预留扩展点。
- 编译器差异化：针对 ARMCC V5/V6、GNU GCC、ICCARM 分别定义 `__weak`、`__packed`、`__ALIGN_BEGIN`、`__ALIGN_END`、`__RAM_FUNC`、`__NOINLINE` 等属性宏。

### 3. 包含顺序与依赖

```text
stm32f1xx_hal.h → stm32f1xx_hal_conf.h ← 用户/工具链覆盖
                ↓
stm32f1xx_hal_def.h → stm32f1xx.h (芯片系列头) + Legacy/stm32_hal_legacy.h
```

所有 HAL 源文件通过统一入口 `stm32f1xx_hal.h` 间接获得配置，保证编译期宏的一致性。

### 4. 运行时零开销

由于所有配置都是 `#define` 宏，未启用的模块代码会被预处理器彻底裁剪掉，不产生任何运行时开销。不存在配置解析函数、配置文件读取路径或运行时特征探测。

## 约定与约束

- **模板必须复制到应用目录并重命名为 `stm32f1xx_hal_conf.h`**——由 `stm32f1xx_hal_conf_template.h` 顶部注释明确要求，且 `stm32f1xx_hal.h` 只 include 该文件名。
- **当前 HAL 版本禁止启用 RTOS**——`stm32f1xx_hal_def.h` 中 `USE_RTOS == 1U` 会触发编译期 `#error`。
- **每类外设对应独立的 ENABLED/CALLBACKS 宏对**——`HAL_<X>_MODULE_ENABLED` 控制编译，`USE_HAL_<X>_REGISTER_CALLBACKS` 控制回调接口生成，二者独立。
- **振荡器频率默认值通过 `#if !defined(...)` 保护**——可在工具链编译器命令行通过 `-DHSE_VALUE=...` 等方式覆盖，无需修改模板。
- **断言功能默认关闭**——`USE_FULL_ASSERT` 默认注释掉，需显式取消注释以启用。
- **LL 层（`stm32f1xx_ll_*.h`）不经过 HAL 配置层**，直接暴露寄存器级 API，不受 `HAL_*_MODULE_ENABLED` 控制。

## 关键文件

- `Inc/stm32f1xx_hal_conf_template.h` — HAL 配置模板（模块开关、时钟、系统参数、回调开关、以太网配置）
- `Inc/stm32f1xx_hal.h` — HAL 总入口，唯一 include `stm32f1xx_hal_conf.h`
- `Inc/stm32f1xx_hal_def.h` — HAL 公共类型、锁宏、编译器属性宏、RTOS 限制
- `Src/stm32f1xx_hal.c` — HAL 初始化入口，消费上述配置宏
- `Inc/Legacy/stm32_hal_legacy.h` — 旧版 HAL 兼容宏