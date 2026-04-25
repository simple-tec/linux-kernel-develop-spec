# Linux 4.9.229 日志与调试规范

本文档基于 `include/linux/printk.h`、`include/linux/device.h`、`Documentation/printk-formats.txt`、`Documentation/dynamic-debug-howto.txt`。

## 1. 日志 API 选型

- 通用内核代码：使用 `pr_*`（`pr_err/pr_warn/pr_info/pr_debug`）。
- 驱动代码：优先 `dev_*`（自动携带设备上下文）。
- 网络设备路径：优先 `netdev_*` / `netif_*`。
- 不建议直接裸用 `printk`，除非必须控制前缀或极早期场景。

## 2. 级别使用规范

- `pr_emerg/pr_alert/pr_crit`: 系统生存级故障，极少使用。
- `pr_err/dev_err`: 功能失败、需关注的问题。
- `pr_warn/dev_warn`: 异常但可继续运行。
- `pr_notice/pr_info`: 关键状态变化、初始化结果。
- `pr_debug/dev_dbg`: 调试信息，默认可关闭。

规则：日志级别要反映“处置优先级”，不是“代码分支重要性”。

## 3. 格式与内容规范

- 日志文本必须可检索、可定位：建议包含对象 ID、寄存器值、状态码。
- 不拆分一条用户可见消息为多行拼接字符串。
- 使用内核格式规范：地址和类型按 `printk-formats.txt`。
- 指针打印遵循 `%p` 扩展语义，敏感内核指针按 `%pK` 策略处理。
- 不打印密钥、明文凭据、未脱敏用户数据。

## 4. 频率控制与去重

- 高频路径使用 `*_ratelimited`（如 `pr_err_ratelimited`、`dev_warn_ratelimited`）。
- 单次提示使用 `*_once`（如 `dev_warn_once`）避免刷屏。
- 同一错误不要在调用栈多层重复打印；确定“唯一责任层”。

## 5. 调试开关与动态调试

- 在模块文件中定义 `pr_fmt(fmt)` 统一前缀（模块名/函数域）。
- 调试信息优先走 `pr_debug/dev_dbg`，结合 `CONFIG_DYNAMIC_DEBUG` 动态启停。
- 动态调试控制文件：`<debugfs>/dynamic_debug/control`。
- 建议按文件/函数精准开启，避免全局打开导致噪声。

示例：

```bash
echo 'file drivers/foo/bar.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'func foo_probe +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module foo_driver +p' > /sys/kernel/debug/dynamic_debug/control
```

## 6. 错误日志编写建议

- 打印失败动作 + 关键参数 + 错误码。
- 建议模板：`"xxx failed: id=%d ret=%d\n"`。
- 对可恢复场景先 `debug/warn`，不可恢复场景再 `err/crit`。
- panic 路径日志要简短确定，避免二次故障。

## 7. 上下文与时序注意事项

- 中断或原子上下文日志要克制，避免长字符串和高频输出。
- 持锁区域内日志最小化，避免影响时序和锁竞争。
- 大块数据转储优先 `print_hex_dump_debug()`，并配合动态调试开关。

## 8. 调试实践流程（建议）

1. 先用 `dmesg -T | grep <module>` 观察主故障点。
2. 定点打开 dynamic debug 到目标文件/函数。
3. 若日志过多，启用 `*_ratelimited` 或缩小匹配范围。
4. 问题定位后关闭动态调试，提交前清理临时日志。

