# HIBMC DRM 驱动代码总结

> 路径：`drivers/gpu/drm/hisilicon/hibmc`

## 概述

HIBMC DRM 驱动为 HiSilicon HIBMC 设备提供基础显示能力，采用 DRM Atomic/KMS 框架，核心特点是：

- 基于 VRAM 的 GEM 显存管理
- 单 CRTC + 单主平面（primary plane）显示管线
- VGA(DAC) 输出（支持 DDC/EDID）
- DisplayPort 输出（AUX、HPD 中断、链路训练、SerDes 配置）

该实现面向 BMC/服务器场景，重点是稳定可用和可调试性。

## 目录与模块职责

| 文件 | 作用 |
| --- | --- |
| `hibmc_drm_drv.c` | PCI/DRM 入口：probe/remove、硬件初始化、KMS 初始化、IRQ/MSI、PM |
| `hibmc_drm_de.c` | 显示引擎（DE）：plane/CRTC、模式设置、vblank、gamma LUT |
| `hibmc_drm_vdac.c` | VGA 输出：connector/encoder、EDID 探测与连接状态 |
| `hibmc_drm_i2c.c` | GPIO bit-bang I2C（供 VGA DDC） |
| `hibmc_drm_dp.c` | DP 的 DRM glue：connector/encoder、HPD threaded IRQ |
| `hibmc_drm_debugfs.c` | DP debugfs：链路状态展示和 colorbar 控制 |
| `dp/dp_aux.c` | AUX 事务实现（native + I2C-over-AUX） |
| `dp/dp_link.c` | DP 链路训练状态机（CR/EQ + 失败降级） |
| `dp/dp_hw.c` | DP 硬件初始化、时序/视频流配置、显示开关 |
| `dp/dp_serdes.c` | SerDes 速率切换、TX swing/pre-emphasis 配置 |
| `hibmc_drm_regs.h` | 主显示相关寄存器定义（电源门控/PLL/中断/CRTC） |

## 初始化与运行流程

1. `probe` 阶段分配 DRM 设备并使能 PCI 功能。
2. 映射 MMIO（BAR1），完成电源门控与 local memory reset。
3. 初始化 VRAM helper（BAR0）。
4. 初始化 KMS 对象：
   - DE（primary plane + CRTC）
   - DP（若硬件存在）
   - VGA/VDAC
5. 初始化 vblank 与 MSI 中断。
6. 注册 DRM 设备并启用 fbdev generic。

## KMS/显示管线能力

- 不支持平面缩放（`src_w/h` 必须等于 `crtc_w/h`）
- 不允许负的 CRTC 坐标
- pitch 必须 128 字节对齐
- CRTC mode 校验限制：
  - 刷新率约束为 59~61Hz
  - 分辨率需要命中内置 PLL 表
- 支持 gamma LUT（256 项）

## 输出路径

### VGA (VDAC)

- 通过 GPIO bit-bang I2C 读取 EDID。
- 若 EDID 读取失败，回退 no-EDID 模式，默认偏好 `1024x768`。

### DisplayPort

- AUX 负责 DPCD/EDID 事务。
- HPD 插拔通过 threaded IRQ 处理并上报热插拔事件。
- 链路训练包含：
  - Clock Recovery（CR）
  - Channel Equalization（EQ）
  - 按 sink 能力选择 TPS2/TPS3/TPS4
- 训练失败会执行降级策略（降速率和/或降 lane）。
- SerDes 按训练请求配置电压摆幅和预加重参数。

## 中断模型

- IRQ 向量 0：vblank 中断处理
- IRQ 向量 1：DP HPD 中断（top-half + threaded handler）

## 调试与可观测性

`hibmc_drm_debugfs.c` 提供：

- DP 当前 lane 数、link rate、vrefresh、DPCD 版本、HPD 状态
- `colorbar-cfg` 写接口，用于链路/显示调试

## 配置项

- Kconfig：`CONFIG_DRM_HISI_HIBMC`
- 模块名：`hibmc-drm`
- 依赖：DRM、PCI、MMU、I2C 及相关 DRM helper 组件

