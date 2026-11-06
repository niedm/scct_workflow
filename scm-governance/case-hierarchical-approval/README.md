# Case 分级审批流（3.4.2 MoA）

SpiffWorkflow 实现，对应《供应链控制台平台 · 业务需求分析文档 11.04》3.4.2「业务分级审批流机制（四级风险授权矩阵 MoA）」。

## 文件

| 文件 | 说明 |
|---|---|
| `case-hierarchical-approval.bpmn` | 主流程：风险判定 → L1~L4 路由 → 逐级签批 → 通过/驳回/超时催办 |
| `risk_evaluation.dmn` | 风险决策表，决策 ID `risk_level_decision` |
| `approval-decision-schema.json` / `-uischema.json` | 通用审批表单（L1~L4） |
| `l4-special-approval-schema.json` / `-uischema.json` | L4 特批表单（视频会商、联合签署） |
| `process_model.json` | 流程模型元数据 |

流程模型标识：`scm-governance:case-hierarchical-approval`

## 启动变量

通过 `POST /v1.0/process-instances/scm-governance:case-hierarchical-approval` 创建实例时必须提供：

| 变量 | 类型 | 说明 |
|---|---|---|
| `case_id` | string | Case 标识 |
| `solution_id` | string | 方案标识 |
| `solution_version` | int | 方案版本 |
| `added_cost` | number | 追加成本（元） |
| `customer_tier` | string | `T1` / `T2` / `T3` |
| `delay_days` | number | 延期天数 |
| `ot_mode` | string | `NONE` / `WEEKEND_SINGLE_8H` / `WEEKEND_DOUBLE` |
| `transfer_scope` | string | `NONE` / `FRIEND_PLANT` / `CROSS_BASE` |
| `legal_claim` | boolean | 重大法律索赔/丢单 |
| `plant_shutdown` | boolean | 工厂停产 |
| `labor_law_breach` | boolean | 突破劳动法规加班红线 |
| `risk_snapshot_hash` | string | 风险快照 hash（展示与审计） |
| `rule_version` | string | 风险规则版本 |

可选变量：

| 变量 | 默认 | 说明 |
|---|---|---|
| `l2_approver_rule` | `JOINT` | L2 路由：`R3` / `R5` / `JOINT`（R3+R5 串行会签） |
| `case_sync_url` | `http://case-service:8080/internal/cases/{case_id}/approval-events` | Case 状态同步地址 |
| `notification_gateway_url` | `http://notification-gateway:8080/api/notify` | 催办/通知网关地址 |

## 审批结论

表单字段 `decision`：

- `APPROVE`：通过
- `APPROVE_WITH_CONDITIONS`：附条件通过（`conditions` 记录条件）
- `REJECT`：驳回，流程结束并回传 `approval_result=REJECTED`

L3 终审、L4 各节点要求填写 `comment`；L2~L4 要求 `signature_evidence`（生物识别凭证）。L4 特批使用 `l4-special-approval-schema.json`，要求视频会商链接与联合签署凭证。

## 路由与 SLA

| 风险等级 | 审批链 | 定时器 |
|---|---|---|
| L1 | R2 自决 | `PT15M` |
| L2 | R3 / R5 单签，或 R3→R5 串行会签 | 每个节点 `PT30M` |
| L3 | R3 初审 → R1 终审 | 初审 `PT20M`，终审 `PT25M` |
| L4 | R3 初审 → R1 终审 → COO/经委会特批 | `PT15M` / `PT15M` / `PT30M` |

> L3/L4 终审的定时器为固定值（总 SLA 的近似实现）；生产环境如需严格绝对截止，可在任务前用脚本计算剩余 SLA 并改为动态定时器，或在流程级增加总 SLA 监控。定时器为非中断型，只触发催办，不自动通过/驳回。

## 依赖

1. **用户组**：泳道名即用户组标识，需在后端权限文件中创建并加入账号：
   `scm_r2_planner`、`scm_r3_sop_manager`、`scm_r5_plant_manager`、`scm_r1_scm_vp`、`scm_group_coo`、`scm_committee_director`、`scm_group_office`、`scm_risk_legal`。
2. **Connector Proxy**：`ServiceTask_SyncApproved`、`ServiceTask_BroadcastRejection`、`ServiceTask_SendEscalation` 使用 `http/PostRequest`，需要部署 `spiffworkflow-connector` 并设置后端 `SPIFFWORKFLOW_BACKEND_CONNECTOR_PROXY_URL`（本仓库 `docker-compose.yaml` 已补充）。
3. **权限文件**：如未定义上述用户组，任务可能无人可认领；测试时可给测试账号加入对应组，或在实例数据中通过 `lane_owners` 临时映射。

## 本地验证

已使用 SpiffWorkflow 3.2.0 解析器验证：

- BPMN + DMN 可成功解析，58 个任务节点；
- DMN 17 组用例（L1~L4、边界值、硬触发）全部通过；
- L1/L2（R3/R5/会签）/L3/L4 均可正确路由到首个审批节点。

## 已知限制

- L2 会签按 **R3→R5 串行** 建模，保证两人均签署；如需并行，可改为并行网关 + 汇聚。
- 终审/特批定时器为固定值，见上。
- 每个审批节点未单独调用 Case 同步接口；通过/驳回时统一同步。如需逐节点同步，可在每个用户任务前增加 `ServiceTask_SyncNode`。
- 失败时 `fault_or_suspend_on_exception=suspend`，审批实例会挂起等待人工处理，不会静默丢失。
- 通知/同步服务任务失败会按异常策略挂起；生产建议为这些服务任务配置错误边界事件，或由 connector/通知网关内部重试，确保通知失败不阻塞主审批任务。
