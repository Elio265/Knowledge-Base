---
tags: [project, kerberos, nfs, linux, python, mongodb, interview]
status: 面试重点
created: 2026-07-17
updated: 2026-07-20
---

# EDS NFSv4 Kerberos 域接入与用户映射管控

## 项目一句话

为 EDS 文件存储补齐 NFSv4 `sec=krb5` 企业认证能力：管理面编排 KDC 域、SPN 和全节点凭据，NFS-Ganesha 校验票据并将 Principal 映射为 UNIX 身份，最终由 PHXDFS 按原有权限模型授权。

## 1 分钟介绍

原有 NFS 共享仅支持 `sec=sys`，依赖客户端提供 UID/GID，无法接入企业 Kerberos 认证体系。我负责从需求分析、方案设计到核心开发和问题收口，完成了域管理、SPN/keytab、Principal 映射与 Ganesha 联动。

管理员配置 Realm、KDC 和 SPN 策略后，系统以异步任务在全节点生成并验证 keytab，同步 `krb5.conf`、映射和 Ganesha 配置；客户端可用 `sec=krb5` 挂载。难点在于 KDC、数据库、节点配置和数据面的状态收敛，因此实现了回滚、互斥、健康检查和扩容同步。最终形成了从身份认证到文件权限生效的完整闭环。

## 认证与授权链路

```text
客户端 kinit 获取 TGT
  -> rpc.gssd 以 TGT 向 KDC 申请 nfs/<fqdn>@REALM 服务票据
  -> RPCSEC_GSS 将 AP-REQ 交给 NFS-Ganesha
  -> Ganesha 用服务端 keytab 验证票据并建立 GSS 上下文
  -> principal_map.conf: user@REALM -> UNIX 用户
  -> NSS/LDAP 解析 UID/GID，PHXDFS 按 UID/GID、mode、ACL 授权
```

Kerberos 解决“你是谁”，Principal 映射决定“以哪个 UNIX 身份执行”，文件系统权限决定“你能做什么”。缺少映射时，即使票据合法也可能身份解析失败或被拒绝访问。

### 服务票据与 GSS 上下文

1. 用户通过 `kinit`、PAM 或域登录获取 TGT，保存在客户端凭据缓存。
2. NFSv4 挂载触发 `rpc.gssd`，它用 TGT 向 TGS 申请 `nfs/<fqdn>@REALM` 服务票据。
3. 客户端在 RPCSEC_GSS 上下文初始化中发送 AP-REQ，其中包含服务票据和用会话密钥生成的 authenticator；不会发送用户密码或服务端 keytab。
4. Ganesha 通过 GSSAPI 在 keytab 中找到服务 Principal 的长期密钥，解开服务票据、校验 authenticator、时间戳和服务名后建立上下文。
5. 后续 RPC 复用该上下文，不需要每次请求访问 KDC。`krb5i` 在此基础上增加完整性保护，`krb5p` 再加密 RPC 数据。

DNS 名称、KDC 中的 SPN、服务端 keytab 必须一致。错误 FQDN 或使用 IP 挂载可能导致服务票据的目标 Principal 不匹配；实际是否规范化为某个名称取决于客户端 Kerberos 配置。

### 安全模式

| 模式 | 认证 | 完整性 | 加密 | 当前产品 |
|---|---|---|---|---|
| `sys` | 客户端 UID/GID | 无 | 无 | 支持 |
| `krb5` | 有 | 无业务数据完整性保护 | 无 | 支持 |
| `krb5i` | 有 | 有 | 无 | 未实现 |
| `krb5p` | 有 | 有 | 有 | 未实现 |

当前允许共享配置 `sys`、`krb5` 或二者并存，目标是身份可信和存量客户端渐进迁移。`krb5i/krb5p` 属于未实现的需求范围，不应表述为已做过专项性能取舍。

### 协议与兼容性范围

NFSv4、NFSv3 和 pNFS 在协议理论上都可能结合 RPCSEC_GSS；当前限制是产品实现范围，而不是协议绝对限制：

- NFSv4 的挂载、状态和安全上下文较统一，本期完成了普通 NFSv4 路径的 SPN、keytab、映射和验收目标。
- NFSv3 还涉及 mountd、lockd、statd 等辅助 RPC 服务，未完成整链路兼容验证，产品保持 `sys` 以保护存量客户端。
- pNFS 需要同时验证 MDS、多个 DS、layout 切换和故障重连后的认证一致性；当前未实现该数据路径的专项验证，产品也保持 `sys`。
- 当前共享数据模型只有统一 `sec_type`，无法分别表达 NFSv3/NFSv4/pNFS 的安全类型；pNFS 与 `krb5` 的互斥主要依赖页面规则，后端校验不完整。

## 核心实现

### 加域：五阶段补偿流程

1. 前置检查：存储池、重复加域、KDC 88/749 端口及 NFS 导出分布式锁。
2. 全节点生成并分发 `krb5.conf`。
3. 管理 SPN：不存在则创建，已有 SPN 可复用或显式覆盖。
4. 所有节点执行 `ktadd -norandkey` 和 `kinit -kt`。
5. 最后写入域状态并启用 Ganesha，将根导出切为 `sys,krb5`。

Taskflow 在进程内普通异常时逆序补偿：清理 keytab、按归属删除本次创建的 SPN、恢复 `krb5.conf` 和 Ganesha 配置。数据库与数据面在最后阶段才启用，尽量缩小半加域窗口。

#### 部分成功与回滚顺序

例如 keytab 阶段中，部分节点 `ktadd/kinit` 成功、某节点失败时，异常会向上抛给 Taskflow，随后执行逆序补偿：全节点删除本次 keytab、按 `spn_created` 删除本次创建的 SPN、恢复旧 `krb5.conf`。此时数据库通常尚未写入 `connected`，Ganesha 也尚未启用 Kerberos。

若最后的状态落库或 Ganesha 分发失败，补偿还会删除域记录，重新生成未启用 Kerberos 的 Ganesha 配置，并将根导出从 `sys,krb5` 恢复为 `sys`。这属于 Saga 式补偿，不是严格分布式事务：节点失联、回滚失败或进程退出时，无法保证所有外部副作用都自动恢复。

### SPN、keytab 与所有权

- `ktadd -norandkey` 只提取当前 SPN 密钥，保证所有节点拥有相同 Principal 和 KVNO；普通 `ktadd` 会轮换密钥，导致旧节点无法验证新票据。
- 复用既有 SPN 时不改密钥，`spn_created=false`，失败或退域均不删除外部 SPN。
- 覆盖会删除并重建 SPN，代表管理员明确授权 EDS 接管；旧 SPN 的密钥无法完整恢复，因此不是可逆操作。

复用模式适合管理员预创建、历史 EDS 或外部系统正在使用的 SPN。覆盖模式下若“删除旧 SPN 后、新 SPN 创建前”失败，或后续加域失败，当前实现不会恢复旧 SPN 的密钥、KVNO、策略和元数据；因此默认应优先复用，覆盖必须由管理员确认没有其他使用方。

### Principal 映射

映射是有序正则规则表：按 `priority` 从小到大首次匹配即停止。正则支持批量映射，优先级解决重叠规则；精确规则应排在 Realm 级兜底规则之前。当前限制最多 1000 条，防止认证热路径与管理复杂度失控。

例如 `^alice@EXAMPLE\\.COM$ -> eds_admin` 与 `^.*@EXAMPLE\\.COM$ -> eds_domain_user` 会同时命中 `alice@EXAMPLE.COM`，最终取决于优先级。当前只拒绝完全相同的 pattern，不会证明两个正则是否语义重叠。映射规则排序因此属于权限语义，应审计变更；更稳妥的后续能力是规则试算，展示全部命中规则、最终选择和解析出的 UID/GID。

### 互斥与健康检查

- 加域、退域、NFS 共享创建/编辑/删除共用集群范围的 `nfs_export_mutex`，避免数据库、Ganesha 配置和运行态导出相互覆盖。
- 健康检查在各节点实际执行 `kinit -kt` 与 `kadmin get_principal`；任一节点失败则域状态为 `disconnected`。
- 告警每分钟检测，单节点连续失败 3 次才报警；相同告警去重，恢复后 180 秒未复发自动清除。

健康检查验证 keytab、KDC、管理凭据和 SPN，不等价于真实客户端挂载和文件 I/O 成功。

健康检查每 3 分钟在每个节点的 chroot 中执行 `kinit -kt /etc/kerberos/krb5.keytab <SPN>` 和 `kadmin get_principal <SPN>`：前者覆盖 keytab、KVNO、KDC 连通性与基本时间/Realm 配置，后者覆盖管理凭据、KDC 管理面与 SPN 存在性。任一节点失败即更新域状态为 `disconnected`，没有连续失败去抖；告警任务则独立每分钟运行，连续 3 次失败后告警。因此短时可能出现状态已断开、告警尚未触发，两者语义不同。

### 扩容与替换

仅对新文件服务节点同步 `krb5.conf`、`principal_map.conf`、重新提取并验证 keytab、再分发 `ganesha.conf`。不直接复制旧 keytab，也不轮换 SPN 密钥。NFS 服务启动后仍需用真实 `sec=krb5` 挂载验证新节点。

同步前，无域配置会直接跳过；有域时，扩容会被未恢复的 Kerberos 连接告警阻止。主机替换可放行仅来自被替换故障节点的告警，但保留节点存在告警仍会阻止操作。新节点的同步顺序为：`krb5.conf`、映射、`ktadd -norandkey + kinit -kt`、`ganesha.conf`；Ganesha 尚未启动时只分发文件，后续部署启动时加载配置。

理想的回滚会按已完成阶段在新节点逆序清理：恢复 Ganesha、`kdestroy` 并删除 keytab、删除映射、恢复不含 Realm 的 `krb5.conf`。但当前阶段状态只记录“全部节点完成”，无法表达节点 A 已成功、节点 B 已失败的部分成功现场。

## 已确认边界

| 主题 | 当前事实 |
|---|---|
| 锁与宕机恢复 | TTL 约 10 分钟，但 Kerberos 任务没有续租、owner token 或 fencing token；进程宕机只会在重启后标失败和解锁，不会自动补偿业务状态。 |
| 回滚 | 普通异常可触发 Taskflow 补偿；回滚普遍 `catch_not_throw`，节点失联时可能残留配置或 keytab。 |
| 扩容同步 | 同步异常会被 `ExpandKerberosTask` 记录后吞掉；阶段状态不是逐节点记录，部分成功可能残留。旧集群通常不受影响。 |
| 映射规则 | 仅拒绝完全相同的 pattern，不识别不同正则的语义重叠；错误排序可能映射到错误甚至高权限 UNIX 用户。 |
| 协议范围 | NFSv3 与 pNFS 当前产品只允许 `sys`，这是实现和兼容性范围，不是协议绝对不支持 Kerberos。pNFS 与 `krb5` 的互斥主要是页面规则，后端校验不完整。 |
| 管理员凭据 | API 入参 RSA 解密，数据库可逆加密保存，查询使用占位符；但运行时仍以 `kadmin -w` 传入命令行，并经 Node Agent 发送，存在 argv、日志和传输保密风险。 |

### 锁、TTL 与进程故障细节

`nfs_export_mutex` 的 TTL 索引约为 600 秒。Kerberos 流程只在入口获取一次锁，不会续租；超过 TTL 后，MongoDB TTL Monitor 可能删除锁记录，原任务并不知道自己已失锁，新任务可能进入临界区。TTL 删除不是精确发生在第 600 秒，而受 MongoDB 扫描周期影响。

当前释放和续租不校验任务 owner 或租约代次。旧任务锁过期后，若新任务已获得同名锁，旧任务结束时可能释放新任务的锁。进程/节点直接宕机时，`finally`、Taskflow 回滚和任务失败处理都不会立即执行；同 hostname 重启后会清理该 hostname 的锁并将 processing 任务标失败，但不会恢复 SPN、keytab 或节点配置。

此外，任务表可能阻止第二个加退域任务，但普通 NFS 共享操作同样依赖此锁，不受任务表约束。锁只能协调使用同一把锁的管理面操作，无法阻止管理员绕过系统直接修改 KDC 或节点配置。

### 管理员凭据生命周期

管理 API 以 RSA 密文提交管理员密码，Provider 用服务端私钥解密后传入 eventlet 异步 Taskflow。密码通常不写入通用 task 表，但会短暂存在于 Python 参数、Task 属性和共享上下文中；Python 字符串无法可靠原地清零。

加域成功后，管理员密码以 `Prpcrypt` 的可逆 AES-CBC 密文保存在 `kerberos_domain.admin_password`，因为退域、扩容、健康检查仍需执行 `kadmin`。查询接口返回固定占位符；编辑时提交占位符代表复用旧密码，避免把真实密码回显给前端。

当前最大暴露面是 `kadmin -w <password>`：`pipes.quote()` 只能防 shell 注入，不能阻止密码出现在远端进程 argv、`/proc/<pid>/cmdline`、审计工具或 Node Agent 请求中。部分日志路径会脱敏，但 HTTP Debug、异常分支或不完整的正则脱敏仍可能泄露。字段级 RSA 也不能替代 HTTPS；Node Agent 是否全程 TLS/mTLS 需要按部署确认。

## 改进建议（非当前交付）

1. 先为分布式锁补齐 owner token、CAS 释放/续租与 fencing token；失锁后禁止继续写入。
2. 持久化业务状态和逐节点阶段结果，将流程改为探测、收敛、读回验证的幂等步骤。
3. 上线状态对账：比对 MongoDB、KDC SPN/KVNO、各节点配置/keytab 和 Ganesha 运行态，先告警和人工修复，再开放低风险自动补偿。
4. 使用受限管理 Principal 的 keytab 替代 `kadmin -w`；Node Agent 使用 TLS/mTLS，统一结构化脱敏，并迁移至支持轮换的认证加密或密钥管理服务。

## 测试与验收

本轮仅静态盘点到约 208 个 Kerberos 专属 `test_*`，**未实际执行，不作为通过率或覆盖率结论**。单元测试重点覆盖 API 校验、SPN 三分支、加退域补偿、映射排序、健康检查、NFS 安全类型和扩容入口。

单元测试证明的是 Cluster Manager 的编排、分支和回滚被正确调用，不能证明真实 Kerberos 与 NFS 数据面可用。应重点补齐：`ktadd_and_kinit_all_nodes()` 的命令和异常传播、加退域每阶段故障注入、部分节点 keytab 成功后的清理、扩容各阶段回滚、未加域时拒绝创建 `krb5` 共享，以及健康告警的阈值、去重和恢复。

端到端验收必须同时证明：

1. 客户端获得 `nfs/<fqdn>@REALM` 服务票据，并以 `sec=krb5` 挂载。
2. 客户端 I/O 成功，服务端 `stat` 的 UID/GID 等于映射后的 EDS 用户，而非客户端本地 UID。
3. 合法但无映射、无票据、错误 FQDN/SPN、`sec=sys` 访问仅 `krb5` 共享等负向场景均失败。
4. 多节点、扩容/替换和退域后行为一致。

仓库系统测试目前主要验证 API 可保存/读取 `sec_type=["krb5"]`，不等价于真实票据、挂载、映射与权限的端到端验收。

推荐端到端环境使用两个 Principal 和两个 EDS UNIX 用户，并刻意让客户端本地 UID 与 EDS 映射 UID 不同。共享先只允许 `krb5`，以排除实际悄悄降级为 `sys` 的可能。除正向 I/O 外，还要验证：无票据后的真实 I/O、合法但无映射的 Principal、错误 FQDN/SPN、`sec=sys` 挂载仅 `krb5` 共享，以及 Alice/Bob 对 0700 私有目录和共享组目录的交叉权限矩阵。

## 真实问题案例：Windows DNS 导致加域约 5 分钟

**现象**：EDS 使用 Windows LDAP/AD 域 DNS 时，加入独立 Kerberos KDC 最终能成功，但耗时约 5 分钟，超出 30 秒需求目标。

**定位**：88/749 端口、账号、SPN 均可用；卡顿集中在生成 `krb5.conf` 后的 `kinit/kadmin`。配置未显式关闭反向 DNS 规范化，Windows DNS 的反查缺失、缓慢或不一致触发 Kerberos 超时重试。

**修复**：在 `[libdefaults]` 增加 `rdns = false`，在 Realm 中显式设置 `master_kdc`；AD 与独立 Kerberos 共用生成逻辑，退域恢复路径一并修正。

**验证**：在原 DNS 环境复测加域，确认消除分钟级等待并满足需求目标；校验所有节点配置、`kinit -kt`、`kadmin get_principal`、Principal/KVNO 一致性，并增加配置生成单测断言。提交：`576bad4e4c`。

## 高频追问

1. 为什么 Principal 必须映射为 UNIX 用户？认证、映射、授权分别解决什么问题？
2. 客户端如何拿到服务票据？Ganesha 如何用 keytab 验证？
3. 为什么使用 `ktadd -norandkey`？SPN 复用与覆盖如何保证所有权？
4. 为什么加退域要和 NFS 共享操作共用集群锁？TTL 过期和宕机的边界是什么？
5. 如何证明功能真的生效，而不只是 API 或配置写入成功？
6. 你处理过什么问题？如何定位 Windows DNS 导致的 Kerberos 加域慢？

## 关联知识

- [[Linux知识地图]]
- [[项目知识地图]]
