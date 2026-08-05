---
tags: [python, fastapi, asgi, backend, web]
created: 2026-08-04
updated: 2026-08-04
status: 待学习
---

# FastAPI 与 ASGI 后端框架

## 一句话理解

FastAPI 是基于 Python 类型注解的 ASGI Web 框架，把数据校验（Pydantic）、依赖注入、OpenAPI 文档开箱集成，适合做 API 网关、控制面服务和 BFF。

## 核心原理

### ASGI 与 WSGI

- WSGI 是同步接口（每请求一个调用，阻塞模型）。
- ASGI 是异步接口：应用接收 `scope`（请求元数据）与 `receive`/`send` 两个异步函数，可在事件循环内处理并发请求，也支持 WebSocket/流式响应。
- Uvicorn 是 ASGI 服务器，负责接收 socket 连接并把 HTTP 消息转成 ASGI 调用。

### FastAPI = Starlette + Pydantic

- Starlette 提供路由、中间件、生命周期、静态文件、WebSocket 等核心能力。
- Pydantic（v2 用 Rust 核心）做请求体校验、序列化，类型注解同时生成 OpenAPI schema。
- FastAPI 在启动时根据路由与模型自动生成 `/docs` 与 `/openapi.json`。

### 依赖注入

- `Depends()` 声明依赖，FastAPI 在请求作用域内解析并注入。
- 依赖可以嵌套、参数化（`Depends(require_permission("vcs:GetDiff"))`），返回值参与注入与 OpenAPI。
- `use_cache` 默认为 True：同一请求内多次声明同一依赖只实例化一次。

## 底层实现

- 请求生命周期：Uvicorn 解析连接 -> 中间件链 -> 路由匹配 -> 依赖解析 -> 视图函数 -> 响应。
- 中间件分两种：Starlette 的 `BaseHTTPMiddleware` 与纯 ASGI 中间件（直接操作 scope/receive/send）；顺序按注册顺序嵌套，请求进入正序、响应返回逆序。
- `async def` 视图在事件循环中执行；普通 `def` 视图 FastAPI 自动用线程池（`run_in_threadpool`）执行，避免阻塞循环。
- 生命周期事件（lifespan）在应用启动/关闭时执行，适合初始化连接池、加载配置。

## 常见应用场景

- [[SangBridge AI统一接入与管理控制台]]：Vue 控制台 -> FastAPI 网关 -> phxoss 等上游；网关负责登录身份、请求转发、错误转换、审计与限流。
- 身份注入：VCS Router 用 `Depends(get_identity)` 取得登录身份，再通过 `X-User-Arn` 与用户凭据签名向 phxoss 转发。
- 统一错误模型：把上游错误码/异常转换成对外稳定契约，前端只依赖一种错误结构。
- 控制面管理 API：Kerberos 域管理、健康检查、任务状态查询等。

## 容易踩坑的地方

- async 路径里做同步阻塞（见 [[Python并发与asyncio]]）。
- 模块级可变全局状态：多请求/多 worker 下不是线程安全的“共享缓存”，要显式设计。
- 依赖里做重量级初始化：应放 lifespan，而不是每个请求都建连接。
- 中间件里吞异常或忘记 `await call_next`，响应时序会乱。
- 认证依赖只取身份不做授权，或信任客户端传来的 `X-User-*` 头而不清洗——网关必须自己注入并覆盖同名字段。
- 默认文档会把接口与参数暴露出去，生产环境要关闭或鉴权。

## 面试高频问题

1. ASGI 和 WSGI 的区别？FastAPI 为什么用 ASGI？
2. 依赖注入是怎么实现的？`use_cache` 是什么意思？
3. `async def` 与普通 `def` 视图的区别？
4. 中间件的执行顺序？如何在响应阶段统一处理错误/审计？
5. FastAPI 与 Flask/Django 的定位差异？
6. 网关转发用户身份时，如何防止客户端伪造身份头？

## 关联知识

- [[Python并发与asyncio]]
- [[后端接口设计与分页]]
- [[鉴权与Token方案]]
- [[SangBridge AI统一接入与管理控制台]]
