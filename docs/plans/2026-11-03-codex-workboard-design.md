# Codex Workboard 二次开发设计

## 目标

在 Lore 已有 Codex 会话归档、Web UI、SQLite 和 MCP 能力上增加一个本地全局开发看板，用来回答：

- 用户在多个 Codex 会话中提出了哪些独立需求；
- 每个需求属于哪个会话、工程目录和 Git 分支；
- 当前是否仍在运行、是否已经完成、是否阻塞或需要用户输入；
- 当前工作区是否有未提交改动、是否推送远端；
- 是否存在测试、合并、部署或上线的直接证据。

## 复用边界

- 复用 `lore.extractors.codex.CodexExtractor` 解析 Codex JSONL，避免重复实现 Codex 历史格式兼容。
- 复用 Lore 的 SQLite 连接、WAL、迁移和本地输出目录约定。
- 复用 Flask Blueprint、请求来源校验和 Jinja 模板体系。
- 复用现有 MCP Server 和 `MCPToolDeps`，增加独立的 Workboard 工具组。
- 不修改 `~/.codex/sessions`，不启动或控制 Codex，不执行 Git 写操作，不执行部署操作。

## 数据模型

新增独立数据库 `workboard.sqlite3`，避免把用户可读的任务状态和 Lore 会话索引耦合。数据库包含：

- `workboard_sources`：Codex rollout 文件路径、会话 ID、mtime、大小和最后解析时间，用于增量解析；
- `work_items`：一个会话中的一个用户 prompt 对应一条需求，保存需求文本、摘要、会话、工程、分支、提交、状态、测试证据、部署证据和更新时间；
- `workboard_events`：刷新、状态变化和来源变化的审计时间线；
- `workboard_meta`：schema 版本和最后刷新时间。

状态分类只表达证据能支持的结论：`in_progress`、`completed`、`partial`、`blocked`、`waiting_user`、`unknown`。没有直接证据时使用 `unknown`，不会因为提交数量、文件数量或时间推断上线成功。

## 解析和状态规则

扫描器只处理 `~/.codex/sessions/**/rollout-*.jsonl` 及已有 Codex extractor 支持的归档文件。增量扫描按文件 mtime 和大小判断是否需要重新解析。

一个会话按用户可见 prompt 切分成多个 Work Item。每个 Work Item 的助手回复、工具调用和错误事件作为它的证据窗口。状态推断遵循以下顺序：

1. 明确的任务完成事件和最终助手结论支持完成时，标记 `completed`；
2. 明确的失败、异常、中断或等待用户确认支持对应状态时，标记 `blocked` 或 `waiting_user`；
3. 有工具执行或正在追加的 rollout 内容但没有终态时，标记 `in_progress`；
4. 证据不足时标记 `unknown`，并保留原因。

Git 状态通过只读参数数组调用 Git，分别保存会话记录中的分支/提交和当前工程目录实时分支/提交/工作区/上游 ahead-behind。Git 不可用或目录不是仓库时明确记录 `unknown`。

部署状态只从命令执行和最终结论中提取可审计证据：成功的部署命令记录为 `evidence_found`，失败命令记录为 `failed`，没有证据记录为 `unknown`。不会将“已修改代码”或“已提交”转换成“已上线”。

## 接口和实时更新

新增：

- `/workboard`：看板页面；
- `GET /api/workboard/summary`：总览统计和最后刷新时间；
- `GET /api/workboard/items`：按状态、工程、分支和关键字过滤需求；
- `GET /api/workboard/items/<id>`：需求详情和证据时间线；
- `POST /api/workboard/refresh`：仅触发本地只读扫描；
- `GET /api/workboard/events`：SSE 长连接，文件变化后推送摘要刷新事件。

默认绑定本机。看板 watcher 使用现有轮询范式，按 `LORE_WORKBOARD_WATCH` 开启并支持独立轮询间隔；刷新失败写入日志和事件，不把失败当作空看板。

新增 MCP 工具：

- `workboard_summary`；
- `list_work_items`；
- `get_work_item`；
- `refresh_workboard`。

这些工具只读查询或触发本地只读扫描，不提供 Git、部署和会话写操作。

## 验证

- 使用合成 Codex rollout 验证一个会话拆出多个需求；
- 验证增量刷新不会重复写入或丢失旧需求；
- 验证 Git 分支、脏工作区和上游状态采集；
- 验证无部署证据时为 `unknown`，失败命令不会被标记为上线；
- 验证 Web API、SSE、页面和 MCP 工具返回同一数据库中的状态；
- 保留 Lore 原有测试通过，并运行新增 Workboard 专项测试。
