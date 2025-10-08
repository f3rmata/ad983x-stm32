# AD983X 驱动说明与使用指南

本仓库提供 AD983X（AD9833/AD9834 等）函数发生器芯片的简洁 STM32 HAL 驱动，支持通过 SPI 配置输出波形、频率、相位以及睡眠/符号输出等特性。

- 语言/平台：C，依赖 STM32 HAL（`main.h`）
- 设备支持：AD9833 / AD9834（同系列寄存器兼容）
- 接口：SPI + 两个 GPIO（CS、RESET）

## 文件结构

- `ad983x.h`：类型、枚举与对外 API 声明
- `ad983x.c`：寄存器定义与 API 实现
- `LICENSE`：开源许可
- `README.md`：使用文档

## 快速开始

1) 硬件连接（以 AD9833 为例）：
- VDD、DGND/AGND 按芯片规格连接
- SPI 接口：SCLK、SDATA 连接到 MCU 的 SPI 时钟/数据引脚
- FSYNC（片选）连接到任意 GPIO（作为 CS）
- RESET 引脚连接到任意 GPIO（作为复位）
- 输出：`OUT`（正弦/三角），`COMP`（方波/比较器输出，视配置而定）

2) HAL 外设初始化：
- 配置 `SPI_HandleTypeDef`（模式 0，MSB first，合适的波特率）
- 将 CS、RESET 配置为推挽输出，默认拉高

3) 代码初始化与基本使用：

```c
#include "ad983x.h"

// 假设这些来自 CubeMX 生成的 HAL 代码
extern SPI_HandleTypeDef hspi1;

// 片选与复位引脚（示例）
#define AD983X_CS_GPIO_Port   GPIOA
#define AD983X_CS_Pin         GPIO_PIN_4
#define AD983X_RESET_GPIO_Port GPIOA
#define AD983X_RESET_Pin      GPIO_PIN_5

void app_init_ad983x(void) {
    AD983X dev;

    // clk_mhz：输入时钟频率（MHz），AD9833 典型为 25MHz
    AD983X_init(&dev,
                &hspi1,
                AD983X_CS_GPIO_Port, AD983X_CS_Pin,
                AD983X_RESET_GPIO_Port, AD983X_RESET_Pin,
                25 /* clk_mhz */);

    // 选择输出波形：0=正弦，1=三角，2=方波
    AD983X_setOutputWave(&dev, 0); // 正弦波

    // 设置频率（Hz），选择频率寄存器 0/1
    AD983X_setFrequency(&dev, 0, 1000.0); // 1 kHz

    // 可选：设置相位（0..4095，12-bit）
    AD983X_setPhaseWord(&dev, 0, 0);
}
```

4) 频率计算说明：
- 频率字计算：`freq_word = frequency * (2^28 / f_clk)`，其中 `f_clk` 为输入参考时钟（Hz）
- 本驱动在 `AD983X_init()` 中预先计算了缩放因子：`m_clk_scaler = (2^28) / (clk_mhz * 1e6)`
- `AD983X_setFrequency()` 内部会将期望频率转换为频率字并写入 FREQ0/FREQ1 寄存器的低 14 位与高 14 位两次（按照芯片要求的分两次写入）

## API 概览

以下为主要 API 与用途（详见 `ad983x.h`）：

- 初始化与基础读写
  - `void AD983X_init(AD983X *self, SPI_HandleTypeDef *hspi, GPIO_TypeDef *select_port, uint16_t select_pin, GPIO_TypeDef *reset_port, uint16_t reset_pin, uint8_t clk_mhz);`
  - `void AD983X_writeReg(AD983X *self, uint16_t value);`（内部使用，用于写控制字）

- 波形/输出
  - `void AD983X_setOutputWave(AD983X *self, uint8_t mode);`  
    模式：`0=正弦`，`1=三角`，`2=方波`
  - `void AD983X_setOutputMode(AD983X *self, OutputMode out);`  
    `OUTPUT_MODE_SINE` / `OUTPUT_MODE_TRIANGLE`
  - `void AD983X_setSignOutput(AD983X *self, SignOutput out);`  
    选择符号输出：MSB/比较器等（影响 OUT/COMP 输出行为）

- 频率/相位
  - `void AD983X_setFrequency(AD983X *self, uint8_t reg, double frequency);`  
    设置频率（Hz），`reg=0/1` 选择 FREQ0/FREQ1
  - `void AD983X_setFrequencyWord(AD983X *self, uint8_t reg, double frequency);`  
    直接写频率字（已分片写入）
  - `void AD983X_setPhaseWord(AD983X *self, uint8_t reg, uint32_t phase);`  
    设置相位字（12-bit），`reg=0/1` 选择 PHASE0/PHASE1

- 低功耗/复位
  - `void AD983X_setSleep(AD983X *self, SleepMode out);`  
    选择睡眠模式：`SLEEP_MODE_MCLK` / `SLEEP_MODE_DAC` / `SLEEP_MODE_ALL`
  - `void AD983X_reset(AD983X *self);`  
    通过硬件 RESET 引脚执行短延时复位

### 枚举值说明（节选）

- `OutputMode`：
  - `OUTPUT_MODE_SINE` 正弦
  - `OUTPUT_MODE_TRIANGLE` 三角（寄存器中置位 `REG_MODE`）
- `SignOutput`（影响比较器/方波等）：
  - `SIGN_OUTPUT_MSB`
  - `SIGN_OUTPUT_MSB_2`
  - `SIGN_OUTPUT_COMPARATOR`
- `SleepMode`：
  - `SLEEP_MODE_MCLK` 关闭 MCLK
  - `SLEEP_MODE_DAC` 关闭 DAC
  - `SLEEP_MODE_ALL` 全部睡眠

## 使用要点与注意事项

- SPI 时序：模式 0，MSB first，`FSYNC`（CS）为低选中；每次写入 16-bit 控制字。
- 初始化会拉高 CS，执行硬件复位并写入默认控制字 `0x2100`（B28 连续写入、RESET 生效）。
- 频率设置流程中，驱动会临时进入连续写模式（`0x2100`），写完后恢复（`0x2000`）。
- 方波输出源于 `SignOutput` 与比较器配置；若需稳定方波，请选择比较器输出并注意硬件负载。
- `clk_mhz` 必须与板上参考时钟匹配（如 AD9833 的 25MHz），否则频率计算会偏差。
- `phase` 为 12-bit，范围 `0..0x0FFF`，对应 0..360° 线性映射。

## 示例：在运行时切换频率与波形

```c
void sweep_example(AD983X *dev) {
    AD983X_setOutputWave(dev, 0); // 正弦
    for (double f = 100.0; f <= 10000.0; f += 100.0) {
        AD983X_setFrequency(dev, 0, f);
        HAL_Delay(10);
    }
    AD983X_setOutputWave(dev, 1); // 三角
    AD983X_setPhaseWord(dev, 0, 0x0800); // 加 180° 相位偏置
}
```

## 常见问题

- 输出频率不准确：
  - 检查 `clk_mhz` 参数与实际晶振是否一致
  - SPI 写入是否为 16-bit 控制字，是否正确的字节序
- 无输出或输出畸变：
  - RESET 引脚是否连接正确并能拉高/拉低
  - 输出端负载是否合适（建议高阻/合适缓冲）
- 方波输出不如预期：
  - 比较器输出需正确配置 `SignOutput`，并确保 COMP 引脚连接到期望的负载

## 许可

本项目遵循 `LICENSE` 文件所示的开源许可。