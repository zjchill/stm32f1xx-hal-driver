## 1. 使用的系统/方法

本仓库是 STM32CubeF1 HAL/LL 驱动库，不包含任何独立的日志框架（无 `log/`、`logging/` 目录，也无结构化日志）。运行时诊断仅由两类手段组成：
- **参数校验断言**：通过宏 `assert_param(expr)` 对公开 API 的参数进行检查；其默认实现是空操作，需在应用侧复制并启用 `USE_FULL_ASSERT` 以调用用户实现的 `assert_failed(file, line)`。
- **条件式 `printf` 输出**：仅在极少量源文件中使用标准 C `printf`/`sprintf`/`puts` 作为原始调试输出，未封装为统一的日志级别或 sink。

该库不依赖 FreeRTOS log、SEGGER SystemView、SWO/ITM 等嵌入式日志中间件——任何此类能力均由应用集成层提供。

## 2. 关键文件

- `Inc/stm32_assert_template.h`：`assert_param` 宏的模板实现。注释明确说明 “This file should be copied to the application folder and renamed to stm32_assert.h”，因此 assert 的行为完全由调用方替换实现决定。
- `Inc/stm32f1xx_hal_conf_template.h`：HAL 配置头模板，其中定义了 `ASSERT_TIMEOUT_VALUE` 等调试相关宏，但未见日志级别/通道开关。`DEBUG` / `DEBUG_LEVEL` / `USE_DEBUG` 在头文件中未找到定义（搜索结果为 0）。
- `Src/stm32f1xx_hal_tim_ex.c`：出现 3 处 `printf`/`sprintf` 调用。
- `Src/stm32f1xx_hal_adc.c`：1 处 `printf`。
- `Src/stm32f1xx_ll_tim.c`：2 处 `printf`/`sprintf`。
- `Inc/*_hal*.h` 中的大量枚举（如 `HAL_UART_StateTypeDef` 配合错误码字符串）配合 389 处含 `Debug`/`Error` 关键字的出现，是 HAL 的错误/状态上报形式，而非日志流。

## 3. 架构与约定

- **断言即唯一受约束的诊断点**：`stm32_assert_template.h` 中 `assert_param` 的定义被 `USE_FULL_ASSERT` 宏严格控制，否则直接 `((void)0U)`。这使断言在 Release 可构建时为死代码，不引入额外二进制体积，除非应用显式启用。
- **没有日志级别体系**：仓库内未定义 `LOG_INFO`/`LOG_WARN`/`LOG_ERROR` 等统一级别，也未发现按级别的开关宏。
- **`printf` 属于遗留调试痕迹**：上述三个 `.c` 文件中出现的 `printf` 调用散乱分布，不是统一日志接口，且可能包含浮点格式化（需链接新版 libc），在裸机应用中通常不可用。
- **外部 sink 归属应用**：文档化约定把 `assert_failed` 的实现交由应用工程自定义；同理，HAL/LL 本身不会主动初始化 UART、SWO 或外部日志设备。

## 4. 约定与约束

- `assert_param(expr)` 是 HAL/LL 公共 API 参数校验的统一入口；其行为完全由是否在 `stm32_assert.h`（应用侧文件）中定义 `USE_FULL_ASSERT` 决定。（来源：`Inc/stm32_assert_template.h`）
- 禁用断言时宏体为 `(void)0U`，保证断言分支零开销。（来源：`Inc/stm32_assert_template.h`）
- 不存在仓库级日志级别常量、日志记录函数或 sink 注册表；`printf` 仅在极少数 `.c` 文件中作为非正式诊断输出存在，不应被视为稳定 API。
- 库自身不调用 `InitLogging`、`Log_*`、`SYS_LOG` 或任何第三方 logger 初始化例程，也没有 `_htmresc/mini-st_2020.css` 之类用于日志 Web UI 的前端资源（CSS 实际属项目根目录下的 HTML 文档生成产物）。

简言之：本仓库的“日志系统”是一个空洞——它仅提供可被应用覆盖的断言外壳和零星 `printf` 调试语句，真正的日志框架、级别策略与输出路由必须由上层应用提供。