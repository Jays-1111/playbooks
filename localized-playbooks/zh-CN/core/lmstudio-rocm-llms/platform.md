<!--
Copyright Advanced Micro Devices, Inc.

SPDX-License-Identifier: MIT
-->

# 平台配置

本文档描述运行该本地化 playbook 所需的目标平台配置。

## Windows

### 目标设备

该版本面向 AMD Ryzen™ AI Max+（设备 ID：`halo`）上的 Windows 环境。

### LM Studio 安装

LM Studio 不假设预安装。请按照 playbook 中的说明从 [LM Studio 下载页面](https://lmstudio.ai/download) 下载安装程序，并在安装后启动一次 LM Studio 以初始化命令行工具 `lms`。

| Component | Version | Location |
| --- | --- | --- |
| **LM Studio (Program)** | 用户安装版本 | `C:\Program Files\LM Studio` |
| **LM Studio (Models + CLI metadata)** | 用户安装版本 | `C:\Users\...\.lmstudio` |
| **LM Studio (Cache)** | 用户安装版本 | `C:\Users\...\AppData\Roaming\LM Studio` |

### 模型下载

该版本使用 LM Studio 下载并运行 `Qwen3.5-35B-A3B` 模型。

| Device | Model Type | Quantization | Approx. Size (GB) | Location |
| --- | --- | --- | --- | --- |
| AMD Ryzen™ AI Max+ | Qwen3.5-35B-A3B | `Q4_K_M` | 22.07 | LM Studio models directory |

### 内存配置

为了运行需要更高内存的大型模型，建议根据 playbook 中的步骤检查 AMD 可变显示卡内存（iGPU VRAM）配置。64 GB 对大多数工作负载来说已经足够，但在高上下文长度下运行更大的模型可能需要 96 GB。

---

## Linux

该本地化第一版暂不覆盖 Linux。
