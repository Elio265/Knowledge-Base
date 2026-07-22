---
tags: [project, sangbridge, ai, gateway, fastapi, vue, vcs, interview]
status: 面试重点
created: 2026-07-20
updated: 2026-07-22
---

# SangBridge AI统一接入与管理控制台

## 项目定位

SangBridge 面向 EDS 5.3.2，以 Vue 3 控制台和 Python 3.11/FastAPI 网关统一接入 VCS、UniView 等 AI 与基础设施上游服务：

```text
用户 -> SangBridge Web / FastAPI 网关 -> phxoss VCS、UniView 等上游服务
```

系统负责统一界面、登录身份、上游请求转发、错误转换、审计、限流和部署测试等能力。本人已核验的主要贡献聚焦 **VCS 管控面**：权限治理与游标分页正确性；登录、Token、WAF、审计、SigV4/S3 Express 和底层 phxoss/univ-srvd 实现不计入个人成果。

## 个人工作范围

内网代码与提交盘点显示，作者 `王志豪17061 <17061@...>` 有 98 个相关提交。已核验范围包括：

- VCS 仓库、分支、标签、提交、对比、MR、设置等页面的前端和网关适配。
- 数据面 `/permissions` 权限集合到按钮、请求前校验和资源创建人约束的前端治理。
- 仓库、分支、标签、提交图、文件树等接口的游标分页契约对齐。
- VCS 表单、国际化、时区等零散质量 TD；仅作追问素材，不作为项目主线。

## 案例一：VCS 权限治理

### 背景与问题

关联 TD：`2026052700070`、`2026070300114`。

修复前，部分 VCS 操作入口只判断页面角色或资源创建人，没有完整消费 phxoss `/permissions` 返回的 API 权限集合。缺少 `CreateTag`、`DeleteTag`、`DeleteBranch`、`CreateMR` 或 `vcs:GetDiff` 时，用户仍可能点击按钮并向上游发起请求，随后才收到权限错误。

这不是已证实的数据泄露或后端越权：phxoss Bucket Policy 仍是最终授权边界，无权限请求应由上游拒绝。前端治理的价值是让控制台状态与数据面策略一致，提前阻止误操作和无效请求，并给出明确反馈。

### 授权分层与调用链

```text
ResourceDetail(route.params.id)
  -> repoPermissionsStore.load(repoId)
  -> repoApi.permissions(repoId)
  -> GET /api/sangbridge/v1/vcs/repos/{repoId}/permissions
  -> RepoMgr / VcsClient
  -> GET phxoss /vcs/v1/repos/{repoId}/permissions
  -> Action、Button、Method+URL 权限集合
  -> useRepoPermissions 生成 canCreate/canDelete/canGetDiff 等能力
  -> 按钮置灰 + 部分 Handler 请求前拦截
  -> SangBridge 转发用户身份，phxoss Bucket Policy 最终授权
```

- SangBridge VCS Router 使用 `Depends(get_identity)` 取得登录身份，并通过 `X-User-Arn` 与用户凭据签名向 phxoss 转发。
- phxoss 权限响应包含 `bucket_name`、`user`、`is_owner` 与 `apis[]`；前端可按 IAM Action、Button 标识或 Method+URL 三种方式匹配。
- 权限数据按 `repoId` 缓存，仓库切换会重新加载，不同仓库不直接共用记录。

关键路径：

- `web/src/views/vcs/ResourceDetail.vue:137,212,348,382`
- `web/src/stores/repoPermissions.ts:62,88,102,110`
- `web/src/composables/useRepoPermissions.ts:29,36`
- `web/src/api/vcs/index.ts:48`
- `server/sangbridge_server/module/storage/vcs/repo/web_api/router.py:77`
- `server/sangbridge_server/module/storage/vcs/repo/manager/repo_mgr.py:89`
- `server/sangbridge_server/client/phxoss/vcs/vcs_client.py:812`

### 具体改动与个人归属

| 提交/TD | 本人改动 | 证据与规模 |
|---|---|---|
| `1e91157` / TD `2026052700070` | 分支删除叠加数据面权限与创建人判断；标签创建/删除权限；新建 MR 校验源分支归属 | 作者和提交者均为本人；6 文件，`+225/-18` |
| `63d8fdb` | 新增仓库权限查询接口、Pinia Store、`useRepoPermissions`，并接入分支、标签、提交/回退、MR 等入口 | 作者和提交者均为本人；32 文件，`+962/-47` |
| `5115ec0` / TD `2026070300114` | 补齐 `vcs:GetDiff` 与 compare 映射；对比/刷新按钮置灰；`loadDiff()` 请求前拦截；补无权限不请求 Diff/Diff Summary 的测试 | 作者和提交者均为本人；4 文件，`+61/-4` |

`5115ec0` 后由其他成员通过 `5879c6a` 合入开发分支。因此个人表述应是“我完成 `5115ec0` 修复”，不能把他人的合并提交写成自己的工作。

### 权限与创建人约束

权限最终由四类条件共同决定：上游权限集合、当前仓库角色、资源创建人归属和页面业务条件。

- `admin` 可以操作任意创建人的资源。
- `developer` 只能操作 `creator_id` 等于当前用户名的资源。
- `visitor` 不允许写操作；角色未加载时也按不可操作处理。
- 业务条件例如：不能删除默认分支；创建 MR 要求源/目标均为分支；源分支归属不明确或属于其他人时禁止提交。

涉及页面：`BranchTab.vue`、`TagTab.vue`、`CommitTab.vue`、`CompareTab.vue`、`NewPRModal.vue`、`PRApprovalModal.vue`。

### 设计取舍与已确认边界

按钮置灰解决正常用户误点击和错误反馈滞后；Handler 前置判断覆盖组件方法被直接调用或状态切换期间的事件触发。两层都不能替代服务端授权，直接构造 SangBridge HTTP 请求可绕过前端，最终仍依赖 phxoss Bucket Policy。

| 当前行为 | 结论 |
|---|---|
| 首次加载、加载中、权限接口异常 | Fail-closed：权限默认为 false，异常缓存空权限集合 |
| 已缓存仓库切回 | 直接复用旧缓存 |
| 权限动态变化 | 无 TTL、未发现强制刷新/清理调用，可能短暂陈旧 |
| 权限接口首次失败 | 空权限会标记为已加载，普通加载不会自动重试 |
| Handler 保护一致性 | `Compare.loadDiff()`、新建 MR、MR 提交/关闭有前置判断；部分分支、标签、MR Handler 仍主要依赖按钮 `disabled` |

需要后续补齐的风险包括：权限缓存 TTL 或失效通知、失败重试、仓库 A/B 切换与旧请求乱序专项测试、统一 Handler 前置检查，以及真实 phxoss 拒绝 `vcs:GetDiff` 的端到端验证。

### 测试证据

- 历史提交正文记录 5 个测试文件、41 个用例；缺少独立 CI 产物，不能还原当时执行现场。
- 本次实际复跑权限相关定向测试：7 个文件、118 个用例通过。
- 本次完整前端验证：类型检查通过，49 个测试文件、560 个用例通过。
- 代表测试：`repoPermissions.spec.ts` 的三类权限匹配和异常空权限；`CompareTab.spec.ts:367` 的无权限不请求 Diff/Diff Summary；`BranchTab.spec.ts:1065` 的创建人限制；`NewPRModal.spec.ts:302` 的非本人源分支拦截；`PRApprovalModal.spec.ts:1009` 的无更新/关闭权限拦截。

## 案例二：VCS 游标分页治理

### 背景与问题

关联 TD：`2026062500043`、`bug-vcs-repo-pagination-17061`。

VCS 各接口不是同一种分页模型：仓库列表、分支/标签、提交图、文件树和 S3 快照分别拥有不同游标或 token 语义。旧代码曾依据当前返回条数推测下一页，或把文件名当作 S3 continuation token，造成 exact-page 假下一页、加载更多失败、快照漏数据，以及仓库默认只能显示前 50 条等问题。

核心原则是：**前端以服务端 `truncated` 与 `after` 为准；只有部分 ID 型接口确实缺少游标时，才用最后一项标识作兼容兜底；S3 continuation token 不可用文件名替代。**

### 分页语义与调用面

| 场景 | 正确语义 |
|---|---|
| 仓库列表 | 传 `limit/after`；优先使用上游 opaque `after`，确实缺失时才回退最后 `bucket_name`；搜索、页大小和刷新应重置游标 |
| 分支/标签 | 以后端 `truncated` 决定是否存在下一页，优先使用后端 `after`；来源选择器同样按 cursor 加载更多 |
| 提交图 | `after` 属于不透明游标，加载更多应透传、追加结果；末页禁用继续加载 |
| 文件树 | 普通路径与 tag/commit snapshot 的上游不同；S3 snapshot 必须使用 continuation token 续页，不能把文件名当 token |
| 对比/MR/设置选择器 | 分支、标签选择器使用 cursor，搜索或仓库切换时清空旧 `after` |

通用前端状态还处理了：末页 total 修正、fetch 失败清空状态、请求乱序丢弃旧响应、`pageSize`/`reset` 清空游标。对应 `web/src/composables/useCursorPagination.spec.ts:31,75,101,132`。

### 具体改动与个人归属

| 提交/TD | 本人改动 |
|---|---|
| `f419af0` / TD `2026062500043` | 修复 VCS 假下一页，梳理分支、标签、提交图、文件树等页面的 cursor 续页、搜索与重置行为 |
| `5909c83` | 修复标签和提交快照文件树把文件名错误当作 S3 continuation token 的问题 |
| `d680d54` | 恢复直接消费后端 `truncated/after` 的契约，避免依据前端记录数量臆测下一页 |
| `fea89f6` / `bug-vcs-repo-pagination-17061` | 仓库列表接入服务端游标分页与搜索，突破默认仅显示前 50 条 |

代表前端路径：

- `web/src/views/vcs/index.spec.ts:182,201,228,253`
- `web/src/views/vcs/tabs/BranchTab.spec.ts:615,669,717,784,821,1157`
- `web/src/views/vcs/tabs/TagTab.spec.ts:87,118,214,283`
- `web/src/views/vcs/tabs/CommitTab.spec.ts:259,392,935,974`
- `web/src/views/vcs/tabs/DataTab.spec.ts:480,1331,1361`
- `web/src/views/vcs/tabs/CompareTab.spec.ts:111`
- `web/src/views/vcs/modals/NewPRModal.spec.ts:204,266`
- `web/src/views/vcs/tabs/SettingsTab.spec.ts:140`

### 测试证据与未覆盖项

本次实际运行前端 Vitest：

```text
useCursorPagination + index + BranchTab + TagTab + CommitTab + DataTab
+ CompareTab + NewPRModal + SettingsTab
```

结果：9 个测试文件、175 个用例通过、0 失败、退出码 0。

后端已确认存在分页测试代码，但本次**未执行**，不能写成测试通过：

- `server/tests/integration/vcs/test_branch_router.py`
- `server/tests/integration/vcs/test_tag_router.py`
- `server/tests/integration/vcs/test_commit_router.py`
- `server/tests/integration/vcs/test_workspace_router.py`
- `server/tests/unit/vcs/test_workspace_mgr.py`
- `server/tests/unit/client/test_vcs_client_coverage.py`

后端 pytest 需通过远端测试宿主机和 CI 镜像入口执行，本地 pytest 不作为验证证据。

当前仍缺：仓库 Router `GET /repos?limit=&after=` 集成测试；后端 exact-page 契约；空页但 `truncated=true`、`after` 不前进等异常契约；三页以上翻页和回退；仓库切换后的旧请求覆盖；搜索与翻页并发；S3 token 失效/变化；以及上游重叠数据的去重策略。

## 可用于简历的表述

负责 EDS 5.3.2 SangBridge VCS 模块的权限与分页治理：建设仓库权限查询、Pinia 权限缓存及按钮级能力控制，覆盖分支、标签、提交回退、差异对比和 MR，并结合资源创建人限制开发者误操作；统一仓库、分支、标签、提交图和 S3 快照文件树的游标分页契约，修复假下一页、快照漏数据和仓库列表仅展示前 50 条等问题。权限核心提交 `63d8fdb` 涉及 32 个文件、`+962/-47`；本次复跑权限和分页前端专项测试共 16 个文件、293 个用例通过。

## 1 分钟面试回答

SangBridge 是 EDS 5.3.2 的 AI 与基础设施统一管控平台，使用 Vue 3 和 FastAPI 统一接入 VCS、UniView 等上游服务。我主要负责 VCS 管控面的权限治理和游标分页正确性。

权限方面，原来部分分支、标签、MR 和对比入口只判断角色或创建人，未完整消费 phxoss 的权限集合。我新增仓库权限 API、按仓库缓存的 Pinia Store 和统一权限组合函数，接入分支、标签、提交回退、对比和 MR 页面；同时叠加 `creator_id` 约束。后续发现对比页漏用了 `vcs:GetDiff`，我补齐按钮置灰和 `loadDiff()` 请求前拦截。这里的前端治理负责体验和防误操作，最终安全边界仍由 phxoss Bucket Policy 保证。

分页方面，我统一了仓库、分支、标签、提交图和 S3 文件树的游标语义，坚持以后端 `truncated/after` 为准，修复了 exact-page 假下一页、S3 快照 token 错用和仓库超过 50 条后无法继续查看的问题。本次复跑相关前端测试，权限与分页共 16 个文件、293 个用例通过。

## 高频追问

1. 为什么前端权限治理不能替代后端授权？Fail-closed 和权限缓存的边界是什么？
2. 为什么按钮置灰后还要在 Handler 前再次校验？哪些入口尚未统一？
3. `truncated`、普通 `after`、提交图 opaque cursor、S3 continuation token 分别为什么不能混用？
4. 如何证明分页没有假下一页或漏数据？前后端测试分别覆盖什么？
5. 你在项目中做了什么，哪些工作明确不属于你？

## 事实边界

- TD 平台查询返回 403，TD 正式负责人、验收状态主要由 Git 作者和提交正文交叉证明，不能过度承诺。
- `1e91157` 不在当前开发分支祖先链，需谨慎表述为个人提交；`63d8fdb` 是当前权限基础设施的核心证据。
- 未拿到生产无效请求下降比例、性能变化、用户规模或缺陷逃逸率，不写虚构量化结果。
- UniView 契约适配可作为后续补充案例，当前未纳入个人项目主线。

## 关联知识

- [[项目知识地图]]
- [[Linux知识地图]]
