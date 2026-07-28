<!--
Copyright Advanced Micro Devices, Inc.

SPDX-License-Identifier: MIT
-->

<!-- @os:windows -->

<!-- @device:halo -->

在 Windows 上，为了运行需要更高内存的大型模型，我们需要使用 AMD 可变显示卡内存（iGPU VRAM）分配。虽然 64 GB 对大多数工作负载来说已经足够，但在高上下文长度下运行最大的模型可能需要 96 GB。

可以打开 **AMD Software: Adrenalin Edition™**，并导航到 **Performance → Tuning → AMD Variable Graphics Memory** 来完成此配置。请重启系统以使更改生效。

<!-- @device:end -->

<!-- @os:end -->
