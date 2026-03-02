---
description: Code Review 当前分支改动或指定文件，输出结构化评审意见（P0/P1/P2）
---

对当前改动（或指定文件/代码段）做 Code Review，输出结构化评审意见。

## 执行步骤

1. 如果用户传入了文件路径或代码片段，针对该内容 review；否则用 `git diff develop...HEAD` 获取当前分支相对 develop 的全部改动。
2. 阅读相关上下文（被修改函数的调用方、接口定义、相关测试等），理解改动意图。
3. 按以下项目规范逐项检查，输出评审意见。

---

## 输出格式

### 总体评价
一句话概括改动质量。

### 问题清单

每条格式：**[P0/P1/P2]** `文件名:行号` — 问题描述，建议修改方式。

- **P0**：Bug / 安全漏洞 / 数据正确性，必须修复
- **P1**：逻辑缺陷 / 边界条件 / 性能，强烈建议修复
- **P2**：代码风格 / 可读性，可选

### 亮点（可选）

---

## 项目专属 Checklist

### 异常处理

- **必须用项目异常体系**，不得抛 `RuntimeException` / `IllegalArgumentException`：
  - `BellaException.AuthorizationException` → 401，AK 不存在/无权限
  - `BellaException.RateLimitException` → 429，QPS/月额度超限
  - `BellaException.ChannelException` → 上游供应商错误
  - `BizParamCheckException` → 400，业务参数校验失败
  - `ResourceNotFoundException` → 404
- **@EndpointAPI vs @BellaAPI 不得混用**：
  - `/v*/chat/completions` 等 AI 能力接口用 `@EndpointAPI`，响应为 OpenAI 格式
  - `/console/**` 等管理接口用 `@BellaAPI`，响应为 `BellaResponse<T>`
  - 两套 ResponseAdvice 分别处理，交叉会导致格式错误
- **日志级别**：5xx 用 `log.warn(msg, e)` 含 stacktrace；4xx/401 用 `log.info`
- **try-catch 不得吞异常**：catch 后必须 rethrow 或有明确降级逻辑

### 并发计数生命周期

- `limiterManager.incrementConcurrentCount()` 的 decrement 依赖 `EndpointLogger.log()` 异步触发（通过 Disruptor → `LimiterLogHandler`）
- **凡新增的 Controller 方法**，若有 increment，其响应必须经过 `EndpointResponseAdvice`（即不能直接操作 `HttpServletResponse` 绕过 ResponseBodyAdvice），否则 concurrent count 永不 decrement，造成计数泄漏

### 缓存规范

- `@Cached` 的 `name` 必须用 `private static final String` 常量；`key` 多段拼接必须加 `':'` 分隔（防 key 碰撞）
- `cacheType = CacheType.BOTH` 时必须设 `syncLocal(true)`，否则多实例本地缓存不一致
- `@CacheInvalidate` 的 `name`/`key` 必须与对应 `@Cached` 完全一致（包括末尾冒号）
- 高频查询且可能返回 null 的方法必须设 `cacheNullValue(true)` + `penetrationProtect(true)`

### 数据库访问

- **Service 层不得出现 `DSLContext`**，数据库操作只能在 `db/repo/` 包下
- 多步写操作事务需 `@Transactional(rollbackFor = Exception.class)`（默认只 rollback `RuntimeException`）
- insert 必须走 `UniqueKeyRepo.insert()`，不得裸调 `db.insertInto()` —— 否则 creator/updator 信息不填充
- Service 内自调用（self-invocation）要走 `applicationContext.getBean(XxxService.class).method()` 代理，否则事务/缓存注解不生效

### API 设计

- 分页接口必须返回 `Page<T>`，不得自定义分页结构
- 参数校验优先级：`@Valid` 注解 > `Assert.notNull/isTrue` > 手动 `if-throw`
- 约束注解要配合 `@Valid`，否则注解不生效

### 安全

- **AK 必须脱敏**：日志/异常 message 中出现 AK 相关字段必须先调 `EncryptUtils.desensitize()`，不得直接拼接明文
- **修改操作必须校验 ownerCode**（不只是检查资源存在）：参考 `ApikeyService.checkPermission()`
- 新接口若返回 `ChannelDB`，`channelInfo` 含供应商密钥，需评估是否应过滤或脱敏
- 鉴权逻辑集中在 `AuthorizationInterceptor`，Controller 用 `EndpointContext.getApikey()` 获取，不得自行解析 `Authorization` header

### 代码风格

- 字符串常量统一在 `EntityConstants` 定义，通过 `import static` 使用，不写魔法字符串（如 `"active"` `"person"` `"high"`）
- 多字段对象构建用 Builder（`@Builder`），不写大量 setter
- 不用 `parallelStream()`（项目内未使用）
- **TODO 必须关联 issue**，权限相关 TODO 不得合入主干（已知遗留：`ApikeyService.java:374,410`、`ModelService.java:335`、`ChannelService.java:103,126`，CR 时注意不要在同模块新增类似遗留）