---
tags: [project, sangbridge, ai, gateway, fastapi, vue, vcs, uniview, architecture]
status: 学习中
created: 2026-07-20
updated: 2026-08-13
---

# SangBridge AI统一接入与管理控制台

> **视角**：整个项目是我的。**唯一边界**：phxoss VCS 是上游数据面（由 phxoss 团队负责）。
> **本笔记用途**：项目理解——架构、模块、机制、边界。

## 项目价值

> 视角：从"为什么存在"反推项目骨架。**价值锚点决定架构优先级**——架构图、模块设计、薄弱点都按核心价值排序，而不是反过来。这一段是整篇笔记的"第一性原则"基础。

### 核心痛点（按重要性排序）

1. **统一入口与用户体验** — 把 UniView、phxoss VCS 等分散能力收敛到同一 Web/API 入口，减少用户在多个系统间切换。
2. **身份与权限贯通** — 复用 EDS 统一用户身份：对 UniView 透传 ARN 做权限/租户隔离；对 VCS 用用户自己的 AK/SK 做 SigV4 签名，保留真实操作人。
3. **安全边界统一** — 浏览器不接触 AK/SK；所有访问经 WAF、Cookie 会话校验、IP 绑定、限流和可选审计，再调用内部上游。
4. **将上游协议复杂性从前端隔离** — 前端只调用统一 REST API；SangBridge 屏蔽 UniView 专用头、VCS/S3 SigV4、目录桶临时凭据、分片上传下载、超时和错误翻译等复杂细节。
5. **高可用下的连续访问** — 服务部署在 EDS 各节点，随 VIP 漂移切换；会话和用户状态放在共享 Mongo，用户不必因节点切换重新登录。
6. **统一运维与治理** — 统一 WAF 入口、日志、请求 ID、限流、审计和部署升级路径，使跨上游问题可追踪、可运维。

**其他维度**：SangBridge 也是 EDS 面向 AI / 数据管理能力的网关层，为后续新增上游服务提供统一接入模式。

### 关键用户与场景

**最关键用户**：EDS 平台中的**统一用户**，具体三类：

| 用户角色 | 核心职责 | 关键操作路径
|---|---|---|
| **数据管理员** | UniView catalog / 数据源 / 成员权限配置；查看迁移任务、处理预热 / 数据流转 | 登录 → UniView catalog 管理 → 数据源配置 → 任务监控 |
| **数据工程师 / 数据使用者** | 跨数据源搜索、定位数据；执行导出 / 迁移 | 登录 → UniView 搜索 → 数据定位 → 导出 / 迁移任务 |
| **数据版本管理人员** | VCS 仓库 / 分支 / 提交 / Diff / MR；对象文件上传下载 | 登录 → VCS 仓库 → 分支 / 提交 → Diff → MR → 对象管理 |

**最关键操作路径（高层）**：

```text
登录 SangBridge
  → 通过 UniView 查找/管理数据源与目录
  → 必要时创建迁移、预热或导出任务
  → 通过 VCS 管理对象数据版本、分支、提交和合并请求
```

**核心底层路径（用户价值的技术映射）**：

```text
用户登录
  → SangBridge 复核统一身份
  → 以 ARN 调 UniView、以用户 AK/SK 签名调 VCS
  → 上游按真实用户执行权限判断与审计
```

**核心用户价值不只是"提供页面"**，而是让用户能在一个统一入口中安全地完成**"找数据、管数据、迁数据、管版本"**四类操作。

### 上线前后对比

#### 上线前的痛点

- UniView、phxoss VCS 等能力入口分散，用户需在不同页面或系统间切换
- 前端或调用方需要分别适配上游协议、认证和错误，尤其 VCS 的 S3 / SigV4、上传下载、目录桶等复杂度高
- 用户身份难以一致传递：UniView 需要 ARN，VCS 需要用户 AK/SK；跨系统的权限和审计链路容易断裂
- VIP / 节点切换时，统一登录态和操作连续性需要各系统自行处理
- 网关治理能力分散：WAF、限流、请求追踪、审计策略不能在一个入口统一执行

#### 上线后可验证的改进

| 改进 | 可验证证据 |
|---|---|
| 统一入口 | `https://<集群VIP>:4431/open/api/sangbridge/v1` 可访问 UniView + VCS 功能 |
| 身份贯通 | UniView 收到 `X-IFGW-User`；VCS 请求带该用户 AK 的 SigV4 签名 |
| 凭据不暴露 | 浏览器仅有 HttpOnly `sb_auth` cookie；响应和前端存储不出现 ARN / AK / SK |
| 权限正确 | 无 ARN 的 UniView 请求被上游拒绝；无 AK/SK 的 VCS 请求返回 `VCS-CREDENTIALS-MISSING` |
| 切换连续性 | 节点 A 登录后 VIP 切到 B，同一 cookie 可继续使用（token 存于共享 Mongo） |
| 统一安全治理 | 请求必须经 WAF；伪造 / 缺失 WAF 签名转发头被拒；限流超限返回 `rate_limited` |
| 可追踪性 | 每个请求带 `X-Request-Id`，可在 SangBridge 与 WAF 日志按 ID 关联排查 |
| 上游复杂性屏蔽 | 前端仅调 SangBridge REST；SigV4、目录桶 Session、Multipart 等由后端完成 |

**注意**：持久化接口审计当前默认 `audit.enabled=false`。若把"审计日志已落库"作为上线收益，应先在生产启用 `audit.enabled=true` 并验收 Mongo `audit_logs` 写入。

### 失败成本

**完全失去**：

- SangBridge Web 页面 + `/open/api/sangbridge/v1/*` API 入口
- UniView 能力（数据目录 / 数据源 / 搜索 / 迁移 / 预热 / 导出 / 搜索模板）
- VCS 能力（仓库 / 分支 / 提交 / 标签 / Diff / MR / 对象文件上传下载）
- SangBridge 统一登录态、身份透传、网关安全治理能力

**不失去**：

- EDS 核心存储、集群管理、对象存储数据本身
- phxoss VCS、univ-srvd 上游服务本身（独立服务，仍运行；只是用户无法通过 SangBridge 正式入口访问）
- 已存数据和 Mongo 中已有 token / 审计记录

**本质**：**"AI 数据管理与数据版本管理入口不可用"**，而不是整个 EDS 存储集群不可用。若上游提供其他受支持入口，管理员理论上可绕过 SangBridge 访问，但会失去 SangBridge 提供的统一身份映射、浏览器侧凭据保护和标准操作体验。

### 价值驱动的优先级

从核心痛点反推各模块 / 机制的优先级（后续架构图和薄弱点按此重排）：

| 价值层 | 对应模块 / 机制 | 优先级 |
|---|---|---|
| 统一入口 | `## 架构设计` 中的部署架构 + 模块图；Router 层 | **P0** |
| 身份贯通 | 登录 + Token 存储 + get_identity + Identity 注入 | **P0** |
| 凭据不暴露 | AES-CBC 加密 + 运行时解密 + Cookie HttpOnly | **P0** |
| 上游协议隔离 | VcsClient（SigV4、目录桶 Session）+ UnivSrvdClient（X-IFGW-User 头） | **P0** |
| 高可用连续访问 | 进程内 stateless + 共享 Mongo 恢复身份 | **P0** |
| 统一安全治理 | WAF（外层）+ WafFilterMiddleware（内层）+ TokenBucket + Audit（**默认关闭**是 P1 风险） | **P0**（治理） / P1（审计启用） |
| 可追踪性 | SysLogHandler → EDS syslog；X-Request-Id 关联 | **P1** |
| 快速接入新上游 | 上下游分层 + Client 抽象（已支持 UniView / VCS） | **P2** |

后续 **## 核心模块设计**、**## 网关机制** 段落的展开顺序也按此表。

---



## 项目定位

SangBridge 面向 EDS 5.3.2，以 Vue 3 控制台和 Python 3.11/FastAPI 网关统一接入 VCS、UniView 等 AI 与基础设施上游服务：

```text
用户 -> SangBridge Web / FastAPI 网关 -> phxoss VCS、UniView 等上游服务
```

系统负责统一界面、登录身份、上游请求转发、错误转换、审计、限流和部署测试等能力。本人已核验的主要贡献聚焦 **VCS 管控面**：权限治理与游标分页正确性；登录、Token、WAF、审计、SigV4/S3 Express 和底层 phxoss/univ-srvd 实现不计入个人成果。


## 架构设计

> 视角：整个项目是我的。从全局视角梳理 SangBridge 的部署、模块、数据流。这部分用于建立**架构骨架**，具体模块细节在后续"## 核心模块设计"/"## 网关机制"段落中深挖。

> 视角：整个项目是我的。从全局视角梳理 SangBridge 的部署、模块、数据流。这部分用于建立**架构骨架**，具体模块细节在后续"## 核心代码"/"## 调试记录"等段落中深挖。

### 部署架构
**核心路径**（粗实线/红色节点）：用户 → VIP → 当前持有节点 WAF → SangBridge → VcsClient → phxoss VCS
**辅助路径**（细线/普通色）：SangBridge → UnivSrvdClient → univ-srvd；SangBridge → Mongo / cluster-manager
**说明**：粗实线/红节点 = 核心价值路径（高可用连续访问 + 凭据不暴露 + 身份贯通 + 上游协议隔离）；细线 = 辅助路径（UniView 能力、统一运维）


```mermaid
graph TB
    Browser[浏览器] -->|HTTPS :4431| VIP{EDS 集群管理 VIP<br/>phxha 漂移}
    VIP -.-> NodeA[节点 A 当前持有]
    VIP -.-> NodeB[节点 B 备用]
    VIP -.-> NodeC[节点 C 备用]

    NodeA --> WAF_A[sangfor_waf<br/>Nginx 反代]
    NodeB --> WAF_B[sangfor_waf]
NodeC --> WAF_C[sangfor_waf]

    WAF_A -->|127.0.0.1:7200| SB_A[SangBridge<br/>systemd :7200]
    WAF_B -->|127.0.0.1:7200| SB_B[SangBridge]
    WAF_C -->|127.0.0.1:7200| SB_C[SangBridge]

    SB_A -.->|127.0.0.1:18080| UniView[univ-srvd Go]
    SB_B -.-> UniView
    SB_C -.-> UniView

    SB_A -.->|127.0.0.1:7480| Phxoss[phxoss tentacle / RGW]
    SB_B -.-> Phxoss
    SB_C -.-> Phxoss

    SB_A --> Mongo[(EDS MongoDB<br/>eds 副本集 3 节点<br/>sangbridge DB)]
    SB_B --> Mongo
    SB_C --> Mongo

    SB_A -.->|只读| CM[cluster-manager.user]
    SB_B -.-> CM
    SB_C -.-> CM
```

**部署关键设计权衡**：

| 维度 | 现状 | 设计意图 |
|---|---|---|
| 部署形态 | 宿主机 RPM + systemd（后端 Wheel 装在 Python 3.11 site-packages） | 跟 EDS 节点同生命周期，5.3.2 起已不再用 chroot |
| HA 方式 | phxha 漂移 VIP | 被动热备，**非**主动-主动负载均衡；正常时一个 VIP 对应一个承载节点 |
| 流量入口 | sangfor_waf（Nginx，仓库外） | TLS + WAF 规则 + 反代 + 静态资源；WAF 配置由 EDS 基础设施维护 |
| 端口暴露 | SangBridge 仅监听 127.0.0.1:7200 | 实际入口和安全边界都是 WAF；外部不能直连 SangBridge |
| 集群规模 | 随 EDS 节点数 | 无 SangBridge 专属固定副本数或自动扩缩容 |
| 重配置机制 | cluster-manager 遍历 hosts → `systemctl restart sangbridge.service` | 节点加/减时自动同步 |
| 跨节点用户体验 | "节点 A 登录 → VIP 切到 B → 同 token 继续可用" | session 不存本机，依赖共享 Mongo 恢复身份 |

### 模块拆分
**核心模块**（粗实线/红色）：VCS 模块（含 VcsClient）
**辅助模块**（细线/普通色）：UniView 模块（含 UnivSrvdClient）、ClusterManager 模块
**说明**：粗实线/红节点 = 直接服务核心价值"统一入口 + 身份贯通 + 上游协议隔离"的主路径模块；细线 = 价值衍生模块（管理面、扩展能力）


```mermaid
graph TB
    subgraph Frontend["SangBridge Web (Vue 3 SPA, /var/www/sangbridge)"]
        UI[控制台 UI]
        Store[Pinia Stores<br/>repoPermissions / ...]
        API[web/src/api/...]
    end

    subgraph Backend["SangBridge Server (Python 3.11 + FastAPI :7200)"]
        subgraph Middleware["中间件链"]
            WafFilter[WafFilterMiddleware<br/>验 WAF 签名/XFF]
            RateLimit[TokenBucket 中间件<br/>进程内限流]
            Audit[Audit 中间件<br/>写 Mongo async]
        end

        Router[Router 层]
        Identity[get_identity 依赖]

        subgraph Modules["业务模块"]
            VCS_M[VCS 模块<br/>module/storage/vcs/...]
            UV_M[UniView 模块<br/>module/uniview/...]
            CM_M[ClusterManager 模块]
        end

        subgraph Clients["上游 Client"]
            VcsClient[VcsClient<br/>SigV4 + S3 + 目录桶]
            UnivSrvdClient[UnivSrvdClient<br/>HTTP+JSON]
        end

        subgraph Core["基础组件"]
            TokenRepo[Token Repository<br/>sangbridge.tokens]
            AuditRepo[Audit Repository<br/>sangbridge.audit_logs<br/>TTL 30d]
            SysLog[SysLogHandler local6<br/>→ EDS syslog]
        end
    end

    UI --> WafFilter
    WafFilter --> RateLimit --> Audit --> Router
    Router --> Identity
    Router --> VCS_M
    Router --> UV_M
    VCS_M --> VcsClient
    UV_M --> UnivSrvdClient
    VcsClient -.->|127.0.0.1:7480| Phxoss[phxoss VCS]
    UnivSrvdClient -.->|127.0.0.1:18080| UniView[univ-srvd]
    VCS_M --> TokenRepo
    Audit --> AuditRepo
    VCS_M --> SysLog
    TokenRepo --> Mongo[(MongoDB sangbridge)]
    AuditRepo --> Mongo
```

**模块关键点**：

- **每个上游一个专用 client**（不是统一泛化 client）—— 协议和认证差异太大（VCS 要 SigV4/S3，UniView 只要 JSON 头）
- 中间件链是**洋葱结构**：`WafFilter → RateLimit → Audit → Router`
- **限流/审计/并发 limiter 都是进程内的**——不跨节点共享（关键设计权衡）
- 前端 Pinia Store 做权限缓存（按 `repoId`），仓库切换重新加载

### 关键数据流：登录 → 调 VCS
**核心路径**（实线）：登录 → SangBridge 身份复核 → 调 VCS（带 SigV4）
**辅助路径**（虚线）：调 UniView（X-IFGW-User 头）—— 跟 VCS 链路相似但认证机制不同
**说明**：实线 = 价值驱动优先级 P0 路径（凭据不暴露 + 身份贯通 + 高可用连续访问）；虚线 = 价值衍生路径（管理面能力）


```mermaid
sequenceDiagram
    autonumber
    participant B as 浏览器
    participant W as sangfor_waf
    participant SB as SangBridge
    participant M as MongoDB
    participant CM as cluster-manager
    participant P as phxoss VCS

    rect rgb(245,245,245)
        Note over B,P: ① 登录
        B->>W: POST /login (用户名+密码)
        W->>SB: 转发 + WAF 签名头
        SB->>M: 写 sangbridge.tokens (新 token)
        SB->>CM: 读 cluster-manager.user (arn, AK, AES-CBC(SK))
        SB-->>W: Set-Cookie sb_auth (HttpOnly)
        W-->>B: 登录成功
    end

    rect rgb(245,245,245)
        Note over B,P: ② 业务请求（创建分支）
        B->>W: POST /api/.../vcs/repos/{id}/branches + sb_auth cookie
        W->>SB: 转发 + WAF 签名头 + XFF
        SB->>M: 查 sangbridge.tokens (验 cookie)
        SB->>CM: 读 cluster-manager.user (验凭据指纹)
        SB->>SB: 运行时 AES-CBC 解密 SK<br/>组装 Identity(arn, AK, SK)
        SB->>P: AWS SigV4 签名 + X-User-Arn
        P-->>SB: 200
        SB-->>W: 200
        W-->>B: 200
    end

    rect rgb(245,245,245)
        Note over B,P: ③ VIP 漂移场景
        Note right of SB: phxha 切到节点 B<br/>新 SangBridge 仍能从共享 Mongo 恢复身份
    end
```

**身份传递关键点**：

- 浏览器只拿 `sb_auth` HttpOnly cookie，**永远不接触** ARN/AK/SK
- 每次请求 SangBridge 从 Mongo `sangbridge.tokens` 查 cookie
- 再回查 `cluster-manager.user` 验凭据指纹（密码/AK/SK 轮换**立即生效**）
- 运行时 AES-CBC 解密 SK → 内存中的 Identity(arn, AK, SK)
- 调 VCS 用 **AWS SigV4**（service=s3）+ 审计关联头 `X-User-Arn: <plain ARN>`
- 调 UniView 用 `X-IFGW-User: base64(JSON({user_arn, is_admin}))`（**不**是 X-User-Arn 旧协议）
- 调 VCS 时若用户无 AK/SK → 返回 `VCS-CREDENTIALS-MISSING`，**不降级**为服务账号

### 关键中间件与外部依赖

| 类别 | 实现 | 备注 |
|---|---|---|
| 数据库 | MongoDB（EDS 副本集 3 节点） | sangbridge 库存 `tokens` + `audit_logs`；只读 `cluster-manager.user` |
| 缓存 | **无** | 进程内限流、目录桶凭据缓存都是单机 |
| 消息队列 | **无** | 审计是进程内异步写 Mongo |
| 日志 | Python `logging` → `SysLogHandler(local6)` → EDS syslog | 落盘 `/sf/log/today/sangbridge.log` |
| 监控 | **无** Prometheus / Grafana 接入 | 故障定位纯靠日志 |
| WAF | `sangfor_waf`（Nginx，仓库外） | TLS + WAF 规则 + 反代；WAF 配置在 EDS 基础设施 |
| 应用内 WAF 防御 | `WafFilterMiddleware` | 验证 `Y-Forwarded-For`/`Y-HMAC-For`，只信任受信代理发来的 XFF |
| 限流 | 自研 FastAPI 中间件 + 进程内 Token Bucket | 用户桶 300 容量/秒补 10；IP 桶 600 容量/秒补 20；上传端点独立大桶 |
| 审计 | 自研中间件，异步写 Mongo `audit_logs` | TTL 默认 30 天；**默认 `audit.enabled=false`** |
| 并发保护 | 进程内 VCS 上传/下载/ZIP limiter | 非分布式 |
| `/metrics` | 暂无 | 监控接入缺失点 |

### 上游服务连接

| 上游 | 默认地址 | 协议 | 认证方式 | Client |
|---|---|---|---|---|
| univ-srvd (UniView) | `127.0.0.1:18080/api/v1` | HTTP + JSON REST | `X-IFGW-User: base64(JSON)` | `UnivSrvdClient` |
| phxoss tentacle / RGW (VCS) | `127.0.0.1:7480` | HTTP；控制面 VCS REST + 数据面 S3 兼容 | **AWS SigV4** (service=s3) | `VcsClient` |
| cluster-manager | 内部 | 只读 user 表 | — | 直接 Mongo 读 |

**关键设计点**：

- 默认本机 loopback 共置，地址都通过 `upstream.*.base_url` 可覆盖
- 没有 gRPC，没有"内部 SDK 嵌入上游"——用 `httpx.AsyncClient` 连接池发真实 HTTP
- 多上游分别用专用 client，因为协议和认证差异大；配置统一在 `upstream` 模块
- UniView 各领域模块（Catalog/Search/Source/Promote）复用同一个 app 级 `UnivSrvdClient`
- `VcsClient` 同时覆盖 VCS 控制面 REST 与 S3 数据面（统一处理 SigV4、对象流式、Multipart、目录桶 Session）
- 目录桶（`--x-s3` 后缀）特殊流程：用户 IAM AK/SK 签 `GET /{bucket}?session` → 拿短期凭据 → 缓存 5 分钟 / LRU 1024 → 后续用 S3ExpressAuth 签

### 关键架构薄弱点（按价值驱动重排）

> 排序依据见 [[#价值驱动的优先级]]：**威胁核心价值 = P0，影响但非核心 = P1，演进/细节问题 = P2**。

#### P0（直接威胁核心价值）

1. **AK/SK AES-CBC 加密密钥管理** — 加密密钥在哪？谁解密？轮换策略？失守的爆炸半径需要评估。威胁**凭据不暴露 + 身份贯通**（核心 P0 价值），失守=整套 SigV4 体系崩溃。
2. **审计默认关闭** — `audit.enabled=false` 是默认生产配置；写操作无痕，安全事件无追溯依据。威胁**安全边界统一 + 失败成本（事件追溯）**。
3. **session 不存本机的性能瓶颈** — 每次请求都回查 Mongo `tokens` + `cluster-manager.user`；高并发下 Mongo 可能成瓶颈。威胁**高可用连续访问**（核心 P0 价值）。

#### P1（影响但非核心）

4. **限流不跨节点** — 进程内 Token Bucket，VIP 漂移后用户桶重新计数；攻击者通过 VIP 漂移可绕过单节点限流（用户桶 300/s × N 节点 = N×300/s）。影响**安全边界统一**。
5. **VIP 漂移时旧请求处理** — 节点切换瞬间在飞请求会失败，重试策略？用户体验？影响**高可用连续访问**。

#### P2（演进/细节问题）

6. **WAF 在仓库外** — 改 WAF 配置（签名密钥、超时、规则）会绕过 SangBridge CI，需要独立变更流程。
7. **没 Redis** — 所有"分布式"功能（限流、缓存、幂等、防重）都缺基础组件，影响**快速接入新上游**。
8. **没 Prometheus** — 故障时无指标可查；纯靠 `/sf/log/today/sangbridge.log` 定位。影响**统一运维与治理**。
9. **S3 Express Session 缓存** — 5 分钟 LRU 1024 桶，过期边界、凭据轮换兼容性、与目录桶快速删除的兼容性都需要验证。
10. **CI/CD 流程不清晰** — 笔记里提"CI 镜像"但没流程；后端 RPM 构建、前端 build、EDS 节点部署的链路需要梳理。


### 事实边界更新（按"整个项目是我的"视角）

> 原"## 事实边界"段落是 **VCS 视角**的事实边界（哪些工作是我的）。从"整个项目"视角看，那些边界变成了"需要深入理解的模块"。本节列出**架构层面**的事实边界。

| 旧笔记内容 | 实际情况 | 备注 |
|---|---|---|
| `X-User-Arn: <plain ARN>` 用于 UniView | **旧协议已废弃**；当前用 `X-IFGW-User` (Base64 JSON) | VCS 仍用 `X-User-Arn`（审计关联头，不是授权主体） |
| "登录 Token WAF 审计 SigV4/S3 Express 不计入个人成果" | 从"整个项目是我的"视角，这些**都是 SangBridge 的一部分**，要全部搞懂 | 之前是 VCS 视角的边界；新视角下边界变成"需要深入理解的模块" |
| 默认生产 `audit.enabled=false` | 笔记没提——**审计默认关闭** | 架构薄弱点 #2 |
| WAF 在仓库外 | 笔记没提 | 架构薄弱点 #3 |
| 限流不跨节点 | 笔记没提——进程内 Token Bucket | 架构薄弱点 #1 |
| 没 Redis / 消息队列 / Prometheus | 笔记没提 | 架构薄弱点 #6 #7 |
| 凭据解密 / AES-CBC 密钥管理 | 完全空白 | 架构薄弱点 #5 |
| CI/CD 流程 | 笔记里提"CI 镜像"但没流程 | 架构薄弱点 #10 |

## 核心模块设计

> 视角：从整个项目设计意图出发，讲清每个模块"是什么 / 为什么这样设计 / 关键设计点"。

### VCS 模块

VCS 模块是 SangBridge 管控面最核心的模块之一，对接 phxoss 数据面（边界外），提供仓库、分支、标签、提交、MR、S3 对象存储的访问能力。

#### 核心职责

- **VCS 控制面 REST**：仓库、分支、标签、提交、MR、对比的 VCS REST 调用（按 `phxoss /vcs/v1/*` 接口）
- **数据面 S3 兼容**：对象上传下载、Multipart、目录桶 Session（通过 phxoss tentacle / RGW，S3 兼容 API）
- **权限治理子模块**：消费上游 `/permissions` 权限集合，做"按钮能力 → 请求前校验 → 上游最终授权"的分层
- **游标分页子模块**：统一 `truncated/after`、opaque cursor、S3 continuation token 的消费语义

#### 权限治理子模块（核心机制）

**设计目标**：让控制台状态与数据面策略一致，提前阻止误操作和无效请求；安全边界仍是 phxoss Bucket Policy。

**完整调用链**：

```text
ResourceDetail(route.params.id)
  → repoPermissionsStore.load(repoId)
  → repoApi.permissions(repoId)
  → GET /api/sangbridge/v1/vcs/repos/{repoId}/permissions
  → RepoMgr / VcsClient
  → GET phxoss /vcs/v1/repos/{repoId}/permissions
  → Action、Button、Method+URL 权限集合
  → useRepoPermissions 生成 canCreate/canDelete/canGetDiff 等能力
  → 按钮置灰 + 部分 Handler 请求前拦截
  → SangBridge 转发用户身份，phxoss Bucket Policy 最终授权
```

**关键设计点**：

- SangBridge VCS Router 用 `Depends(get_identity)` 取得登录身份，以 `X-User-Arn` 与 SigV4 签名向 phxoss 转发
- phxoss 权限响应含 `bucket_name`、`user`、`is_owner`、`apis[]`；前端按 IAM Action / Button 标识 / Method+URL 三种方式匹配
- 权限数据按 `repoId` 缓存，仓库切换会重新加载
- **权限四类条件**共同决定：上游权限集合 + 当前仓库角色 + 资源创建人归属 + 页面业务条件
  - `admin`：可操作任意创建人资源
  - `developer`：只能操作 `creator_id == 当前用户名` 的资源
  - `visitor`：不允许写操作；角色未加载时按不可操作处理
  - 业务条件：默认分支不可删、源/目标必须都是分支、源分支归属不明确禁提 MR

**前端涉及页面**：`BranchTab.vue`、`TagTab.vue`、`CommitTab.vue`、`CompareTab.vue`、`NewPRModal.vue`、`PRApprovalModal.vue`

**前端关键代码路径**：

- `web/src/views/vcs/ResourceDetail.vue`
- `web/src/stores/repoPermissions.ts`
- `web/src/composables/useRepoPermissions.ts`
- `web/src/api/vcs/index.ts`

**后端关键代码路径**：

- `server/sangbridge_server/module/storage/vcs/repo/web_api/router.py`
- `server/sangbridge_server/module/storage/vcs/repo/manager/repo_mgr.py`
- `server/sangbridge_server/client/phxoss/vcs/vcs_client.py`

#### 游标分页子模块（核心机制）

**设计目标**：以后端 `truncated/after` 为唯一可信源，前端不臆测下一页；不同分页语义（普通 after、opaque cursor、S3 continuation token）需要清晰区分。

**核心契约**：

- **普通 cursor**（仓库/分支/标签列表）：`truncated: bool` + `after: str` —— 前端用 `after` 当下一次请求参数
- **opaque cursor**（提交图/树形结构）：后端返回 base64 编码的不透明 cursor，前端**不能**解析
- **S3 continuation token**（快照文件树）：从 S3 ListObjectsV2 返回的 `NextContinuationToken`，不能混淆为普通 cursor

**关键修复点**（机制层面）：

- **exact-page 假下一页**：之前前端用 "current + 1" 推算下一页，遇到 truncated=true 但 `after` 不前进时就翻错。修复：以后端返回的 `after` 为准
- **S3 快照 token 错用**：之前用普通 `after` 当 S3 token 传给 S3 API。修复：明确区分两类 cursor
- **仓库超过 50 条无法继续**：默认 page size 设了硬上限。修复：分页契约统一到 `truncated/after`，无硬上限
- **快照漏数据**：游标构造错位导致跳页。修复：cursor 构造在 client 层封装，调用方不直接处理

**后端测试覆盖**（涉及 py 集成测试）：

- `server/tests/integration/vcs/test_branch_router.py`
- `server/tests/integration/vcs/test_tag_router.py`
- `server/tests/integration/vcs/test_commit_router.py`
- `server/tests/integration/vcs/test_workspace_router.py`
- `server/tests/unit/vcs/test_workspace_mgr.py`
- `server/tests/unit/client/test_vcs_client_coverage.py`

**已知薄弱点**：仓库 Router `GET /repos?limit=&after=` 集成测试；后端 exact-page 契约；空页但 `truncated=true`、`after` 不前进等异常契约；三页以上翻页和回退；仓库切换后的旧请求覆盖；搜索与翻页并发；S3 token 失效/变化；以及上游重叠数据的去重策略。

#### AK/SK 加密管理（已深入理解，P0 薄弱点 #1）

> 详见 [[#关键架构薄弱点]] #1。这部分把"为什么是 P0"展开成具体机制 + 风险 + 应急链路。

**密钥存储链路**：

```text
/sf/etc/eds_secrets.yml  (encryption.aes_cbc_key: <16-byte AES-128>)
  → /usr/bin/phxcp_runtime get encryption.aes_cbc_key
  → /sf/bin/sangbridge_env_wrapper.sh
  → export SANGBRIDGE_EDS_AES_CBC_KEY=<key>
  → exec /usr/local/bin/sangbridge-server
  → service 启动期读取环境变量
  → load_runtime_secrets() 强制校验 16 字节 ASCII
  → AuthProvider 解密 cluster-manager.user.keys[*].secret_key
```

**关键事实**：

- 密钥通过 systemd 的 wrapper 注入到进程环境变量，**不写入** `/etc/sangbridge/config.yml`、数据库或应用日志
- 启动时强制 16 字节校验，缺失/长度不对**直接启动失败**
- SK **不是**启动时一次性解密，而是**每次受保护请求都动态解密**——登录只计算 user.keys 指纹
- 解密后的 SK 只存在当前请求的进程内存，**不写回** `sangbridge.tokens`
- AES key 缺失或解密失败 → Identity 中 AK/SK 为空 → VCS 返回 `VCS-CREDENTIALS-MISSING`

**轮换策略**：

- **不定期轮换**——被动检测 + 立即使旧登录态失效
- 触发：外部修改 `user.keys[]` → SangBridge 下次请求立即发现
- 范围：单个 `cluster-manager.user.keys[]` 为单位
- 轮换后该用户 `sb_auth` token 被删除，响应 `401 auth_credentials_rotated`，必须重新登录
- **没有双凭据平滑期**——SangBridge 不实现"旧、新凭据同时被同一登录态接受"
- **安全轮换操作顺序**（强制重新登录换取安全）：
  1. 在 RGW / cluster-manager 侧创建新 AK/SK
  2. 更新该用户的 `keys[]`，确保新 key 是 SangBridge 将优先选取的第一把有效 key
  3. 预期该用户所有 SangBridge 会话被强制重新登录
  4. 确认新登录后的 VCS 操作成功
  5. 再撤销旧 key
- **不混为一谈**：平台 `phxcp_runtime rekey` 轮换的是本机 runtime 配置文件的保护密钥（AES-256-GCM），**不会**自动改变 `encryption.aes_cbc_key` 明文值，**也不会**重加密 Mongo 里的 `user.keys[].secret_key`

**失守的爆炸半径**：

- 一个节点 root 失守 = **高概率"全体用户 SK 的解密能力泄漏"**（不是只影响该节点）
  - 原因：`cluster-manager.user` 在集群共享 Mongo；任意接管 VIP 的 SangBridge / cluster-manager 节点必须能解同一批 `keys[]`
  - `/sf/etc/phxcp_runtime.key` 每节点不同，但 `encryption.aes_cbc_key` 必须是集群内一致（否则各节点解不出同一份共享数据）
- **检测能力不足**（应假设无告警下已失守）：
  - 没有密钥访问审计 / FIM、HSM / KMS 取钥审计、`phxcp_runtime get` 异常告警、SIEM / EDR 联动
  - 现有可观测性（请求日志、进程存活、IP 不匹配、`user.keys` 变更、签名请求审计）只能发现异常使用，**不能可靠发现"攻击者已复制 AES key"**
  - **应将节点 root 失守视作高优先级安全事件**，不能等 SangBridge 自己报警
- 现有 rekey ≠ 轮换用户 SK 解密 key——仅 `phxcp_runtime rekey` 不足以止血

**应急链路（仓库无完整 runbook，需平台/运维/对象存储团队协同）**：

1. 发现/确认节点失守
2. 隔离受侵节点、摘除 VIP / 停止相关服务、保全证据
3. 评估 Mongo 与运行时密钥是否也已暴露
4. 轮换受影响用户的 RGW AK/SK，并撤销旧 AK/SK
5. 生成新的 `encryption.aes_cbc_key`
6. 使用旧 key 解密、使用新 key 重加密所有受影响的 `user.keys[].secret_key`
7. 将新的运行时配置安全分发到各节点
8. 各节点执行 `runtime rekey`，重启 cluster-manager / SangBridge 等依赖方
9. 吊销 SangBridge token、验证新登录与 VCS SigV4
10. 复盘并监控旧 AK/SK 是否还有使用痕迹

**应急处置对旧 sb_auth cookie 的影响**：

| 处置动作 | 旧 sb_auth cookie 是否仍可用 |
|---|---|
| 仅重启 SangBridge / VIP 漂移 | ✓ 可继续使用（token 在共享 Mongo） |
| 只执行 runtime rekey，业务 CBC key 不变 | ✓ 通常可继续使用；但**不足以应对** AES-CBC key 已泄漏 |
| 修改任一用户 `keys[]` | ✗ 该用户下次请求即 `401 auth_credentials_rotated`，必须重登 |
| 全员重发 AK/SK 或全量重加密后 `keys[]` 改变 | ✗ 所有活跃用户会被迫重新登录 |
| 主动删除/吊销 `sangbridge.tokens` | ✗ 对应 cookie 立即失效，必须重登 |

**关键洞察**：在完成上游 RGW 旧 AK/SK 撤销前，**攻击者已复制的旧 AK/SK 仍可能直接对 phxoss 有效**；SangBridge token 失效本身**不能撤销被盗的底层对象存储凭据**。

**遗留薄弱点**（已识别但当前未实现）：

- 仓库无完整自动化的"一键全局 AES-CBC key 轮换" runbook 或演练记录
- 关键路径（"批量重加密 Mongo 用户 SK、分发新 CBC key、全局 token 吊销"）在当前仓库里没看到完整自动化
- 节点 root 失守无直接检测能力（应假设无告警下已经失守，需主动监控 + 演练）



### UniView 模块

> **状态**：待补全——目前掌握信息有限（只通过 UnivSrvdClient 接入）。需要后续梳理：

- 内部领域模块（Catalog / Search / Source / Promote）的具体职责
- 与 VCS 模块的差异（身份头 vs SigV4）
- 数据流转：SangBridge 业务操作 → UnivSrvdClient → univ-srvd 各领域服务

### ClusterManager 模块

> **状态**：待补全——目前只通过只读 user 表接入。需要后续梳理：

- sangbridge 对 cluster-manager 的接口范围
- EDS 集群重配置逻辑
- 用户/凭据/ARN 的生命周期管理

## 网关机制

> 视角：SangBridge 作为 FastAPI 网关，对外统一控制台流量入口，对内转发到上游服务。这一节讲清 SangBridge 网关的核心机制设计。

### 中间件链（洋葱结构）

请求按以下顺序穿过中间件：

```text
HTTP 请求
  → WafFilterMiddleware（验 WAF 签名/XFF）
  → TokenBucket 中间件（进程内限流）
  → Audit 中间件（写操作异步写 Mongo audit_logs）
  → Router 层（业务处理）
```

### 身份与转发

- `Depends(get_identity)` 注入登录身份（从 `sb_auth` cookie → sangbridge.tokens → cluster-manager.user）
- 上游转发用各 client 自己的认证机制：
  - VCS：AWS SigV4（service=s3），运行时 AES-CBC 解密 SK
  - UniView：`X-IFGW-User: base64(JSON({user_arn, is_admin}))`
  - ClusterManager：只读 Mongo，无认证转换
- 凭据轮换**立即生效**（每次请求都回查 user 表验指纹）

### 限流

- **自研 FastAPI/Starlette 中间件 + 进程内 Token Bucket**
- 默认阈值：用户桶 300 容量/秒补 10；IP 桶 600 容量/秒补 20
- 上传端点独立大桶
- **架构权衡**：进程内实现，**不跨节点共享**——VIP 漂移后用户桶重新计数（见[[#关键架构薄弱点]]#1）

### 审计

- 自研中间件，**写操作**（POST/PUT/PATCH/DELETE）异步写 Mongo `sangbridge.audit_logs`
- TTL 默认 30 天
- **架构权衡**：**默认 `audit.enabled=false`**——生产需运维显式打开（见[[#关键架构薄弱点]]#2）

### WAF 防御

- **外层 WAF**：`sangfor_waf`（Nginx，仓库外）承担 TLS + WAF 规则 + 反代
- **应用内 WAF 防御**：`WafFilterMiddleware` 验证 WAF 签名的 `Y-Forwarded-For`/`Y-HMAC-For`，只信任受信代理发来的 XFF
- WAF `/open/api/` 代理超时 6000 秒

### 访问日志

- 每个 HTTP 请求由 `WafFilterMiddleware` 记录 access log
- 走 Python `logging` → `SysLogHandler(local6)` → EDS syslog → `/sf/log/today/sangbridge.log`
- **注意**：访问日志 ≠ 审计日志——访问日志是每个请求，审计只针对写操作

## 项目边界

按"整个项目是我的"视角，**唯一边界**是：

- **phxoss VCS 上游是数据面**（由 phxoss 团队负责）
  - SangBridge 通过 VcsClient 消费 phxoss 的 VCS REST API 和 S3 兼容 API
  - phxoss 的 Bucket Policy 是 SangBridge VCS 模块的最终授权边界
  - 权限契约（`/permissions` API、IAM Action 等）由 phxoss 团队定义，SangBridge 适配
  - phxoss 的 RGW 改造、tentacle 服务、底层存储等都不在 SangBridge 边界内

**其他都是 SangBridge 的一部分**——包括之前 VCS 视角下被划为"不计入个人成果"的部分：

- 登录、Token 管理（属于 SangBridge 网关层）
- WAF 签名验证（`WafFilterMiddleware` 在 SangBridge 仓库内）
- 限流、审计（自研中间件在 SangBridge 仓库内）
- SigV4 签名、S3 Express Session 缓存（`VcsClient` 在 SangBridge 仓库内）
- UniView 协议适配（`UnivSrvdClient` 在 SangBridge 仓库内）
- cluster-manager user 表的只读访问（属于 SangBridge 数据访问层）

**EDS 基础设施**（不在 SangBridge 仓库内）：

- sangfor_waf / NGSWAF 配置
- EDS MongoDB 副本集
- phxha VIP 漂移
- EDS syslog 落盘


## 关联知识

- [[项目知识地图]]
- [[Linux知识地图]]
- [[FastAPI与ASGI后端框架]]
- [[鉴权与Token方案]]
- [[后端接口设计与分页]]
