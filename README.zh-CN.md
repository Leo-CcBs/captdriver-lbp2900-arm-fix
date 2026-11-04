# captdriver 修复版：解决 LBP2900 在 ARM64（aarch64）上打完一页就卡死

这是 **captdriver**（佳能 LBP2900 / LBP3000 / LBP3010 系列激光打印机的免费 CAPT 驱动）的补丁版本，修复了在 ARM64 系统上**打完第一页就永久卡住**的问题。

已在 UOS Desktop 20（aarch64）+ Canon LBP2900 上实测通过。

## 症状

- `lpstat -p` 一直显示 `printing` / `Processing page N...`，永远不结束
- 多页文档只打第一张，后面的永远排队
- 偶尔吐一张白纸，然后卡住
- 删除重装打印机能打出一次测试页，之后就打不出来了

## 根因

`captdriver 0.1.3` 源码里有好几处**没有超时的死等循环**。只要打印机回报一个意外状态（比如瞬时的"无纸"标志、延迟的状态更新），过滤器就永远不返回，CUPS 作业就永远停在"打印中"。

| 文件 | 函数 | 等什么 |
|---|---|---|
| `prn_lbp2900.c` | `lbp2900_page_epilogue` | `page_out == page_decoding` |
| `prn_lbp2900.c` | `lbp2900_job_epilogue` | `page_completed == page_decoding` |
| `prn_lbp2900.c` | `lbp2900_page_prologue` | `CAPT_FL_BUFFERFULL` 清除 |
| `prn_lbp2900.c` | `lbp2900_wait_user` | 面板按钮 |
| `capt-status.c` | `capt_wait_ready` / `capt_wait_xready` | `CAPT_FL_BUSY` 清除 |

另外 `rastertocapt.c` 对失败的页**没有重试上限**，同一页可能被反复重打。

## 修复内容

1. **加超时**：所有状态等待 120 秒超时，按钮等待 60 秒超时。超时后打印 `ERROR: CAPT: ... timeout` 并继续，而不是卡死。
2. **加重试上限**：每页最多重试 2 次，超过就跳过并报错（`CAPT_MAX_PAGE_RETRIES`），不再无限重发。
3. ARM64 编译小修：`_POSIX_C_SOURCE`、缺失的函数声明、一个 CAPT 常量。

完整改动见 `0001-fix-hang.patch`（249 行）。

## 编译

需要 CUPS 头文件（`libcups2-dev` / `cups-devel`）和 C 编译器。

```bash
cd src
gcc -std=c99 -O2 -c capt-command.c capt-status.c generic-ops.c \
    hiscoa-common.c hiscoa-compress.c paper.c printer.c \
    prn_lbp2900.c rastertocapt.c
gcc -o rastertocapt *.o -lcups -lcupsimage -lpthread
```

如果系统只有运行库没有 `libcups.so` 符号链接，这样链接：

```bash
gcc -o rastertocapt *.o -l:libcups.so.2 -l:libcupsimage.so.2 -lpthread
```

## 安装

```bash
sudo install -o root -g root -m 755 rastertocapt /usr/lib/cups/filter/rastertocapt
sudo systemctl restart cups

# 添加打印机（USB 地址按实际设备替换）
URI=$(lpinfo -v | awk '/^direct usb:\/\// && tolower($0) ~ /lbp2900/ {print $2; exit}')
sudo lpadmin -p Canon-LBP2900 -E -v "$URI" -P Canon-LBP2900.ppd
sudo lpadmin -d Canon-LBP2900
```

## 说明

- 打印机偶尔还会吐 1 张白纸 + 停 60 秒左右（打印机报瞬时"无纸/按钮"状态导致），但现在**作业会自己恢复**，不再卡死。如果频繁出现，检查搓纸轮。
- 不要和官方 `ccpd` 守护进程同时跑，它会抢占 USB 设备。
- 本驱动直接通过 `usb://` 讲 CAPT 协议，不需要 `ccpd`。

## 许可

GPLv3，见 `COPYING`。

基于 Alexey Galakhov 及贡献者的 **captdriver**，附加本仓库的卡死修复补丁。
