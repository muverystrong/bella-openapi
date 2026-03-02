# 通道路由支持Endpoint维度 - 影响范围分析

## 1. 背景

### 1.1 问题描述

目前通道路由仅支持到**模型名称维度**，无法支持同一个模型在不同能力点使用不同的通道配置。

**当前限制：**
```
模型 (gpt-4) → 通道1, 通道2
```

**期望支持：**
```
/v1/chat/completions + gpt-4 → 实时通道
/v1/batches + gpt-4 → 批处理折扣通道
```

### 1.2 典型场景

1. **成本优化**：实时对话用高成本通道，批处理用50%折扣通道
2. **能力隔离**：streaming能力点用支持流式的通道，批处理用队列通道
3. **合规隔离**：不同能力点使用不同数据流向的通道

---

## 2. 影响范围详细分析

### 2.1 数据库层

#### 2.1.1 表结构变更

**channel表：**

| 变更类型 | 详细内容 | 影响程度 |
|---------|---------|---------|
| 新增字段 | `endpoint VARCHAR(128) DEFAULT NULL` | 高 |
| 新增索引 | `idx_endpoint_model_status (endpoint, entity_code, status)` | 高 |
| 保留索引 | `idx_type_code (entity_type, entity_code)` 用于兼容 | - |

**影响点：**
- DDL迁移脚本：`api/server/sql/22-20260309-channel-endpoint.sql`
- 回滚脚本
- 生产环境需要分批执行（避免锁表）

#### 2.1.2 jOOQ代码生成

**影响文件：**
- `api/server/src/codegen/java/com/ke/bella/openapi/tables/Channel.java`
- `api/server/src/codegen/java/com/ke/bella/openapi/tables/records/ChannelRecord.java`
- `api/server/src/codegen/java/com/ke/bella/openapi/tables/pojos/ChannelDB.java`

**影响评估：** 低（执行DDL后需要重新生成）

---

### 2.2 核心路由层

#### 2.2.1 ChannelRouter.java

**文件路径：** `api/server/src/main/java/com/ke/bella/openapi/protocol/ChannelRouter.java`

**影响方法：**

| 方法名 | 行号 | 影响类型 |
|-------|------|---------|
| `route()` | 46-82 | 重构 |
| `route()` (重载) | 235-241 | 重构 |
| `listAvailableChannels()` | 243-269 | 重构 |
| `filter()` | 100-149 | 修改 |
| 新增方法 | - | 新增 |

**影响说明：**
- 当前是二选一查询（有model查model，否则查endpoint）
- 需要改为三级降级查询：endpoint+model → 通用model → 纯endpoint
- 需要新增方法封装降级查询逻辑
- filter()方法可能需要调整参数

**影响评估：** 高（核心业务逻辑）

---

### 2.3 服务层

#### 2.3.1 ChannelService.java

**文件路径：** `api/server/src/main/java/com/ke/bella/openapi/service/ChannelService.java`

**影响方法：**

| 方法名 | 影响类型 |
|-------|---------|
| `listActivesByEndpointAndModel()` | 新增 |
| `listActivesByModel()` | 新增 |
| `listActives()` | 保留（兼容） |

**影响说明：**
- 需要新增2个查询方法
- 缓存key结构变化
- 现有方法保留用于向后兼容

**影响评估：** 中

---

### 2.4 数据访问层

#### 2.4.1 ChannelRepo.java

**文件路径：** `api/server/src/main/java/com/ke/bella/openapi/db/repo/ChannelRepo.java`

**影响方法：**

| 方法名 | 行号 | 影响类型 |
|-------|------|---------|
| `constructSql()` | 54-71 | 修改 |

**影响说明：**
- 需要新增endpoint相关的SQL条件判断

**影响评估：** 低

#### 2.4.2 ChannelCondition.java

**文件路径：** `api/server/src/main/java/com/ke/bella/openapi/db/query/ChannelCondition.java`

**影响说明：**
- 需要新增2个字段：`endpoint`、`endpointIsNull`

**影响评估：** 低

---

### 2.5 控制器层

#### 2.5.1 RouteController.java

**文件路径：** `api/server/src/main/java/com/ke/bella/openapi/endpoints/RouteController.java`

**影响接口：**

| 接口 | 行号 | 影响类型 |
|-----|------|---------|
| `POST /v1/route` | 38-59 | 无影响 |
| `POST /v1/route/list` | 61-89 | 无影响 |

**影响说明：**
- 路由层已封装变化，Controller无需修改

**影响评估：** 无

#### 2.5.2 MetadataController.java

**文件路径：** `api/server/src/main/java/com/ke/bella/openapi/endpoints/MetadataController.java`

**影响接口：**

| 接口 | 行号 | 影响类型 |
|-----|------|---------|
| `POST /v1/meta/channel/private` | 179 | 修改 |
| `PUT /v1/meta/channel/private` | 197 | 修改 |
| `GET /v1/meta/channel/list` | 108 | 修改 |
| `GET /v1/meta/channel/page` | 114 | 修改 |
| `fetchVoiceProperty()` | 223 | 待确认 |

**影响说明：**
- 请求体需要接收endpoint字段（可选）
- 响应体需要返回endpoint字段
- 需要新增endpoint字段合法性校验

**影响评估：** 中

#### 2.5.3 MetadataConsoleController.java

**文件路径：** `api/server/src/main/java/com/ke/bella/openapi/console/MetadataConsoleController.java`

**影响接口：**

| 接口 | 行号 | 影响类型 |
|-----|------|---------|
| `POST /console/channel` | 140 | 修改 |
| `PUT /console/channel` | 146 | 修改 |
| `GET /console/channels/{queueName}` | 153 | 修改 |

**影响说明：**
- 同MetadataController，需要支持endpoint字段

**影响评估：** 中

#### 2.5.4 ResponsesController.java

**文件路径：** `api/server/src/main/java/com/ke/bella/openapi/endpoints/ResponsesController.java`

**影响方法：**

| 方法 | 行号 | 影响类型 |
|-----|------|---------|
| `createResponse()` | 58-108 | 潜在风险 |
| `routeToChannel()` | 157-167 | 潜在风险 |

**影响说明：**
- **潜在问题**：endpoint从`httpRequest.getRequestURI()`获取（line 60），可能包含路径参数或查询参数
- **风险场景**：
  - POST /v1/responses → endpoint = "/v1/responses" ✅
  - GET /v1/responses/resp_123 → endpoint = "/v1/responses/resp_123" ❌
- **需要确认**：endpoint是否在拦截器中已被规范化
- **降级影响**：改造后如果精确匹配失败，会降级到纯model查询，可能掩盖问题

**影响评估：** 中-高（需要验证endpoint规范化逻辑）

#### 2.5.5 VideoController.java

**文件路径：** `api/server/src/main/java/com/ke/bella/openapi/endpoints/VideoController.java`

**影响方法：**

| 方法 | 行号 | 影响类型 |
|-----|------|---------|
| `createVideo()` | 140 | 修改 |

**影响说明：**
- 需要调整通道查询调用方式

**影响评估：** 低

---

### 2.6 异步执行器

#### 2.6.1 VideoJobExecutor.java

**文件路径：** `api/server/src/main/java/com/ke/bella/openapi/executor/VideoJobExecutor.java`

**影响方法：**

| 方法 | 行号 | 影响类型 |
|-----|------|---------|
| `execute()` | 199 | 修改 |

**影响说明：**
- 同VideoController

**影响评估：** 低

---

### 2.7 缓存层

#### 2.7.1 缓存Key变化

**当前缓存Key：**
```
channels:active:{entityType}:{entityCode}
```

**新增缓存Key：**
```
channels:active:ep:{endpoint}:model:{model}  // endpoint+model
channels:active:model:{model}                // 纯model（endpoint=NULL）
channels:active:ep:{endpoint}                // 纯endpoint（保留）
```

**影响组件：**
- JetCache配置
- ChannelService缓存注解
- `BellaOpenapiBroadcastChannel` 缓存失效广播

**影响评估：** 中（缓存key结构变化）

#### 2.7.2 缓存失效策略

**影响说明：**
- Channel更新时，需要失效相关的所有缓存key（而不是单个key）
- 需要调整缓存失效广播逻辑

**影响评估：** 中

---

### 2.8 单元测试

#### 2.8.1 ChannelRouterTest.java

**文件路径：** `api/server/src/test/java/com/ke/bella/openapi/protocol/ChannelRouterTest.java`

**现有测试用例：**

| 测试方法 | 行号 | 影响类型 |
|---------|------|---------|
| `testListAvailableChannels_WithEndpoint()` | 50-61 | 修改 |
| `testListAvailableChannels_WithModel()` | 63-79 | 修改 |
| `testListAvailableChannels_SortByPriority()` | 102-125 | 修改 |
| `testRoute_ReturnsHighestPriority()` | 127-142 | 修改 |
| `testRoute_NoChannelsAvailable()` | 144-153 | 修改 |
| `testListAvailableChannels_WithQueueMode()` | 155-167 | 修改 |

**新增测试场景：**

| 测试用例 | 测试目标 |
|---------|---------|
| `testRoute_EndpointModelMatch()` | 精确匹配endpoint+model |
| `testRoute_FallbackToGenericModel()` | 降级到通用model |
| `testRoute_FallbackToEndpoint()` | 降级到纯endpoint |
| `testRoute_Priority()` | 优先级测试 |
| `testListAvailableChannels_Combined()` | 合并结果测试 |
| `testCacheKey()` | 缓存key生成测试 |

**影响评估：** 高（6个现有用例需修改，6+个新用例）

#### 2.8.2 ChannelServiceTest.java

**影响说明：**
- 需要新增2个测试方法

**影响评估：** 中

#### 2.8.3 其他单元测试

**影响文件：**
- `VideoControllerTest.java`
- `MetadataControllerTest.java`

**影响评估：** 低-中

---

### 2.9 集成测试

**需要新增的测试场景：**

| 场景 | 测试目标 |
|------|---------|
| 精确匹配 | 验证endpoint+model精确匹配 |
| 降级查询 | 验证三级降级逻辑 |
| 优先级 | 验证查询优先级 |
| 兼容性 | 验证旧数据兼容性 |
| 性能测试 | 验证QPS和延迟指标 |

**影响评估：** 中（需要4+个场景测试）

---

### 2.10 前端Web界面

#### 2.10.1 通道管理页面

**影响页面：**

| 页面路径 | 影响内容 |
|---------|---------|
| `web/src/app/console/channels/page.tsx` | 列表新增endpoint列，筛选器新增endpoint |
| `web/src/app/console/channels/create/page.tsx` | 表单新增endpoint选择器 |
| `web/src/app/console/channels/edit/[code]/page.tsx` | 同create页面 |

**影响评估：** 中（前端工作量2-3天）

#### 2.10.2 前端API层

**文件路径：** `web/src/lib/api/channel.ts`

**影响说明：**
- 类型定义需要新增endpoint字段

**影响评估：** 低

#### 2.10.3 前端组件

**影响说明：**
- 可能需要新增endpoint选择器组件

**影响评估：** 低-中

---

### 2.11 SDK客户端

#### 2.11.1 OpenapiClient.java

**文件路径：** `api/sdk/src/main/java/com/ke/bella/openapi/client/OpenapiClient.java`

**影响方法：**

| 方法 | 行号 | 影响类型 |
|-----|------|---------|
| `route()` | 114-130 | 无影响 |
| `listAvailableChannels()` | 132-148 | 无影响 |

**影响评估：** 无（服务端已封装）

#### 2.11.2 RouteResult.java

**文件路径：** `api/sdk/src/main/java/com/ke/bella/openapi/protocol/route/RouteResult.java`

**影响说明：**
- 可能需要新增endpoint字段（可选）

**影响评估：** 低

---

### 2.12 配置文件

#### 2.12.1 application.yml

**影响说明：**
- 可能需要新增Feature Flag配置，用于灰度发布和紧急回滚

**影响评估：** 低

#### 2.12.2 Apollo配置中心

**影响说明：**
- 需要同步Feature Flag配置

**影响评估：** 低

---

### 2.13 API文档

#### 2.13.1 OpenAPI/Swagger文档

**需要更新的接口：**

| 接口 | 变更内容 |
|-----|---------|
| `POST /v1/meta/channel/private` | 请求体新增endpoint字段说明 |
| `PUT /v1/meta/channel/private` | 同上 |
| `GET /v1/meta/channel/list` | 响应体新增endpoint字段说明 |
| `POST /console/channel` | 请求体新增endpoint字段说明 |
| `PUT /console/channel` | 同上 |

**影响评估：** 低

#### 2.13.2 README文档

**需要更新的章节：**
- Channel配置说明
- 路由逻辑说明
- 示例配置

**影响评估：** 低

---

### 2.14 监控与告警

#### 2.14.1 监控指标

**需要新增的监控指标：**

| 指标名称 | 含义 |
|---------|------|
| `channel.route.endpoint_model_match` | endpoint+model精确匹配次数 |
| `channel.route.generic_model_fallback` | 降级到通用model次数 |
| `channel.route.endpoint_fallback` | 降级到纯endpoint次数 |
| `channel.route.no_match` | 无匹配通道次数 |
| `channel.route.latency` | 路由响应时间 |
| `channel.cache.hit_rate` | 缓存命中率 |

**影响评估：** 中（需要在代码中埋点）

#### 2.14.2 Grafana Dashboard

**影响说明：**
- 需要新增通道路由监控面板

**影响评估：** 中

#### 2.14.3 日志增强

**影响说明：**
- 需要新增路由成功、降级、失败等关键日志

**影响评估：** 低

---

### 2.15 运维脚本

**需要编写的脚本：**

| 脚本名称 | 功能 |
|---------|------|
| `migrate-channel-endpoint.sql` | 迁移现有数据 |
| `verify-channel-data.sql` | 校验数据迁移结果 |
| `rollback-channel-endpoint.sql` | 回滚数据和表结构 |

**影响评估：** 中

---

### 2.16 其他影响点

#### 2.16.1 限流逻辑

**当前维度：** apikey + model

**是否需要调整：** 待确认（是否需要apikey + endpoint + model维度）

**影响评估：** 待确认

#### 2.16.2 成本计算

**影响评估：** 无（priceInfo在channel维度，已支持）

#### 2.16.3 安全合规

**影响评估：** 无（过滤逻辑与endpoint无关）

#### 2.16.4 队列模式

**影响评估：** 无（但可以利用新能力为不同endpoint配置不同queueMode）

---

## 3. 影响范围汇总

### 3.1 按影响程度分类

| 影响程度 | 涉及模块 | 文件数 | 工作量估算 |
|---------|---------|--------|-----------|
| **高** | 数据库DDL、ChannelRouter、ChannelRouterTest | 5 | 3-4天 |
| **中** | ChannelService、前端页面、Controller、监控埋点、集成测试 | 15 | 4-5天 |
| **低** | ChannelRepo、Condition、VideoController、配置、文档 | 15 | 1-2天 |

**总工作量：8-11天（约2周）**

### 3.2 按模块分类

| 模块 | 涉及文件数 | 影响程度 | 备注 |
|------|-----------|---------|------|
| 数据库层 | 3 | 高 | DDL、迁移脚本、jOOQ生成 |
| 核心路由层 | 1 | 高 | ChannelRouter重构 |
| 服务层 | 2 | 中 | ChannelService新增方法 |
| 数据访问层 | 2 | 低 | ChannelRepo、ChannelCondition |
| 控制器层 | 4 | 中 | 接口支持endpoint字段 |
| 异步执行器 | 1 | 低 | VideoJobExecutor |
| 缓存层 | 2 | 中 | 缓存key变化、失效策略 |
| 单元测试 | 5+ | 高 | 修改现有+新增测试 |
| 集成测试 | - | 中 | 新增场景测试 |
| 前端 | 5+ | 中 | 管理页面、API、组件 |
| SDK | 2 | 低 | 无需修改或小改 |
| 配置/文档 | 10+ | 低 | 配置、API文档、README |
| 监控告警 | 3 | 中 | 指标、Dashboard、日志 |
| 运维脚本 | 3 | 中 | 迁移、校验、回滚 |

**总计：40+个文件**

### 3.3 风险点汇总

| 风险点 | 风险等级 | 影响 |
|-------|---------|------|
| 数据库DDL执行锁表 | 高 | 可能影响线上服务 |
| 三级查询性能劣化 | 中 | QPS下降、延迟增加 |
| 缓存key变化导致缓存miss | 中 | 短时间内数据库压力增加 |
| 旧数据迁移错误 | 中 | 路由失败、服务不可用 |
| 路由逻辑bug | 高 | 通道匹配错误、成本异常 |
| 前端兼容性问题 | 低 | 旧版本前端无法显示endpoint |

---

## 4. 待确认问题

需要与团队讨论的技术决策：

1. **Endpoint规范化问题**（重要）
   - 当前Controller中从`httpRequest.getRequestURI()`获取endpoint
   - 可能包含路径参数（如/v1/responses/resp_123）或查询参数
   - **需要确认**：是否需要统一规范化endpoint？在哪一层处理？
   - **建议**：在拦截器层统一提取标准endpoint路径

2. **查询优先级**
   - 三级降级顺序是否合理（endpoint+model → 通用model → 纯endpoint）？
   - 是否需要通过配置调整？

3. **缓存策略**
   - 新旧缓存key如何平滑切换？
   - 是否需要双写缓存？

4. **Feature Flag**
   - 是否需要灰度开关？
   - 回滚策略如何设计？

5. **限流维度**
   - 是否需要调整为endpoint维度？

---

**文档版本：** v1.1
**编写日期：** 2026-03-09
**最后更新：** 2026-03-09