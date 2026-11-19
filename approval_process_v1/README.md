# 分级审批 BPMN v1

流程定义与导入说明。应用侧只通过 SpiffArena REST 启动实例、查询任务、完成任务。

## 文件

- `approval_process.bpmn`：主流程（L1 自动 / L2 顺序双签 / L3 单签）
- `forms/approval_task.form.json`：用户任务表单

## 导入

1. 启动 SpiffArena backend 与其持久化 DB；
2. 打开 SpiffArena Frontend → Process Groups → `scct`；
3. 上传 `approval_process.bpmn`，在 Modeler 中补图形布局并给三个 `Task_Sign_*` 绑定表单；
4. 确认模型标识与 `SPIFF_PROCESS_MODEL_ID` 一致（默认 `scct/approval_process_v1`）；
5. 用 `GET {SPIFF_BASE_URL}/v1.0/openapi.json` 核对应用使用的 REST 路径与字段。

## 约定

- 任务定义 ID 固定为 `Task_Sign_R1` / `Task_Sign_R3` / `Task_Sign_R5`，应用据此映射审批人。
- 流程变量：`case_no` / `solution_id` / `solution_hash` / `approval_level` / `sign_type` / `approver_list` / `sla_minutes` / `rule_id` / `rule_version` / `voucher_no`。
- 最终 `approved=True` 由 Script Task 写回；若部署禁用 Script Task，可改为按结束事件推断。
- 不使用 boundary timer、message event、异步 Service Task、多实例用户任务。
