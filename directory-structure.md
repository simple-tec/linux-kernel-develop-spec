# Linux 4.9.229 内核代码目录组织规范

本文档基于 `linux-4.9.229/` 实际源码结构，约束后续内核开发代码的放置位置、边界和接入方式。

## 1. 顶层目录职责边界

- `arch/`: 架构相关代码（启动、异常、中断、MMU、平台实现）。禁止在此引入通用业务逻辑。
- `include/`: 头文件。  
  - `include/linux/`: 内核内部通用接口。  
  - `include/uapi/`: 用户态可见 ABI，修改需严格兼容。
- `kernel/`: 调度、workqueue、time、locking、syscalls 等核心通用机制。
- `mm/`: 内存管理（页分配、回收、vm、slab、hugepage）。
- `fs/`: VFS 与文件系统实现。
- `net/`: 网络协议栈与网络核心框架。
- `drivers/`: 设备驱动，按总线/子系统分层组织。
- `security/`: LSM 与安全框架。
- `ipc/`: System V IPC/POSIX IPC 等。
- `lib/`: 可复用基础库，要求无设备/平台强绑定。
- `scripts/`: 构建、检查、代码生成工具（如 `checkpatch.pl`）。
- `tools/`: 用户态配套工具（perf 等），不放内核路径代码。
- `Documentation/`: 设计、接口与调试文档。

## 2. 新增代码放置规则

- 新功能优先放入已有子系统目录，避免新建顶层目录。
- 平台/SoC 特定实现放在 `arch/<arch>/` 或对应 `drivers/<subsystem>/` 的平台层。
- 通用算法与工具函数放在 `lib/` 或子系统 `core` 文件，不得散落到驱动目录。
- 头文件最小可见原则：  
  - 仅本子系统使用：放 `drivers/<subsystem>/` 私有头。  
  - 跨子系统内核内部接口：放 `include/linux/`。  
  - 用户态 ABI：仅放 `include/uapi/`。

## 3. Kconfig/Makefile 组织规范

- 每个新增功能必须有明确 Kconfig 项，名称与目录/模块一致。
- `Kconfig` 负责依赖约束（`depends on`/`select`），`Makefile` 只做对象编译拼装。
- 避免滥用 `select` 拉入大型依赖；优先 `depends on` 保持配置透明。
- 模块拆分遵循“核心 + 可选特性”模式，避免单文件过大。

## 4. 驱动目录组织建议（drivers）

- 推荐结构：
  - `core.c`: 核心状态机/公共流程
  - `probe.c` 或 `platform.c`/`pci.c`/`i2c.c`: 总线接入
  - `ops.c`: 硬件操作与回调实现
  - `debugfs.c`/`trace.c`: 调试可观测性
  - `Kconfig` + `Makefile`
- 设备树绑定文档放 `Documentation/devicetree/bindings/...`（若涉及 DT）。
- 禁止把寄存器位定义、魔法数散落在多个文件；统一在 `*_regs.h`/`*_hw.h`。

## 5. 跨子系统与分层约束

- 驱动层不得直接调用不稳定内部符号，优先使用子系统导出的公共 API。
- 通过回调、ops、notifier、tracepoint 建立扩展点，避免硬编码跨目录调用。
- 禁止循环依赖（A 依赖 B，同时 B 再依赖 A）。

## 6. UAPI 与兼容性

- 修改 `include/uapi/`、`ioctl`、`sysfs`、`procfs`、netlink 等用户态接口时：
  - 必须保持向后兼容；
  - 必须同步更新 `Documentation/`；
  - 必须补充兼容性说明与测试样例。

## 7. 文档与测试伴随规则

- 新增子系统能力同时提交：
  - 最少一份设计/接口说明（`Documentation/`）；
  - 最少一组可重复验证路径（Kconfig 组合 + 运行步骤）；
  - 如有工具或自测脚本，放到 `tools/` 或 `samples/`。

## 8. 禁止事项

- 不允许把架构私有代码放入通用目录（如 `kernel/`、`lib/`）。
- 不允许在驱动中复制已有通用框架能力（重复轮子）。
- 不允许在无 Kconfig 开关情况下引入难以关闭的调试/实验代码。

