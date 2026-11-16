---
name: esp32-idf-workflow
description: 在 ESP32/ESP32-S3 项目里建立可复现的 ESP-IDF 构建、烧写与 QEMU 回归流程：环境变量覆盖被写死的工具链路径、分区表与固件大小核验、串口烧写与串口日志抓取、CI 里用 QEMU 跑不需要硬件的冒烟测试。用于新项目起步、接手他人工程、或构建脚本写死了本机绝对路径的情况。
---

# ESP32 / ESP-IDF 工作流

ESP-IDF 的构建脚本一旦把 IDF_PATH、python 解释器、工具链路径写死成某台机器的绝对路径，换机器或换 CI 就跑不起来。这个技能的目标是：**任何一台机器 clone 下来，只用环境变量就能构建**，并且在没有板子的时候也能跑最基本的冒烟测试。

## 硬性约束

- 构建脚本不得写死 IDF_PATH 和 python 路径；从环境变量读，读不到再用 idf.py 所在的默认位置。
- 构建前必须检查 IDF_PATH 指向的目录真实存在、export.sh 可读，否则给一条能直接照做的人话报错，而不是让 idf.py 抛一堆 python traceback。
- 固件大小必须核验：构建完打印 app 分区占用与剩余，超过 90% 打警告。
- CI 里跑 QEMU 冒烟测试的，测试必须有超时和明确的失败退出码；卡住不动的测试等于没有测试。

## 实施流程

1. 读 CMakeLists.txt、sdkconfig.defaults、partitions.csv 和现有构建脚本，确认目标芯片、flash 大小、分区布局。
2. 改写构建入口，把绝对路径换成带默认值的环境变量写法，并在最前面加存在性检查。
3. 检查 sdkconfig.defaults 与 partitions.csv 是否一致——改了分区表却没改 flash 大小，是最常见的"能编译不能启动"来源。
4. 固化烧写与日志命令，目标名和波特率写进 README，不靠记忆。
5. 需要无硬件验证时，用 ESP-IDF 自带的 QEMU 目标构建 qemu 变体，写好超时与退出码，见 [QEMU 冒烟测试](references/qemu-smoke-test.md)。
6. 交付实际构建输出片段（含分区占用）与 QEMU 测试的实际通过输出，不只给命令列表。

## 环境变量约定

| 变量 | 用途 | 默认 |
| --- | --- | --- |
| IDF_PATH | ESP-IDF 根目录 | $HOME/esp/esp-idf |
| IDF_PYTHON_ENV_PATH | IDF 的 python 虚拟环境 | 由 export.sh 决定 |
| ESP_PORT | 烧写串口 | /dev/ttyUSB0 |
| ESP_BAUD | 烧写波特率 | 460800 |

## 参考

- [QEMU 冒烟测试](references/qemu-smoke-test.md)：无硬件时怎么跑、超时与退出码、常见卡死点。
