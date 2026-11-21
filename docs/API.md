# T12｜API 契约

**状态：待实现，共 32 个 HTTP 接口。** 本文档完整定义各接口的 HTTP 方法、路由路径、角色权限、请求参数、响应结构与错误码；业务规则与状态定义详见[需求规格](REQUIREMENTS.md)，架构分层与事务边界详见[系统设计方案](PLAN.md)。文中的 JSON 示例均为目标交付格式，用于指导编码与联调。

## 1. 公共约定

| 维度 | 规范约定 |
| --- | --- |
| 通信协议与认证 | 统一采用 UTF-8 编码的 JSON。凡包含 JSON 请求体的接口均须携带 `Content-Type: application/json`。除明确标注“免登录”的接口外，所有请求均须携带请求头 `Authorization: Bearer <token>`。 |
| 角色简写与权限校验 | `R` 代表上报人（`REPORTER`），`A` 代表评估师（`ANALYST`），`M` 代表管理员（`ADMIN`）；“登录”代表上述任意一类处于启用状态的账号。每次请求均实时校验当前角色，工单相关接口还须按[角色和归属](REQUIREMENTS.md#2-术语和角色)校验数据可见性。 |
| 统一响应结构 | 响应体固定包含 `code`（业务状态码）、`message`（可读提示）、`data`（业务载荷）和 `requestId`（服务端请求追踪标识）。请求成功时 HTTP 状态码与业务 `code` 均为 `200`；无业务返回对象时 `data` 为 `null`。 |
| 资源 ID 与乐观锁版本 | 路径参数 `{id}` 均为正整数数据库主键；对外工单号（`incidentNo`）仅用于检索展示。凡修改工单描述或推进工单状态的请求，均须传入最近一次读取到的 `version`（非负整数）；建单初始版本为 `0`，每次成功修改描述或推进状态后版本加 `1`。列表、详情与写入响应均返回最新 `version`。 |
| 时间格式与时间窗 | 请求时间参数使用带时区偏移的 ISO 8601 格式（如 `2026-11-20T09:00:00+08:00`）；服务端响应统一输出为 UTC 时间（如 `2026-11-20T01:00:00Z`）。按创建时间过滤时统一使用 `createdFrom`（含）与 `createdTo`（不含），两者须同时提供或同时省略，且 `createdFrom < createdTo`。 |
| 分页参数与响应 | 页码 `page` 从 `1` 开始（默认 `1`）；每页条数 `size` 默认 `20`，取值范围 `1`–`100`。分页响应 `data` 结构统一为 `{items, page, size, total}`；无匹配记录时 `items` 为空数组 `[]`，`total` 仍反映过滤后的真实总条数。 |
| 通用入参校验 | 除密码字段外，所有文本字段均在去除首尾空白后执行长度与非空校验（必填文本不得仅由空白字符组成）。严格拒绝未在契约中声明的多余 JSON 字段；任何参数校验不通过均返回业务码 `40001`。当前操作者、角色、目标状态、评估结果及审计时间戳均由服务端权威生成。 |

```json
{
  "code": 200,
  "message": "成功",
  "data": {
    "id": 42,
    "incidentNo": "INC20261120-8K4NR",
    "status": "PENDING_TRIAGE",
    "version": 0
  },
  "requestId": "req-example-42"
}
```

### 常用字段约束

| 字段名 | 类型与推荐初始约束 |
| --- | --- |
| `username` | 3–32 个字符；仅限小写英文字母、数字与下划线；服务端去除首尾空白后统一按小写存储与匹配。 |
| `password` / `oldPassword` / `newPassword` | 8–64 个字符，且 UTF-8 编码后不超过 72 字节（适配 BCrypt 算法上限）；保留原始字符，不做首尾空白截断（trim）。 |
| `realName` / `department` / `phone` | `realName` 必填，1–64 个字符；`department` 与 `phone` 可选，长度上限分别为 100 与 32 个字符。 |
| `assetCode` / `assetName` / `category` / `ownerName` | 新增资产时，`assetCode`（1–32 字符）、`assetName`（1–100 字符）、`category`（1–64 字符）均为必填；负责人 `ownerName` 可选，最长 64 个字符。 |
| `criticality` / `status` / SLA | 资产重要度 `criticality` 为 1–5 的整数；用户与资产的 `status` 取值为 `1`（启用）或 `0`（停用）；`slaResponseMin` 与 `slaResolveHours` 均为正整数。 |
| `severityName` / `aiPromptHint` / `scoreMin` / `scoreMax` | 等级名称 `severityName` 为 1–32 个字符；AI 定级提示 `aiPromptHint` 最长 500 个字符；分数上下界为 0–100 的整数。 |
| `title` / `description` | 工单标题 `title` 为 1–120 个字符；故障描述 `description` 为 1–4000 个字符。 |
| `remark` / `reason` / `triageReason` | 最长 500 个字符；在要求必填的场景下，去除首尾空白后至少包含 1 个有效字符。 |
| `repairSummary` / `verifyMethod` / `expectedResult` / `actionDetail` | 均为必填业务文本，长度上限分别为 2000、1000、1000、2000 个字符；处置耗时 `costMinutes` 可选，为非负整数。 |

## 2. 健康和账号（5 个）

| # | 方法与路径 | 访问权限 | 入参说明与响应语义 |
| --- | --- | --- | --- |
| 01 | `GET /api/health` | 免登录 | 数据库正常连通时返回 HTTP `200`，`data.status` 为 `"UP"`；数据库不可用时返回 HTTP `503`，`data.status` 为 `"DOWN"`。严禁返回数据库地址、凭据或异常堆栈。 |
| 02 | `POST /api/auth/register` | 免登录 | 请求体包含 `username`、`password`、`realName`，可选 `department`、`phone`；成功返回新用户资料摘要，角色固定为 `REPORTER`；用户名已存在时返回 `40904`。 |
| 03 | `POST /api/auth/login` | 免登录 | 请求体包含 `username`、`password`；成功返回 `{accessToken, tokenType: "Bearer", expiresInSeconds}`。账号不存在、已停用或密码错误统一返回 `40101`。 |
| 04 | `GET /api/auth/profile` | 登录 | 返回当前登录用户的个人资料：`{id, username, realName, role, department, phone, status}`。 |
| 05 | `PUT /api/auth/password` | 登录 | 请求体包含 `oldPassword`、`newPassword`。旧密码校验失败返回 `40001`；修改成功返回 `data: null`，并使该账号此前签发的所有令牌立即失效。 |

## 3. 用户与主数据（9 个）

| # | 方法与路径 | 访问权限 | 入参说明与响应语义 |
| --- | --- | --- | --- |
| 06 | `GET /api/users` | M | 支持按 `username`（模糊）、`role`、`status` 分页筛选；默认按 `id` 降序排列，返回管理员专用的用户列表视图。 |
| 07 | `PUT /api/users/{id}/role` | M | 请求体包含目标 `role`；成功返回更新后的账号摘要。若该操作会导致系统中无启用状态的管理员，返回 `40905`。 |
| 08 | `PUT /api/users/{id}/status` | M | 请求体包含目标 `status`（`1` 或 `0`）；成功返回更新后的账号摘要。若该操作会停用最后一名启用状态的管理员，返回 `40905`。 |
| 09 | `GET /api/assets` | 登录 | 支持按 `keyword`（匹配编码或名称）、`category` 分页筛选，默认按 `id` 降序。仅管理员可传 `status` 筛选停用资产；其他角色固定只返回启用资产。 |
| 10 | `GET /api/assets/{id}` | 登录 | 返回资产详情：`{id, assetCode, assetName, category, criticality, ownerName, slaResponseMin, status}`。非管理员查询已停用资产时返回 `40401`。 |
| 11 | `POST /api/assets` | M | 请求体包含 `assetCode`、`assetName`、`category`、`criticality`、`slaResponseMin`，可选 `ownerName`、`status`（默认 `1`）；成功返回新建资产；编码冲突返回 `40904`。 |
| 12 | `PUT /api/assets/{id}` | M | 可更新 `assetName`、`category`、`criticality`、`slaResponseMin`、`ownerName`、`status`（至少提供一项），`assetCode` 不可修改；成功返回更新后的资产详情。 |
| 13 | `GET /api/severity-rules` | 登录 | 按 `P0` 至 `P3` 顺序返回全部四档分级规则，每项包含 `{id, severityCode, severityName, scoreMin, scoreMax, slaResolveHours, aiPromptHint}`。 |
| 14 | `PUT /api/severity-rules/{id}` | M | 可针对 `{id}` 所在档位提交 `severityName`、`slaResolveHours`、`aiPromptHint` 中的至少一项，或提交 `scoreBands` 批量调整四档分数边界（也可同时提交）。成功返回更新后的全部四档规则。 |

第 14 号接口中，路径参数 `{id}` 用于定位需要修改名称、解决 SLA 或 AI 提示词的目标档位。当涉及分数边界调整时，必须通过 `scoreBands` 数组一次性传入全部四档（每个元素包含 `severityCode`、`scoreMin`、`scoreMax`，`P0`–`P3` 各出现一次），不接受仅针对单个档位单独修改 `scoreMin` 或 `scoreMax`。服务端按[分级规则约束](REQUIREMENTS.md#3-账号和主数据)对四档区间执行整体校验，任一档位不合法则整次更新回滚并返回 `40001`。

## 4. 事件列表、详情与状态流转（11 个）

| # | 方法与路径 | 权限与前置状态 | 入参说明与响应语义 |
| --- | --- | --- | --- |
| 15 | `POST /api/incidents` | R | 请求体包含 `title`、`description`、`assetId`、`occurredAt`；成功返回工单摘要与首次评估对象 `assessment`。资产已停用或 `occurredAt` 晚于服务端当前时间返回 `40001`；命中防重规则返回 `40902`。 |
| 16 | `GET /api/incidents` | 登录（按角色与归属过滤） | 支持按 `incidentNo`（精确）、`title`（模糊）、`status`、`finalSeverity`、`assetId` 及创建时间窗分页筛选；默认按 `createdAt DESC, id DESC` 排序。 |
| 17 | `GET /api/incidents/{id}` | 登录（校验归属权） | 返回工单完整详情及分页评估历史 `assessments`（分页参数 `assessmentPage`、`assessmentSize` 遵循通用分页规则，按评估记录 `createdAt DESC, id DESC` 排序）。 |
| 18 | `PUT /api/incidents/{id}` | R 本人，`PENDING_TRIAGE` | 请求体包含 `version`、`description`；成功返回更新后的工单摘要（`version + 1`）。 |
| 19 | `POST /api/incidents/{id}/triage` | A，未认领的 `PENDING_TRIAGE` | 请求体包含 `version`、`finalSeverity`，可选 `assessmentId`、`triageReason`。引用非本单评估记录，或人工定级与参考建议不一致但未填理由时返回 `40001`。成功进入 `TRIAGED`，返回工单摘要与锁定的定级快照。 |
| 20 | `POST /api/incidents/{id}/start` | A 本人负责，`TRIAGED` | 请求体包含 `version`，可选 `remark`；成功流转至 `IN_PROGRESS` 并返回工单摘要。 |
| 21 | `POST /api/incidents/{id}/submit-verify` | A 本人负责，`IN_PROGRESS` | 请求体包含 `version`、`repairSummary`、`verifyMethod`、`expectedResult`，可选 `costMinutes`；成功流转至 `PENDING_VERIFY` 并返回工单摘要。 |
| 22 | `POST /api/incidents/{id}/verify` | R 原上报人，`PENDING_VERIFY` | 请求体包含 `version`、布尔值 `passed` 以及 `reason`（当 `passed=false` 时必填）。验证通过流转至 `RESOLVED`，未通过则退回 `IN_PROGRESS`。 |
| 23 | `POST /api/incidents/{id}/reject` | A，未认领的 `PENDING_TRIAGE` | 请求体包含 `version`、`reason`；成功流转至异常终态 `REJECTED`。 |
| 24 | `POST /api/incidents/{id}/cancel` | R 本人，未认领的 `PENDING_TRIAGE` | 请求体包含 `version`、`reason`；成功流转至异常终态 `CANCELLED`。 |
| 25 | `POST /api/incidents/{id}/close` | M，`RESOLVED` | 请求体包含 `version`，可选 `remark`；成功流转至正常终态 `CLOSED`。 |

- **工单摘要结构：** 包含 `id`、`incidentNo`、`title`、`status`、`assetId`、`reporterId`、可空的 `assigneeId`、`aiSeverity`、`impactScore`、可空的 `finalSeverity`、`occurredAt`、`createdAt`、可空的 `resolvedAt` 及最新 `version`。所有状态流转接口成功时默认返回该摘要对象。
- **工单详情结构：** 在工单摘要基础上，扩展 `description`、关联资产摘要（编码、名称、类别、重要度、响应 SLA）、上报人与当前处理人的 `{id, realName}`、分页评估历史 `assessments`，以及可空的 `referenceAssessmentId` 与 `triageReason`。严禁在详情中暴露协作者的手机号或敏感账号字段。
- **并发与状态冲突判定：** 写入前须先通过角色与工单可见性校验（不满足时分别返回 `40301` 或 `40302`）。当乐观锁条件更新未命中时，服务端重查工单最新状态：若当前状态已不支持所请求的操作，返回 `40901`；若当前状态仍允许该操作但客户端传入的 `version` 已过期（包括两名评估师基于同一待分诊版本并发抢单而失利的一方），返回 `40903`。

## 5. AI、处置和时间轴（4 个）

| # | 方法与路径 | 权限与前置状态 | 入参说明与响应语义 |
| --- | --- | --- | --- |
| 26 | `POST /api/ai/preview-assess` | R | 请求体包含 `assetId`、`title`、`description`；返回实时评估结果（不落库，故无 `id`、`createdAt` 与 `isAdopted` 字段）。 |
| 27 | `POST /api/incidents/{id}/ai-assess` | A（符合归属）或 M；仅限[可复评状态](REQUIREMENTS.md#5-ai-评估处置记录和时间轴) | 无需请求体；返回新落库的评估记录。若在调用外部模型期间工单版本、状态或归属发生变更，落库前重查将拦截并丢弃该过期结果。 |
| 28 | `POST /api/incidents/{id}/actions` | A 本人负责，`IN_PROGRESS` | 请求体包含 `actionType`（取值 `DIAGNOSE`、`MITIGATE`、`REPAIR`、`NOTIFY`）、`actionDetail`，可选 `costMinutes`；返回新增的处置记录。 |
| 29 | `GET /api/incidents/{id}/timeline` | 登录（校验归属权） | 支持分页；合并返回状态变更（`STATUS`）与应急处置（`ACTION`）两类记录。按 `createdAt ASC` 排序；时间相同时 `ACTION` 排在 `STATUS` 之前，同类型再按 `id ASC` 排序。 |

- **评估对象结构：** 包含 `engineType`（`LLM_API` 或 `RULE_FALLBACK`）、可空的 `modelName`、`suggestedSeverity`、`impactScore`、`affectedScope`、`rootCauseGuess`、`mitigationSteps`（文本说明）及 `latencyMs`；已落库的评估记录额外包含 `id`、`createdAt` 与 `isAdopted`。其中 `affectedScope`、`rootCauseGuess`、`mitigationSteps` 的长度上限分别为 1000、1000、2000 个字符；外部模型响应若缺失必填字段、数值越界或等级与分数区间不匹配，服务端自动降级为本地规则评估。
- **时间轴条目结构：** 所有条目共有 `id`、`entryType`（`STATUS` 或 `ACTION`）、`operatorId`、`operatorName` 与 `createdAt`。其中 `STATUS` 条目额外包含 `fromStatus`（建单首条记录为 `null`）、`toStatus`、`remark`；`ACTION` 条目额外包含 `actionType`、`actionDetail`、可空的 `costMinutes`，由“提请验证”生成的修复记录还包含 `verifyMethod` 与 `expectedResult`。由于两张底层表的自增 `id` 可能重复，客户端渲染列表时应使用 `entryType + id` 作为联合唯一键。

## 6. 统计（3 个）

以下三个报表接口均仅限管理员（`M`）访问，支持可选的创建时间窗口参数（`createdFrom` 与 `createdTo`）。各指标的统计口径详见[统计需求](REQUIREMENTS.md#6-复盘与报表)。

| # | 方法与路径 | 响应数据结构与精度规范 |
| --- | --- | --- |
| 30 | `GET /api/stats/severity-summary` | 返回分组数组，元素结构为 `{finalSeverity, status, count, avgImpactScore}`。按 `P0` 至 `P3` 排序，同一等级内按状态编码字典序升序排列；平均影响分保留 2 位小数。 |
| 31 | `GET /api/stats/high-risk-assets` | 返回最多前 5 项数组，元素结构为 `{assetId, assetName, incidentCount, resolvedSampleCount, avgRecoverySeconds}`。平均恢复耗时单位为秒，保留 2 位小数；无已解决样本时 `avgRecoverySeconds` 为 `null`。 |
| 32 | `GET /api/stats/ai-adoption-rate` | 返回统计对象 `{triagedCount, matchedCount, consistencyRate, assessmentCount, avgLatencyMs}`。一致率为 0–1 之间的小数，保留 4 位小数；平均评估耗时单位为毫秒，保留 2 位小数。 |

所有均值与比率均采用四舍五入（`HALF_UP`）保留指定小数位；无样本时的空数组 `[]`、零计数 `0` 与 `null` 语义严格遵循需求规格。评估记录统计与已定级工单统计分别独立聚合，确保单张工单的多次复评不会放大定级一致率的分母。

## 7. 错误响应与客户端恢复

| HTTP 状态码 | 业务码 | 典型触发场景 | 客户端恢复与处理指引 |
| --- | --- | --- | --- |
| `400` | `40001` | 必填参数缺失、格式/长度越界、验证打回未填原因、分级规则区间断档或重叠 | 根据错误提示修正对应字段后重新提交。 |
| `401` | `40101` | 账号密码错误、JWT 缺失/篡改/过期、账号被停用或改密后旧会话失效 | 引导用户使用有效账号重新登录以获取新令牌。 |
| `403` | `40301` | 当前账号角色无权调用该接口（如上报人调用认领接口、管理员代上报人验证） | 切换至具备相应角色权限的账号执行操作。 |
| `403` | `40302` | 角色允许调用该类接口，但目标工单不在当前账号的数据归属范围内 | 仅访问或操作本人上报、本人负责或待分诊池内的工单。 |
| `404` | `40401` | 目标资源 ID 不存在，或非管理员直接查询已停用的资产 | 刷新列表并核对资源 ID 是否正确。 |
| `409` | `40901` | 工单当前状态不支持所请求的流转或操作（如对已认领工单撤回、对已归档工单追加记录） | 刷新工单详情，依据最新状态执行下一步合法操作。 |
| `409` | `40902` | 命中同一上报人、同一资产、同一 UTC 自然分钟内相同标题的防重复提交约束 | 查询本人已上报事件列表，定位已成功创建的工单继续跟进。 |
| `409` | `40903` | 并发认领抢单失利，或客户端提交的乐观锁 `version` 已落后于服务端最新版本 | 重新拉取工单详情以获取最新 `version` 与状态，确认仍可操作后再提交。 |
| `409` | `40904` | 注册用户名或新增资产的 `assetCode` 已存在 | 更换为未被占用的用户名或资产编码。 |
| `409` | `40905` | 试图停用或降级系统中最后一名处于启用状态的管理员 | 先启用或任命另一名管理员，再执行该变更。 |
| `500` | `50000` | 数据库写入异常或未预期的服务端内部错误（已触发事务回滚） | 记录响应中的 `requestId`，先查询工单确认实际状态，再联系运维排查服务端日志。 |
| `503` | `50301` | 健康检查探针检测到数据库连接不可用 | 检查 MySQL 服务状态及数据源环境变量配置。 |

当参数校验失败（`40001`）时，可在 `data` 中附带结构化的 `fieldErrors` 数组（每个元素包含 `field` 与 `message`）。任何错误响应均严禁回显明文密码、令牌、真实环境密钥或 Java 异常堆栈。所有涉及业务数据的请求均须严格按照“1. 身份认证 → 2. 角色权限（`40301`） → 3. 资源存在性与数据归属（`40401` / `40302`） → 4. 业务状态与乐观锁版本（`40901` / `40903`）”的顺序执行校验。
