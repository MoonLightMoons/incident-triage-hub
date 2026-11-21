# T12｜操作手册：从服务启动到工单归档

**状态：待应用交付后执行。** 本手册按实际操作顺序说明如何从空库初始化环境、走通一张工单的完整生命周期、应对异常分支以及执行管理员日常操作。角色边界与业务规则详见[需求规格](REQUIREMENTS.md)，完整请求参数与错误码详见 [API 契约](API.md)；开发排期参阅[实施计划](IMPLEMENTATION.md)，正式测试回归请执行[验收清单](ACCEPTANCE.md)。

## 1. 初始化并检查环境

1. **检查基础依赖：** 准备 Java 21、Maven 与 MySQL 8.0，在终端分别执行 `java -version`、`mvn -v`、`mysql --version` 确认版本就绪。创建独立的开发数据库（如 `incident_triage_dev`）；使用具备建表权限的账号执行初始化脚本，应用运行时连接建议使用仅具备读写权限的数据库账号。
2. **导入建表与种子脚本：** 在仓库根目录下启动 MySQL 客户端并连接开发库，依次执行 `SOURCE sql/schema.sql;` 与 `SOURCE sql/seed.sql;`。注意：`seed.sql` 中的演示数据仅限开发与联调环境使用；生产环境部署时仅导入 `schema.sql`，并通过专用初始化流程创建首个管理员账号及 `P0`–`P3` 分级字典。
3. **配置环境变量并启动服务：** 对照[运行配置](PLAN.md#6-配置运行和交付约束)，在当前终端会话中注入 `DB_URL`、`DB_USERNAME`、`DB_PASSWORD` 与 `JWT_SECRET`。首次启动建议不配置外部模型凭据（`AI_API_KEY`），优先验证本地规则引擎可独立支撑全流程。激活 `dev` 环境后构建并启动应用：

   ```powershell
   $env:SPRING_PROFILES_ACTIVE = 'dev'
   mvn clean verify
   java -jar target/incident-triage-hub.jar
   ```

4. **验证健康状态与登录令牌：** 打开新终端，执行 `Invoke-RestMethod http://localhost:8080/api/health`（若修改了 `SERVER_PORT` 则同步调整端口）。确认返回 `UP` 后，即可通过浏览器访问 Knife4j 文档页面或使用 API 客户端。调用 `POST /api/auth/login` 登录获取 `accessToken`，在后续受保护请求中携带请求头 `Authorization: Bearer <token>`，并调用 `GET /api/auth/profile` 核对当前登录身份与角色。
5. **确认基础主数据就绪：** 使用管理员账号确认系统中至少存在一名启用状态的上报人、一名评估师及一项启用状态的业务资产；通过 `GET /api/assets` 记录可用资产的 `assetId`，通过 `GET /api/severity-rules` 核对 `P0`–`P3` 四档分数区间。若需执行水平越权与并发抢单验收，还需准备第二名上报人（`R2`）与第二名评估师（`A2`）。

## 2. 处理一张工单

以典型故障**“校园统一认证连续超时”**为例：故障描述应清晰包含故障开始时间、受影响业务功能、影响范围（局部或全量用户）以及复现步骤；仅填写排障必需的业务现象，切勿填入真实密码、私钥或无关个人隐私。线下沟通交接使用可读工单号（`incidentNo`），接口调用路径使用数字主键（`id`）。每次修改描述或推进状态前，均须传入最近一次查询响应中的最新 `version`。

| 阶段操作 | 执行角色 | 具体调用步骤 | 成功后状态与流转去向 |
| --- | --- | --- | --- |
| 1. 可选预评估 | 上报人 (`R`) | 选定已启用资产，调用 `POST /api/ai/preview-assess`，传入拟提交的标题、故障描述与 `assetId`，查看建议等级、影响分及初步排查建议。 | 预览结果不落库、不生成工单 ID；由上报人确认信息完整后决定是否正式提单。 |
| 2. 正式上报 | 上报人 (`R`) | 调用 `POST /api/incidents`，除标题、描述与 `assetId` 外，附上带时区的故障发生时间 `occurredAt`。记录返回的 `id`、`incidentNo` 与初始 `version`（`0`）。若在认领前发现描述遗漏，可调用 `PUT /api/incidents/{id}` 补充。 | 状态为 `PENDING_TRIAGE`，进入评估师可见的待分诊池。 |
| 3. 认领并定级 | 评估师 (`A`) | 调用 `GET /api/incidents?status=PENDING_TRIAGE` 查找待分诊工单，并通过 `GET /api/incidents/{id}` 查看资产重要度、故障描述与首次评估。必要时调用 `POST /api/incidents/{id}/ai-assess` 触发深度复评。随后调用 `POST /api/incidents/{id}/triage` 提交 `version`、`finalSeverity` 及可选的 `assessmentId`（若人工定级与参考建议不同，须填写 `triageReason`）。 | 状态进入 `TRIAGED`，当前处理人锁定为该评估师，后续处置由其跟进。 |
| 4. 开始处置 | 当前处理人 (`A`) | 调用 `POST /api/incidents/{id}/start` 将工单推进至处置中；排查与止损过程中，可随时调用 `POST /api/incidents/{id}/actions` 记录排查（`DIAGNOSE`）、止损（`MITIGATE`）、修复（`REPAIR`）或通报（`NOTIFY`）进展。 | 状态为 `IN_PROGRESS`，处理人持续推进故障修复。 |
| 5. 提请验证 | 当前处理人 (`A`) | 故障修复后，调用 `POST /api/incidents/{id}/submit-verify`，提交修复结论 `repairSummary`、验证方法 `verifyMethod`、预期结果 `expectedResult` 及可选耗时。 | 状态进入 `PENDING_VERIFY`，交由原上报人查阅时间轴并按验证方法复测。 |
| 6. 复测确认 | 原上报人 (`R`) | 按处理人提供的验证方法复测后，调用 `POST /api/incidents/{id}/verify`：验证通过传 `passed: true`；若故障依旧，传 `passed: false` 并在 `reason` 中说明复测现象。 | 验证通过则进入 `RESOLVED`（生成 `resolved_at`）并移交管理员归档；未通过则退回 `IN_PROGRESS`，由原处理人继续修复并重新提请验证。 |
| 7. 复盘归档 | 管理员 (`M`) | 调用工单详情与 `GET /api/incidents/{id}/timeline`，全面核查定级依据、处置动作与复测闭环；确认无误后调用 `POST /api/incidents/{id}/close` 归档，可在 `remark` 中记录复盘总结或后续改进项。 | 状态进入正常终态 `CLOSED`，纳入历史档案与统计报表。 |

在未发生“补充描述”或“验证打回”的标准主线流程中，工单建单初始版本为 `0`，归档后版本为 `5`，时间轴中共包含 6 条状态变更日志。触发深度 AI 复评与日常追加处置记录均不会改变工单状态与 `version`。

## 3. 处理中断、撤回或驳回

| 异常或分支情况 | 标准应对与恢复方式 |
| --- | --- |
| 上报信息有误或无需继续处理 | 若工单仍处于未被认领的 `PENDING_TRIAGE` 状态，原上报人可通过 `PUT /api/incidents/{id}` 补充修正描述；若确认属误报或故障已自行消除，可调用 `POST /api/incidents/{id}/cancel` 填写原因并撤回。 |
| 评估师判定工单无效或重复 | 评估师对未认领的 `PENDING_TRIAGE` 工单调用 `POST /api/incidents/{id}/reject`，写明驳回依据；原上报人与管理员可在时间轴中查看驳回原因。 |
| 原上报人复测发现故障未恢复 | 在 `PENDING_VERIFY` 工单上调用 `/verify` 提交 `passed: false` 及具体未通过原因；工单退回 `IN_PROGRESS`，由原处理人继续排查修复后再次提请验证。 |
| 提单时收到重复上报提示 (`40902`) | 表明同一上报人在当前 UTC 自然分钟内已针对同一资产提交过相同标题的工单。先调用 `GET /api/incidents` 查询本人事件列表，定位已创建的工单继续操作。 |
| 认领、撤回或流转时遇到冲突 (`40901` / `40903`) | 重新调用详情接口获取最新状态与 `version`：若工单已被其他评估师抢先认领或状态已变更，按最新状态及责任归属交接，切勿使用旧版本号强行重试。 |
| 遇到认证或权限拒绝 (`40101` / `40301` / `40302`) | `40101` 需重新登录获取新令牌；`40301` 表明当前角色无权调用该接口，需切换至对应角色账号；`40302` 表明当前账号对该工单无归属权，需核对工单 ID 或切换至原上报人/当前处理人账号。 |
| 服务端内部异常 (`50000`) 或客户端请求超时 | 记录响应中的 `requestId`（超时无响应时先通过列表或详情查询操作是否已实际生效）。确认未生效且服务端事务已回滚后，再发起重试或联系运维结合服务端日志排查。 |
| 终态工单关联故障再次复发 | 已处于 `CLOSED`、`REJECTED`、`CANCELLED` 终态的工单不支持重新开启；请重新发起上报创建新工单，并在故障描述中引用原工单号（`incidentNo`）以便关联追溯。 |

## 4. 管理员的日常操作

- **用户与资产维护：** 定期核查账号角色与启用状态；演示与测试使用专用账号。新增或修改业务资产后，可用上报人账号验证启用资产目录的可见性。停用某项资产前先排查关联的在途工单；资产停用后仅拦截新提单，已创建的存量工单仍可正常处置至归档。
- **分级字典维护：** 修改前先调用 `GET /api/severity-rules` 查看当前四档配置；单档的名称、解决 SLA 或 AI 提示词可单独修改。若需调整分数边界，必须通过 `scoreBands` 将 `P0`–`P3` 四档一次性提交；更新完成后重新查询确认，历史已落库的评估记录与已定级快照无需刷新重算。
- **数据复盘与统计报表：** 使用相同的创建时间窗口（`createdFrom` 与 `createdTo`）依次查询 `severity-summary`（各等级与状态的工单分布及均分）、`high-risk-assets`（故障频次 Top 5 资产及平均恢复耗时）和 `ai-adoption-rate`（AI 定级一致率与评估平均耗时）。若返回空数组或 `null`，优先核对时间窗口及窗口内是否存在已完成认领定级（或已解决）的工单；详细计算口径参见[统计需求](REQUIREMENTS.md#6-复盘与报表)。
- **生产发布前核查：** 在目标部署环境显式指定 `SPRING_PROFILES_ACTIVE=prod` 并连接独立生产数据库，逐项核查健康检查接口、管理员初始登录、环境变量注入及日志脱敏（确认 Knife4j 调试页已关闭）。上线前先完成离线规则主线回归；若需启用外部大模型，须使用获准接入的模型配置与脱敏构造样本进行专项验证。
