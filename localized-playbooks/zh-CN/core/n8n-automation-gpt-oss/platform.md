<!--
Copyright Advanced Micro Devices, Inc.

SPDX-License-Identifier: MIT
-->

# 平台配置

本文档描述运行此 playbook 的预期平台配置。

## 前置条件

### Windows

| 组件 | 版本 | 说明 |
|------|------|------|
| **Node.js** | 22.22.1 LTS 或更新版本 | 用于安装并运行 n8n。 |
| **Lemonade Server** | 最新版本 | 在 `http://localhost:13305/api/v1` 提供 OpenAI 兼容 API。 |

## Lemonade LLM

Lemonade server 应加载适用于 AMD Ryzen AI Max+ 的模型。README 中提供了对应的 `lemonade run` 命令。

| 设备 | Endpoint | 模型 |
|------|----------|------|
| AMD Ryzen™ AI Max+ | `http://localhost:13305/api/v1` | `Qwen3.5-35B-A3B` |
