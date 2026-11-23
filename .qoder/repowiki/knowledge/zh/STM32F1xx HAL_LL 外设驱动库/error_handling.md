## 1. 采用的体系

该仓库是 STMicroelectronics STM32CubeF1 的 HAL（硬件抽象层）与 LL（底层寄存器层）C 驱动集合，错误处理采用 ST HAL 的标准模式：
- **同步 API 通过枚举 `HAL_StatusTypeDef` 返回状态**，而不是 C++ 异常或 errno。
- **运行时参数校验通过 `assert_param(expr)` 宏**完成，由应用层提供 `assert_failed` 实现。
- **无 panic/recover、无中间件式错误传播**——这是裸机嵌入式库，不存在进程级异常模型。

## 2. 关键文件

- `Inc/stm32f1xx_hal_def.h`：定义核心错误类型 `HAL_StatusTypeDef`、`HAL_LockTypeDef`，以及锁宏 `__HAL_LOCK/__HAL_UNLOCK`。
- `Inc/stm32_assert_template.h`：`assert_param` 模板，需复制到应用目录并命名为 `stm32_assert.h`。
- `Inc/stm32f1xx_hal_conf_template.h`：用户可在此启用 `USE_FULL_ASSERT` 等配置开关。
- `Src/stm32f1xx_hal.c`：HAL 公共逻辑，包含大量 `assert_param` 校验与 `HAL_OK/HAL_ERROR/HAL_BUSY/HAL_TIMEOUT` 返回。
- 各外设源文件（`stm32f1xx_hal_{i2c,spi,uart,tim,...}.c`）：统一使用上述两种机制。

## 3. 架构与约定

### 3.1 HAL_StatusTypeDef 状态码
`stm32f1xx_hal_def.h` 定义了四值枚举：
```c
typedef enum {
    HAL_OK       = 0x00U,
    HAL_ERROR    = 0x01U,
    HAL_BUSY     = 0x02U,
    HAL_TIMEOUT  = 0x03U
} HAL_StatusTypeDef;
```
所有 HAL 函数返回此类型；调用方根据返回值分支。搜索显示 `Src/` 下约 44 个 `.c` 文件共出现 2312 次相关返回语句，覆盖 GPIO、I2C、SPI、UART、TIM、RCC、RTC、DMA、Flash、SDMMC、USB 等全部 HAL 模块。

### 3.2 资源锁定语义
`__HAL_LOCK(__HANDLE__)` 内部检查 `Handle->Lock == HAL_LOCKED`，若已加锁则直接 `return HAL_BUSY`；否则置 `HAL_LOCKED`。解锁仅清空锁位。当 `USE_RTOS == 1` 时编译为 `#error "USE_RTOS should be 0 in the current HAL release"`，即当前 HAL 版本不支持 RTOS 集成，并发由应用自行保证。

### 3.3 assert_param 参数校验
`stm32_assert_template.h` 给出模板：
- 若应用定义 `USE_FULL_ASSERT`，`assert_param(expr)` 展开为 `((expr) ? (void)0U : assert_failed((uint8_t *)__FILE__, __LINE__))`，并声明 `void assert_failed(uint8_t* file, uint32_t line);`。
- 未定义 `USE_FULL_ASSERT` 时，`assert_param(expr)` 退化为 `(void)0U`，零开销。

模板注释明确说明：**该文件应复制到应用目录并重命名为 `stm32_assert.h`**，以便 `stm32f1xx_hal_conf.h` 中包含自定义实现而非模板。

### 3.4 典型调用模式
在 `Src/stm32f1xx_hal_gpio.c`、`stm32f1xx_hal_i2c.c`、`stm32f1xx_hal_rcc.c` 等文件中，常见模式为：
```c
void HAL_XXX_Init(XXX_HandleTypeDef *hxxx) {
    assert_param(IS_XXX_INSTANCE(hxxx->Instance));
    assert_param(IS_XXX_MODE(...));
    /* ... */
    if (timeout == 0xFFFFFFFFU) return HAL_ERROR;
    /* ... */
    return HAL_OK;
}
```
即先做参数断言，再在超时/失败路径返回 `HAL_ERROR` / `HAL_TIMEOUT` / `HAL_BUSY`。

## 4. 约定与约束

- **约定（观察到的行为）**：所有 HAL 函数的返回值类型为 `HAL_StatusTypeDef`；参数合法性通过 `assert_param` 检查；互斥访问通过 `__HAL_LOCK/__HAL_UNLOCK` 配合 Handle 中的 `Lock` 字段实现。
- **约束（被代码强制）**：
  - 当前 HAL 版本禁止 `USE_RTOS == 1`：`stm32f1xx_hal_def.h` 中显式 `#error "USE_RTOS should be 0 in the current HAL release"`。
  - `assert_param` 的行为由 `USE_FULL_ASSERT` 决定：启用时调用 `assert_failed`，禁用时为空操作——这是由模板头文件直接定义的语义。
- **无以下机制**：未发现 C++ 异常、errno、结构化错误对象、panic/recover 或任何“中间件”风格的错误传播框架。
- **LL 层**：`stm32f1xx_ll_*.c` 也广泛使用 `assert_param`（例如 `stm32f1xx_ll_usb.c`、`stm32f1xx_ll_sdmmc.c`），但通常不暴露 `HAL_StatusTypeDef` 给应用，而是直接写寄存器，错误判断由调用者完成。

## 5. 总结

该仓库的错误处理遵循 ST HAL 的经典范式：以 `HAL_StatusTypeDef` 作为统一的同步错误信号，辅以可裁剪的 `assert_param` 运行时参数校验。设计目标是裸机嵌入式环境——简单、确定性、零运行时开销（当关闭断言时）。