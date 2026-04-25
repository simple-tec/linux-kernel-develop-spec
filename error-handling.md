# Linux 4.9.229 错误处理规范

本文档基于 `include/linux/err.h` 与 4.9.229 中常见 `goto err_*` 清理路径实践。

## 1. 返回值统一约定

- 成功返回 `0`（或有效对象/正值，按接口定义）。
- 失败返回负 errno（`-EINVAL`、`-ENOMEM`、`-ETIMEDOUT` 等）。
- 不得返回正 errno，也不得吞掉底层关键错误码。

## 2. 指针错误编码（必须）

- 可能失败且返回指针的接口，使用 `ERR_PTR(error)` 传递错误。
- 调用方使用 `IS_ERR(ptr)` 检查，并用 `PTR_ERR(ptr)` 转为 errno。
- 允许空指针语义时，使用 `IS_ERR_OR_NULL(ptr)`。
- 可直接返回整型错误时优先 `PTR_ERR_OR_ZERO(ptr)`。

## 3. 失败路径组织（goto 模式）

- 多资源初始化函数必须使用分层标签回滚，禁止复制粘贴回收代码。
- 标签命名按动作：`err_free_x`、`err_unmap`、`err_put_clk`、`out_unlock`。
- 回滚顺序必须与申请顺序相反（LIFO）。
- 标签块只做清理，不夹杂新的业务逻辑。

推荐模式：

```c
int foo_init(...)
{
	int ret;

	res1 = alloc1();
	if (!res1)
		return -ENOMEM;

	ret = init2();
	if (ret)
		goto err_free_res1;

	ret = init3();
	if (ret)
		goto err_deinit2;

	return 0;

err_deinit2:
	deinit2();
err_free_res1:
	free1(res1);
	return ret;
}
```

## 4. 错误码选择原则

- 参数非法：`-EINVAL`。
- 资源不足：`-ENOMEM`、`-ENOSPC`。
- 设备/对象不存在：`-ENODEV`、`-ENOENT`。
- 状态不允许：`-EBUSY`、`-EAGAIN`、`-EPERM`。
- 超时：`-ETIMEDOUT`；中断睡眠：`-ERESTARTSYS` 或上抛原始错误。

规则：优先返回“最接近根因”的错误码，不要统一 `-EIO` 掩盖问题。

## 5. probe/remove 与生命周期错误处理

- `probe()` 失败后必须保证设备处于可安全重试状态。
- 使用 `devm_*` 时仍需关注逻辑资源（状态位、线程、workqueue）回收时机。
- `remove()`/`shutdown()` 不应返回失败来“掩盖”清理问题，必须尽最大努力释放资源。

## 6. 锁与错误路径

- 任意错误返回前必须保证锁平衡（`mutex_unlock/spin_unlock`）。
- 推荐 `out_unlock` 标签统一解锁，避免多分支漏解锁。
- 原子上下文出错路径禁止引入睡眠释放动作。

## 7. 日志与错误分离

- 错误码用于机器判断，日志用于人类定位，二者都要有但避免重复刷屏。
- 由上层统一处理的错误，底层不应层层 `pr_err` 打印同一故障。
- 对预期可恢复错误，优先 `dev_dbg/dev_warn`，避免把正常重试当故障。

## 8. 典型反例（禁止）

- 失败后继续使用未初始化资源。
- 返回 `-1`、`1` 等非标准 errno 风格值。
- 漏掉某一分支的 `kfree/put_device/clk_put`。
- 在错误标签内再次分配资源或修改关键状态导致二次失败。

