# Codex Workboard Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Extend Lore into a local Codex development workboard that splits multi-request sessions into trackable work items and correlates them with Git and deployment evidence.

**Architecture:** Add a framework-free `lore.workboard` service backed by a dedicated SQLite database. Reuse the existing Codex extractor and read-only Git commands, expose the service through a Flask Blueprint, an SSE stream, and a new MCP tool group.

**Tech Stack:** Python 3.11+, SQLite WAL, Flask/Jinja, existing Lore Codex extractor, stdlib subprocess and threading.

---

### Task 1: Add the Workboard storage contract

**Files:**
- Create: `lore/workboard/__init__.py`
- Create: `lore/workboard/models.py`
- Create: `lore/workboard/storage.py`
- Test: `tests/test_workboard_storage.py`

**Steps:**

1. Define status constants and typed serialization helpers.
2. Create an idempotent SQLite schema with sources, work items, events and metadata.
3. Implement upsert, list, detail, summary and event append operations with WAL and mode 0600.
4. Add tests for migration, upsert idempotence, filtering and event ordering.
5. Run `pytest tests/test_workboard_storage.py -q`.
6. Commit with `新增：建立 Codex Workboard 本地存储`.

### Task 2: Add Codex requirement segmentation and evidence extraction

**Files:**
- Create: `lore/workboard/codex.py`
- Create: `lore/workboard/scanner.py`
- Test: `tests/test_workboard_codex.py`

**Steps:**

1. Parse changed Codex rollout files through `CodexExtractor` and preserve source metadata.
2. Split visible user prompts into stable work item IDs and attach following assistant messages.
3. Extract explicit completion, blocker, test and deployment evidence without treating missing evidence as success.
4. Persist source fingerprints and work items transactionally.
5. Add synthetic JSONL tests covering multiple prompts, aborted turns, tests and deployment failures.
6. Run `pytest tests/test_workboard_codex.py -q`.
7. Commit with `新增：拆分 Codex 会话需求并提取完成证据`.

### Task 3: Add read-only Git correlation

**Files:**
- Create: `lore/workboard/git_state.py`
- Modify: `lore/workboard/scanner.py`
- Test: `tests/test_workboard_git_state.py`

**Steps:**

1. Read branch, SHA, status, upstream and ahead/behind through bounded Git subprocess calls.
2. Preserve separate session-time and current-worktree values.
3. Return explicit unknown/error fields for non-repositories and command failures.
4. Add tests with a temporary Git repository and a non-repository directory.
5. Run `pytest tests/test_workboard_git_state.py -q`.
6. Commit with `新增：关联 Workboard 的 Git 工作区状态`.

### Task 4: Add refresh service and watcher

**Files:**
- Create: `lore/workboard/service.py`
- Create: `lore/workboard/watcher.py`
- Modify: `lore/workboard/storage.py`
- Test: `tests/test_workboard_service.py`

**Steps:**

1. Scan Codex files incrementally and refresh Git state for affected projects.
2. Keep source failures as explicit source errors and preserve existing rows.
3. Add a bounded polling watcher that emits revision events to registered listeners.
4. Add tests for first scan, unchanged scan, changed scan and source failure behavior.
5. Run `pytest tests/test_workboard_service.py -q`.
6. Commit with `新增：实现 Workboard 增量刷新和实时监听`.

### Task 5: Add Web UI and APIs

**Files:**
- Create: `lore/interfaces/routes/workboard.py`
- Create: `lore/templates/workboard.html`
- Modify: `lore/interfaces/routes/__init__.py`
- Modify: `lore/interfaces/web.py`
- Test: `tests/test_workboard_web.py`

**Steps:**

1. Register the Workboard Blueprint and render a board with summary cards, filters and a detail drawer.
2. Add JSON endpoints, local-origin protection for refresh and an SSE endpoint with keepalive messages.
3. Start the Workboard watcher only when explicitly enabled by configuration.
4. Add API and SSE tests with a temporary output directory.
5. Run `pytest tests/test_workboard_web.py -q`.
6. Commit with `新增：提供 Codex Workboard 实时网页看板`.

### Task 6: Add MCP tools and documentation

**Files:**
- Create: `lore/interfaces/mcp_tools/workboard.py`
- Modify: `lore/interfaces/mcp.py`
- Modify: `docs/WEB-UI.md`
- Modify: `docs/MCP.md`
- Modify: `docs/CONFIGURATION.md`
- Test: `tests/test_mcp_workboard.py`

**Steps:**

1. Register Workboard summary, list, detail and refresh tools using the existing MCP dependency bundle.
2. Ensure tool results read the same database as the Web UI and return explicit errors.
3. Document `LORE_WORKBOARD_WATCH`, the local URL, data location and evidence semantics.
4. Add MCP protocol tests and run `pytest tests/test_mcp_workboard.py -q`.
5. Commit with `新增：通过 MCP 查询 Codex Workboard`.

### Task 7: Run validation and publish the feature branch

**Steps:**

1. Run focused Workboard tests.
2. Run the existing Lore test suite with the repository's normal command.
3. Run `ruff check`, `mypy` and the documentation drift checks required by `AGENTS.md`.
4. Review `git diff`, `git status` and branch tracking state.
5. Push `feature/2026-11-02-codex-workboard` to `origin`.
6. Report exact test results and any evidence that remains unknown.
