---
tags: [project, kerberos, nfs, linux, python, mongodb, interview]
status: 面试重点
created: 2026-07-17
updated: 2026-07-17
---

# EDS NFSv4 Kerberos 域接入与用户映射管控

> 给 EDS 文件存储接入企业 Kerberos KDC，让 NFSv4 共享支持 `sec=krb5` 身份认证。这是一套从域管理到数据面认证生效的端到端能力，不是简单的“加域接口”。

## 解决什么问题

原有 NFS 共享只使用 `sec=sys`，依赖客户端提交 UID/GID，无法对接企业 Kerberos 认证体系。

本项目为管理员提供 Kerberos 域生命周期管理，并将 KDC、服务凭据、用户映射和 NFS-Ganesha 配置同步到全节点，使 NFS 客户端能够使用 Kerberos 票据以 `sec=krb5` 挂载共享。

## 端到端认证链路

```text
NFS 客户端携带 Kerberos 票据，以 sec=krb5 挂载
  -> NFS-Ganesha 使用服务端 keytab 验证身份
  -> principal_map.conf 将 user@REALM 映射为 UNIX 用户
  -> PHXDFS 按对应 UNIX 身份执行文件访问
```

这里的关键是让 Kerberos Principal 最终落到存储系统能识别的 UNIX 身份，而不是只完成票据校验。

## 核心能力

| 模块 | 实现内容 |
| --- | --- |
| 域管理 | 加域、退域、编辑、查询、测试连接 |
| KDC 对接 | 管理 Realm、KDC 地址和 88/749 端口 |
| 服务身份 | 创建、复用或覆盖 NFS 服务主体 SPN |
| 服务凭据 | 全节点通过 `ktadd` 生成 keytab，并使用 `kinit` 验证凭据 |
| 配置分发 | 同步 `krb5.conf`、keytab、`principal_map.conf`、`ganesha.conf` |
| NFS 联动 | 共享支持 `sys`、`krb5` 两种安全模式 |
| 用户映射 | 将 `user@REALM` 等 Principal 映射为 UNIX 用户 |
| 退域清理 | 共享自动回退 `sec=sys`，清理映射、keytab 和域配置 |

## 架构与关键实现

管控面负责保存域配置、编排异步任务和分发节点配置；NFS-Ganesha 负责认证生效；PHXDFS 最终按映射后的 UNIX 身份执行访问控制。

```text
管理界面 / API
  -> Kerberos 域配置与任务编排
  -> SPN 与 keytab 生成、凭据验证
  -> 全节点配置分发
  -> NFS-Ganesha 刷新配置
  -> Principal 到 UNIX 用户映射生效
```

主要实现位于：

- 需求与设计：`docs/requirements/EDS5.3.1/02_NFSv4_Kerberos认证/01_需求文档.md`
- 核心编排：`cluster_manager/module/storage/filestore/nss_domain/kerberos/manager/kerberos_mgr.py`

## 可靠性与运维

- 使用异步任务和进度展示承载加域、退域等长链路操作。
- 对失败步骤进行回滚，避免 SPN、keytab、共享安全模式和节点配置处于半完成状态。
- 加域和退域使用互斥锁，避免并发任务相互覆盖配置。
- 增加 KDC 健康检测和连接异常告警，提前暴露认证依赖不可用问题。
- 集群扩容、主机替换时自动同步 Kerberos 配置，保证新节点具备认证能力。
- 处理升级迁移，以及 AD 域与独立 Kerberos 域共存场景。

## 难点与取舍

难点不在 Kerberos 协议实现，而在跨组件状态收敛：KDC 服务主体、全节点 keytab、Ganesha 配置、Principal 映射和共享安全模式必须一起生效或被恢复。

退域时不能只删配置：还要把共享安全模式降回 `sec=sys`，并清理映射和凭据，否则客户端访问可能落到不可用的认证状态。

## 交付结果

- 2026 年 3 月至 6 月主导完成需求设计、主体功能和后续问题收口。
- 覆盖扩容、主机替换、Ganesha 刷新、退域卡住、健康检测误报和并发互斥等场景。
- 补充 136 个单元测试，提交记录标注覆盖率 81% 以上。

## 简历表述

> 主导 EDS NFSv4 Kerberos 认证功能设计与实现，完成 KDC 域生命周期、SPN/keytab、多节点配置分发、Principal 用户映射、NFS-Ganesha 认证联动、健康告警及扩容替换适配；支持 NFS 共享以 `sec=krb5` 完成企业 Kerberos 身份认证，并在退域时自动回退 `sec=sys`。

## 面试：30 秒介绍

> 我负责的是 EDS 文件存储的 NFSv4 Kerberos 认证能力。管理员配置 Realm、KDC 和 SPN 策略后，系统会在全节点生成并校验 keytab，同步 `krb5.conf`、Ganesha 配置和 Principal 映射。客户端以 `sec=krb5` 挂载时，Ganesha 用 keytab 校验票据，再将 `user@REALM` 映射为 UNIX 用户，PHXDFS 按该身份做权限校验。项目难点是跨 KDC、节点配置和数据面的状态一致性，因此我们用异步任务、回滚、加退域互斥和健康告警保证整个链路可控。

## 高频追问

### 1. 为什么还要做 Principal 到 UNIX 用户的映射？

Kerberos 解决“你是谁”，而存储访问控制最终依赖 UNIX 用户身份。`principal_map.conf` 将 `user@REALM` 映射为本地 UNIX 用户，PHXDFS 才能按现有权限模型完成授权。

### 2. SPN、keytab 和 `kinit` 分别做什么？

SPN 标识 NFS 服务身份；KDC 通过 `ktadd` 为该服务主体生成 keytab；节点用 `kinit` 验证 keytab 能否获得服务凭据。三者共同保证 Ganesha 能验证客户端票据。

### 3. 为什么 keytab 要全节点生成和验证？

集群中任一节点都可能承接 NFS 请求。只在单节点配置会造成故障切换、扩容或流量调度后认证失败，因此必须保证所有服务节点拥有可用且一致的服务凭据。

### 4. 加域或退域失败时如何处理？

操作作为异步任务分阶段执行，并记录进度。某一步失败时回滚已生效的配置；退域尤其要恢复共享到 `sec=sys`，再清理映射、keytab 和域配置，避免残留无效认证状态。

### 5. 为什么加域和退域需要互斥？

二者会同时修改 Realm、keytab、用户映射和 Ganesha 配置。并发执行会造成配置覆盖或一边创建、一边删除凭据，因此用互斥锁保证同一时刻只有一个域变更任务执行。

### 6. 如何处理集群节点变化？

扩容和主机替换纳入集群生命周期：新节点自动同步 Kerberos 配置并完成凭据可用性校验，避免节点加入后出现“服务可用但 Kerberos 认证失败”。

## 关联知识

- [[项目知识地图]]
- [[Linux知识地图]]
