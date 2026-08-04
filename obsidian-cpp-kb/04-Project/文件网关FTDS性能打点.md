---
tags: [project, ftds, samba, nfs-ganesha, performance, observability, cpp, interview]
status: 持续核验
created: 2026-07-29
updated: 2026-07-29
---

# 文件网关FTDS性能打点

## 项目定位

FTDS 是 EDS 数据面的低开销性能观测基础设施。它把高频 IO 路径的调用次数、成功/失败、时延、IO 大小和在途情况聚合到内存中，按模块、操作和 IO 大小分桶导出，供现场排障和监控采集。

本项目的个人主线是：**基于既有 FTDS 框架，扩展文件网关 Samba 的协议层性能打点，并参与 NFS-Ganesha 的 FTDS 接入。** FTDS 通用框架、统一 DUMP/iostat 和在线命令属于既有能力；个人贡献的准确范围需以提交记录继续补证。

## 要解决什么问题

文件网关同时承载 SMB、NFS 等高频访问路径。出现“读写慢、失败率升高、吞吐下降”时，仅靠日志、抓包和复现很难判断瓶颈位置。FTDS 让系统能够在线回答：

- 慢发生在协议收包、OP 处理、发包，还是底层 FSAL/存储调用？
- 不同 IO 大小的时延、错误和吞吐是否不同？
- 是整体变慢，还是少量长尾请求拉高均值？
- 某个点位是否有 START 未对应 END 的在途请求？

> [!warning] 指标边界
> 当前材料只确认计数、总时延、min/max 等聚合字段，未确认时延直方图或分位数逻辑。因此不能声称 FTDS 能直接输出 P99；平均值也不能替代 P99。

## FTDS 框架机制

### 数据模型

一个 `(module, op_index)` 点位背后维护按 IO 大小分桶的 `FTDS_CNT` 数组。已知桶包含 TOTAL、0B、4K、8K、16K、32K、64K、128K、256K、512K、1M、`>1M`；具体边界由 `get_io_size_index()` 决定，后续需要以源码/测试确认精确区间。

每个统计单元记录调用/返回计数、成功/失败、累计时延、累计字节、min/max 时延和 min/max IO 大小。DUMP 可据此计算当前周期平均时延、平均 IO 大小和在途数量；iostat 维度再将多个 counter 按读写类型聚合为监控文件。

```text
PHX_MODULE
  -> op_index
    -> FTDS_CNT[IO_SIZE_BUCKET]
      -> count / ok / err / totalTime / totalSize / min-max
```

### 三种打点形态

- `START + END`：在开始/结束处显式配对，适合同步包裹或跨函数生命周期。
- `START_WITH_ID + END_WITH_ID`：带业务 ID 的异步配对。TOTAL 桶保留有限元数据槽位，记录未返回样本供 `SHOW_NOT_BACK` 诊断；它不是全量在途请求表。
- `RECORD`：业务方已算好时延时一次提交，语义等价于完整 START/END 的聚合结果。

`used_time` 由调用方计算并传入 END/RECORD；框架中的 `start_us` 主要用于未返回请求诊断，不应误说成框架用它计算对外时延。

### 并发与开销取舍

高频累加采用 `std::atomic` 与 `memory_order_relaxed`，优先降低热路径同步开销。`min/max` 的并发更新允许极小概率丢失，代码以统计近似性换取吞吐；因此导出的瞬时 min/max 不应被当作严格事务一致的值。

FTDS 支持在线启停、清理、详情/总量/未返回查看和 iostat 导出。它必须可控，因为即使是原子累加、取时和分桶，在 SMB/NFS 热路径上也会带来 CPU 与缓存竞争。关闭时是否跳过取时、分桶和原子更新，以及 `DISABLE` 与 `CLEAN` 的精确语义，仍需以源码补证。

## Samba 接入与重构（已确认材料）

### 覆盖范围

Samba 从原有 19 个 SMB2 OP 统计扩展为 21 个点位：19 个 OP 加上收包 `RECV` 与发包 `SEND`。目标是将协议 OP 处理和 socket 收发包时延放到统一 FTDS 视图中，辅助区分协议层路径和后续底层路径的耗时。

### 枚举空间冲突

Samba 自有 `FTDS_SMB2_OP_*` 枚举与公共 `phxinf_iolog_smb.h` 中的 SMB IO 枚举共用 `(module, index)` 注册空间。两套枚举都从低位连续取值时，后注册点位会覆盖先注册点位对应的统计指针，导致数据丢失或 DUMP 显示错位。

改动引入 `FTDS_SMB2_OP_OFFSET = 5`，为公共枚举让出 `0~4`，并将 `RECV/SEND` 放在 Samba OP 枚举末尾。它本质上是跨组件复用注册命名空间时的索引隔离问题，而不是单纯改一个数字。

### 集中生命周期打点

早期版本在 dispatch `switch` 的各个 OP 分支中散落 START/END 模板代码。重构后：

- START 集中到 dispatch 入口，以 opcode 加 offset 映射到点位。
- END 集中到既有的 `smbd_smb2_op_statistics_tick` 统计出口。
- 起始时间保存在请求对象 `req->ftds_start_time_us` 中，跨函数传递；结束后清零，避免重复打点。
- OP 路径复用 dispatch 早期已记录的 `req->start_time_us`，减少重复读取单调时钟。

这属于在既有框架上的 Samba 适配层扩展和重构：将大量重复模板收束到统一入口/出口，并用请求对象承载跨函数生命周期状态。

### 收发包与命名修正

SEND 路径的 `io_size` 通过累加队首 entry 的全部 `iov_len` 得出，而不是仅取单个返回值，使 IO 大小能正确进入 FTDS 的分桶模型。

同时修正 `NOTIFY` 与 `QUERY_DIRECTORY` 使用同一 counter name 的隐藏冲突，并统一 counter 命名为 `smb2_*`，避免两个不同 OP 在导出视图中混淆。

## NFS-Ganesha 的观测边界

NFS 侧 FTDS 当前仅覆盖 FSAL 层 `nfs_readv/nfs_writev` 两个点位，采用 START/END 显式配对。NFS 协议层已有独立 bvar 体系（`NFS_BVARS_*`、`op_ctx->nfs_latency/fsal_latency`）采集协议和 FSAL 时延；FTDS 在 NFS 的作用是补充底层 IO 大小分桶与统一导出。

因此，SMB 与 NFS 可以在同一份 FTDS 报表中分别观察，但 **不能直接比较** `nfs_readv` 与 `smb2_read` 的时延：前者是 FSAL 到 phxdfs 的片段，后者覆盖更长的 SMB 协议处理路径。输出格式一致不代表观测边界一致。

用户说明曾提交 NFS FTDS 相关代码，但具体提交、点位注册、测试和个人归属尚未核验；补证前不把 NFS 具体实现写入个人成果。

## 已确认风险与复盘

### 当前或历史风险

- Samba v2 发包路径曾在循环外缓存 `req`，队首 entry 释放后下轮继续解引用，形成 UAF 风险。该问题后续由同事提交 `9e95723e8d` 修正；不能表述为本人发现并修复。
- 材料指出 `smbd_smb2_op_statistics_tick` 可能将初始化为 `0` 的 `delay_time` 直接传给 FTDS END，导致 OP 时延统计无效。该问题是否仍在当前 HEAD、是否进入生产路径，必须优先源码核验；在修复或证实前，不得宣称 SMB OP 时延统计准确可用。
- NFS 注册的 `idx_count=10` 若为硬编码且与枚举最大值无机械联动，新增点位时可能触发断言或越界；需确认当前代码和修复状态。
- 两端均使用 `CLOCK_MONOTONIC`，不存在时钟源不统一；NFS 侧更值得补的是单调时钟异常回退的防御性处理。

### 后续改进方向（非已交付）

1. 修复并验证 Samba OP `delay_time` 的计算与异常路径。
2. 将 NFS 注册数量从魔数改为由枚举推导，并补对应回归。
3. 明确 bvar 与 FTDS 的职责，必要时统一展示语义而非重复采集。
4. 为启停、清理、在途槽位溢出、异常提前返回和并发 min/max 误差补测试与运维说明。

## 验证要求

当前尚未拿到可靠的性能压测或现场开销数据，不能量化“低损耗”或“性能提升”。完整验证至少包括：

1. 启用模块后触发 SMB 收包、发包和 OP，确认 DUMP/iostat 出现对应 counter、次数、错误、时延和 IO 桶。
2. 以可控请求验证 `io_size` 分桶、成功/失败、START/END 在途数与 `SHOW_NOT_BACK` 语义。
3. 关闭模块后验证业务语义不变，并测量/检查是否跳过热路径的主要打点开销。
4. 对异常、提前返回、超时和高并发路径验证 END 是否遗漏及统计是否可接受。
5. NFS 单独验证 `nfs_readv/writev` 的 FSAL 统计，避免把 bvar 的协议层时延与 FTDS 片段混为同一指标。

## 面试表达

### 1 分钟介绍

我在文件网关性能可观测性项目中，基于 EDS 既有 FTDS 框架扩展 Samba 的性能打点。FTDS 用低开销原子聚合持续记录模块、操作和 IO 大小维度的次数、时延、错误和在途情况，便于现场在线排障。

我的工作重点是补齐 Samba 的 SMB2 OP、收包和发包点位，并解决 Samba 枚举与公共 SMB IO 枚举共用注册空间时的冲突；同时将分散在多个 OP 分支的打点收束到 dispatch 入口和统一统计出口，用请求对象保存跨函数起始时刻。这样既减少了重复代码，也让 Samba 的协议层性能数据能接入统一导出视图。

我会明确区分：FTDS 通用框架是既有能力，我负责的是 Samba/NFS 的接入和适配；SMB 与 NFS 的 FTDS 点位层次不同，不能直接横向比较时延。

### 简历表述

基于 EDS FTDS 性能观测框架扩展文件网关 Samba/NFS 打点能力：完成 SMB2 OP、收发包点位接入与跨函数生命周期重构，解决跨组件注册枚举冲突，接入 IO 大小分桶、时延、错误和在途统计；明确 SMB/NFS 指标观测边界，提升文件网关在线性能排障能力。

> [!warning] 简历使用前
> 需补齐 NFS 个人提交证据，并核验 Samba OP `delay_time` 问题。未确认前，将简历表述收敛为“完成 Samba 接入与重构，参与 NFS 接入”，不要承诺所有 OP 指标准确上线。

## 高频追问

1. 为什么性能打点必须支持开关？关闭后到底还有哪些开销？
2. `memory_order_relaxed` 为什么适合统计？min/max 非原子会带来什么误差？
3. 为什么 TOTAL/IO 桶、平均值和 min/max 不能直接推出 P99？
4. START/END、带 ID 的异步配对和 RECORD 分别适合什么场景？8 槽位能说明什么、不能说明什么？
5. 为什么 SMB 与 NFS 使用同一框架仍不能直接比较 `read` 时延？
6. 枚举 index 冲突为何会导致统计覆盖？如何从设计上避免？

## 关联知识

- [[项目知识地图]]
- [[Linux知识地图]]
- [[内存管理]]
- [[C++11新特性总览]]
