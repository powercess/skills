# QEMU 冒烟测试

没有板子的时候，ESP-IDF 自带的 QEMU 目标能跑起固件，验证"能启动、能初始化外设驱动、不会立刻 panic"。它**不能**验证真实外设时序、射频和屏幕。

## 构建

```bash
. "$IDF_PATH/export.sh"
idf.py --preview set-target esp32
idf.py -D SDKCONFIG=sdkconfig.qemu build
```

set-target 之后要把 sdkconfig.qemu 里的 flash 大小和分区表对齐到真实硬件，否则 QEMU 里能起、实机上起不来。

## 运行

```bash
timeout 120 idf.py qemu --qemu-extra-args="-nographic" 2>&1 | tee /tmp/qemu.log
grep -q "app_main" /tmp/qemu.log || exit 1
```

## 常见卡死点

- **整 120 秒没有任何串口输出**：多半是固件在等某个真实外设的响应（例如 BLE 控制器初始化），QEMU 没有对应模型，代码停在初始化的 while 里。把这类初始化挪到 QEMU 之外或加超时。
- **启动即 panic，Guru Meditation**：分区表与 flash 大小不一致。QEMU 按命令行给的 flash 布局跑，跟 partitions.csv 对不上。
- **能起但很快重启**：看门狗。QEMU 下时间推进比实机快，vTaskDelay 之间的业务耗时会被放大，把 WDT 超时调大或关掉再测。
- **测试永远"通过"**：检查退出码。idf.py qemu 不会因为固件 panic 返回非零，必须靠日志断言，所以 grep 那步不能省。
