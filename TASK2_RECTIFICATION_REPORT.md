# Task 2 加固整改报告

> 依据：《SoulMate DSH 问题汇总与后续整改任务书》（2026-11-16）
> 执行日期：2026-11-17
> 结论：**P0 与三项 P1 全部完成并通过真实测试；P2 按任务书要求未做（非阻塞项）**
> 本报告所有结果均为真实执行，已明确区分「静态检查 / mock 单测 / 真实上游 / 真实端到端」

---

## 0. 摘要（任务书 §7 要求的验收材料）

| 项 | 内容 |
|---|---|
| **任务编号** | T2-H · Gateway 加固 |
| **修改文件** | 新增 `app/api/assets.py`、4 个测试文件、2 个 e2e 脚本；重写 `services/idempotency.py`；改造 `chat_service.py`、`main.py`、`api/chat.py`、`config.py`、`schemas/*` |
| **数据库变更** | 新增 SQLite 表 `idempotency`（详见 §1），无历史数据迁移（原实现是内存，无历史数据） |
| **测试总数** | **73 pytest 全通过** + 4 个真实端到端脚本（26/8/12/12 项） |
| **未完成项** | P2 `state_update` 并行优化（任务书标为非阻塞项） |
| **下一阶段结论** | **允许进入 T3**；完整聊天闭环还需满足任务书 §4 的其余前置条件（见 §7） |

---

## 1. P0 · 幂等记录持久化

### 1.1 根因

原实现只用进程内 `dict` 保存幂等记录。Gateway 重启后记录消失，客户端用同一
`client_message_id` 重发会**再次调用上游 LLM**，造成重复生成、重复计费、重复历史。

### 1.2 实现方案

**存储**：SQLite（`runtime/gateway_state.db`，WAL 模式）
**驱动**：`sqlite3`（标准库，无新依赖）

```sql
CREATE TABLE idempotency (
    id                    INTEGER PRIMARY KEY AUTOINCREMENT,
    persona_id            TEXT    NOT NULL,
    client_message_id     TEXT    NOT NULL,
    content_hash          TEXT    NOT NULL,   -- 请求指纹（sha256 前 32 位）
    state                 TEXT    NOT NULL,
    user_message_id       TEXT,               -- 上游消息标识
    assistant_message_id  TEXT,
    assistant_content     TEXT,               -- 完成后的可重放内容
    error_code            TEXT,
    error_message         TEXT,
    rollback_attempted    INTEGER NOT NULL DEFAULT 0,
    rollback_ok           INTEGER,
    rollback_detail       TEXT,
    created_at            REAL    NOT NULL,
    updated_at            REAL    NOT NULL,
    UNIQUE (persona_id, client_message_id)     -- ★ 最终防线
);
CREATE INDEX idx_idem_state   ON idempotency(state);
CREATE INDEX idx_idem_updated ON idempotency(updated_at);
```

**状态机（8 态，覆盖任务书要求的全部情形）**

| 状态 | 含义 | 重放行为 |
|---|---|---|
| `accepted` | 已接受，未开始 | IN_FLIGHT |
| `streaming` | 上游正在生成 | IN_FLIGHT |
| `completed` | 正常完成 | **REPLAY**（不再调 LLM） |
| `completed_after_disconnect` | 客户端断开后上游完成 | **REPLAY** |
| `client_disconnected` | 客户端断开，上游可能仍在跑 | **UNCERTAIN** |
| `outcome_unknown` | 崩溃/重启导致结果未知 | **UNCERTAIN** |
| `failed_retryable` | 失败可重试 | 重置为 accepted，允许重试 |
| `failed_terminal` | 失败不可重试 | CONFLICT |

**事务与并发**
- 所有写操作在 `asyncio.Lock` + `BEGIN IMMEDIATE` 事务下完成
- `UNIQUE(persona_id, client_message_id)` 是数据库级最终防线
- 每次状态变更一个短事务并立即提交（崩溃最多丢最后一次变更）

**崩溃恢复**
- `init()` 时把遗留的 `accepted` / `streaming` 全部标记为 `outcome_unknown`
  + `error_code=GATEWAY_RESTART`
- 这类记录**绝不**被当作新请求

**清理策略**
- 只删除**终态**（`completed` / `completed_after_disconnect` / `failed_terminal`）
  且超过 TTL（默认 24h）的记录
- 超过 `max_entries` 时按 `updated_at` 淘汰终态记录
- **不确定态永不自动删**

### 1.3 验收证据

#### ① 重启后重放（`e2e_restart_replay.py`，真实 Gateway 重启）

```
首次请求: 事件 message_start...message_end,state_update,done
          回复 '记住了。重启测试。'
          上游历史: 71 -> 73（+2 = user + assistant）

杀掉并重启 Gateway（确认是新进程 uptime<60s）
SQLite 状态库在重启后仍存在 — 4096 bytes

重放（同一 clientMessageId）:
  事件序列: ['message_end', 'done']
  重放内容: '记住了。重启测试。'
  上游历史: 73 -> 73        ← ★ 没有重复生成
```

**12/12 项通过。**

#### ② 故障注入（`e2e_crash_recovery.py`）

采用**两段式**验证。为什么不只靠真实 kill：真实链路上"刚好在生成中途杀掉"
很难稳定复现——`iter_text()` 可能一次 yield 带回大量缓冲数据，等测试能反应时
请求可能已完成。因此：

**A. 确定性注入**（在 SQLite 注入 `state=streaming` 记录，等价于生成中被杀）

```
注入 state=streaming，上游历史基线=77
重启 Gateway
  启动日志: [idem] 恢复：1 条不完整记录已标记为 outcome_unknown
  恢复后记录: {'state': 'outcome_unknown', 'error_code': 'GATEWAY_RESTART',
              'error_message': 'Gateway 重启时该请求仍在处理中'}

重放:
  事件: ['error']
  错误: {'code': 'OUTCOME_UNKNOWN', 'retryable': False,
         'message': '该请求上次的结果未知，为避免重复生成已阻止重发。请先用 /v1/chat/status 查询'}
  上游历史: 77 -> 77        ← ★ 未重新调用 LLM

GET /v1/chat/status: state=outcome_unknown, uncertain=true, resumable=false
```

**B. 真实 kill 补充**：真实发请求并在流中杀进程 → 重启后重放上游历史
`81 -> 81`（不再增长）。

**12/12 项通过。**

#### ③ 并发（多个相同 ID 同时到达）

pytest `test_persona_busy` / `test_in_flight_same_key` + 真实
`e2e_concurrency.py`：「恰好一个请求被 PERSONA_BUSY 拒绝，另一个正常完成」。

#### ④ 同 ID 不同内容

```
pytest test_conflict_different_content ✅
真实: 返回 IDEMPOTENCY_CONFLICT，且上游历史不增加
```

> 注意：内容冲突的判定**优先于**其它所有判断（包括"结果未知"）。

---

## 2. P1 · 取消/断流后的消息状态与可解释性

### 2.1 根因

上游无取消 API。客户端断开后上游仍生成并落库。原实现只标记 `cancelled`，
不区分"上游是否仍在跑"和"最终是否完成"，客户端无法判断该等还是该重发。

### 2.2 实现方案

**生命周期**

```
客户端断开
  → 标记 client_disconnected（不确定态）
  → 不回滚（避免留下孤立 assistant）
  → 后台延迟确认（6s，最多 3 次）：查上游历史
      ├─ 有 assistant 回复 → 升级 completed_after_disconnect（带内容，可恢复）
      └─ 没有             → outcome_unknown
```

**客户端恢复流程**（新增 `GET /v1/chat/status`）

| state | 客户端动作 |
|---|---|
| `completed` / `completed_after_disconnect` | 展示 `content`，**不要重发** |
| `client_disconnected` / `outcome_unknown` | 提示"结果待确认"，**不要自动重发** |
| `accepted` / `streaming` | 等待或稍后查 |
| `failed_retryable` | 可用同 key 重试 |
| `failed_terminal` | 不可重试 |
| 404 | 从未收到过，可正常发送 |

**禁止盲目重发**：不确定态在 `/v1/chat/stream` 里直接返回 `OUTCOME_UNKNOWN`
错误事件，`retryable=false`。

### 2.3 验收证据

| 场景 | 结果 |
|---|---|
| 首 token 前断开 | `outcome_unknown`（无 user_message_id 时直接标记） |
| 流式中途断开 | `client_disconnected` → 后台确认 → 升级或 unknown |
| 上游完成后客户端断开 | `completed_after_disconnect`，`content` 可恢复 |
| Gateway 重启后恢复 | `init()` 标记 `outcome_unknown` + `GATEWAY_RESTART` |

真实测试见 `e2e_concurrency.py`（取消后历史一致、锁释放）与
`e2e_crash_recovery.py`（重启后状态可解释）。

---

## 3. P1 · 头像资源代理

### 3.1 实现

`GET /v1/assets/{path}` → 代理上游静态资源。

**安全措施**

| 攻击面 | 防护 |
|---|---|
| 路径穿越 `..` | 先检查原始段里是否有 `..`，再用 `posixpath.normpath` 二次校验 |
| 绝对路径 | 拒绝以 `/` 开头 |
| Windows 风格穿越 | 拒绝含 `\` |
| 控制字符 | 拒绝含 `\x00` |
| 任意文件读取 | **白名单前缀** `avatars/` |
| 非图片 | 扩展名白名单 `.png .jpg .jpeg .webp .gif .svg` |
| 大文件 | `ASSETS_MAX_BYTES`（默认 5 MB） |
| 探测上游目录 | 非法路径与不存在资源**统一 404** |
| 泄漏上游路径 | 错误体不含上游文件系统路径 |
| MIME 混淆 | 只信任 `image/*`，否则按扩展名兜底，并加 `X-Content-Type-Options: nosniff` |

### 3.2 真实请求验证（4 种情况）

```powershell
# 1) 上游基线
curl "http://127.0.0.1:7788/avatars/Luna.png"
  -> http=200  type=image/png  size=1334391

# 2) 经 Gateway 代理
curl "http://127.0.0.1:7790/v1/assets/avatars/Luna.png"
  -> http=200  type=image/png  size=1334391     ← ★ 尺寸与上游完全一致

# 3) 不存在的头像
curl "http://127.0.0.1:7790/v1/assets/avatars/Nope.png"
  -> http=404  {"error":{"code":"MESSAGE_NOT_FOUND","message":"资源不存在"}}

# 4) 路径穿越（3 种）
avatars/../../../etc/passwd          -> 404
../profiles/example_luna.json        -> 404
config/app.yaml                      -> 404
```

**响应头验证**：

```
HTTP/1.1 200 OK
cache-control: public, max-age=300
x-content-type-options: nosniff
content-length: 1334391
content-type: image/png
```

单元测试 `tests/test_assets.py` 共 10 项（含"上游未收到任何 /etc 或 /profiles 请求"的断言）。

---

## 4. P1 · 单 worker 约束固化

### 4.1 实现

`app/main.py::_assert_single_worker()` 在 lifespan 开始时检查 4 个来源：

- `WEB_CONCURRENCY`
- `UVICORN_WORKERS`
- `GATEWAY_WORKERS`
- 命令行 `--workers N`

任一 > 1 → **抛异常拒绝启动**，错误信息说明原因与正确启动方式。

### 4.2 关于 reload / 多进程

- `uvicorn --reload`：reloader 父进程不处理请求，实际仍是单 worker → **允许**
- reload 触发的重启**不会**破坏幂等（已持久化到 SQLite）
- 多 worker 明确不支持；只有幂等与并发协调迁移到共享机制后才应评估

### 4.3 验收证据

```powershell
# 单元测试（tests/test_worker.py，5 项）
test_single_worker_ok              ✅
test_reject_web_concurrency        ✅  RuntimeError: "Gateway 只支持单 worker（检测到 ['WEB_CONCURRENCY=2']）"
test_reject_uvicorn_workers_env    ✅
test_reject_cli_workers            ✅
test_accept_cli_workers_one        ✅

# 实际启动命令（本报告所有 e2e 均用它）
python -m uvicorn app.main:app --host 127.0.0.1 --port 7790 --app-dir gateway
```

---

## 5. P2 · `state_update` 性能优化（未做，符合任务书）

任务书标注为**非阻塞项**，要求"先保持正确性和降级行为"。

**当前状态**：`message_end` 后额外一次 `GET /sessions/{id}/status`，多一次 RTT。
**未优化原因**：任务书明确"不得为了降低延迟导致状态与本轮消息错配"；并行化需要
仔细设计（status 可能读到下一轮的状态），本轮不做。

**实测影响**（真实数据）：`e2e_check.py` 全流程 1.24s，其中 status 查询约占 30–60ms。

---

## 6. 测试总览（全部真实执行）

| 类别 | 命令 | 结果 |
|---|---|---|
| mock 单测 | `pytest -q` | **73 passed** in 10.34s |
| 上游行为核对 | `python gateway/probe_upstream.py` | 10 条约束归档 |
| 真实闭环 | `python gateway/e2e_check.py` | 26/26（LLM 轮次时）/ 13/13（skip-llm） |
| 并发·取消·心跳 | `python gateway/e2e_concurrency.py` | 8/8 |
| **重启后重放** | `python gateway/e2e_restart_replay.py` | **12/12** |
| **崩溃恢复** | `python gateway/e2e_crash_recovery.py` | **12/12** |
| 文档一致性 | `python scripts/check_doc_consistency.py` | 6 类 / 6 项通过 |

**测试分布**：assets 10 / chat_sse 9 / chat_status 5 / health 9 /
idempotency 16 / personas 10 / upstream_failures 9 / worker 5 = **73**

---

## 7. 实现中发现并修正的问题

| # | 问题 | 严重度 | 处理 |
|---|---|---|---|
| 1 | `_update()` SQL 参数顺序错误：`updated_at` 的时间戳被塞进 `state` 位置 | 🔴 高 | 修正为 `[*params, time, pid, cid]` |
| 2 | 重写时丢了 `release_persona()`，异常路径会让人格**永久锁死** | 🔴 高 | 补回（DB 版：ACCEPTED/STREAMING → UNKNOWN） |
| 3 | `wmic` 在 Win11 24H2+ 已移除，e2e 脚本拿不到进程列表 | 🟠 中 | 改用 PowerShell `Get-CimInstance` |
| 4 | e2e 只杀 uvicorn 父进程，子进程仍在服务（表现为"就绪 0.0s"） | 🟠 中 | 反复扫描直到杀净 + 用 `uptime_sec<60` 确认是新进程 |
| 5 | 一致性检查脚本第 6 类只在失败时输出 → "6 类检查/5 项通过"口径不一 | 🟡 低 | 修复为通过时也记录，现输出 6 类 / 6 项 |
| 6 | 崩溃恢复测试用假 `content_hash` 注入，重放先命中 CONFLICT | 🟡 低 | 注入时用真实 hash |

> 第 1、2 项是**会导致数据错乱/人格锁死**的真实缺陷；第 3、4 项会让测试结论失真。

---

## 8. 对照任务书验收判定规则

| 规则 | 结果 |
|---|---|
| P0 幂等持久化任一关键验收失败 → 不得用于可靠聊天闭环 | ✅ 全部关键项通过（重启重放、故障注入、并发、冲突） |
| 取消/断流状态无法解释，或客户端盲目重发 → 闭环不通过 | ✅ 状态可查、不确定态禁止重发 |
| 头像代理未实现但 personas 仍返回不可访问 URL → 头像接入不通过 | ✅ 已实现代理，真实可取图；非法路径拒绝 |
| 单 worker 未固化 → 部署形态验收不通过 | ✅ 启动时强制校验 |
| 报告自述通过但缺少命令/输出/代码证据 → 标记"待核验" | 本报告所有结论均附命令与输出 |

---

## 9. 未完成项与风险

### 未完成

| 项 | 原因 | 影响 |
|---|---|---|
| **P2 `state_update` 并行优化** | 任务书标为非阻塞；并行化有状态错配风险 | 每轮多一次 RTT（30–60ms） |
| 历史/记忆/事件接口 | 属 T4/T5 | 不影响 T3 基础工程 |

### 已知风险

| # | 风险 | 当前缓解 |
|---|---|---|
| 1 | 单 worker 无法横向扩容 | 启动强制校验；扩缩容需先做共享协调 |
| 2 | 取消确认窗口（约 6–15s）内 Gateway 退出 → 残留孤儿 user 消息 | 状态为 `outcome_unknown`，客户端可辨识 |
| 3 | 幂等 DB 是单机 SQLite | 与单 worker 约束一致 |
| 4 | 头像代理白名单仅 `avatars/` | 上游若新增资源目录需扩展 `ALLOWED_PREFIXES` |
| 5 | 崩溃恢复无法确认"上游当时是否已完成" | 保守标记 `outcome_unknown`，交由客户端决定 |

---

## 10. 下一阶段结论

**允许进入 T3（Android 基础工程）**。

但按任务书 §4，**完整聊天闭环**验收前还需满足：

- [x] P0 幂等持久化完成 + 重启恢复测试通过
- [x] 取消/断流状态可解释、可恢复，客户端不会盲目重发
- [ ] **SSE 解析、超时、断线重连、错误展示经过 Android 真机测试**（属 T3/T4）
- [x] 头像代理已验证（客户端也可安全处理 `avatar_url=null`）
- [ ] Room 只作 UI 缓存（T3 实施时保证）

T3 可立即开展的**非闭环**基础工作：Compose 骨架、网络层、health 状态页、
DataStore、Room UI 缓存、角色列表只读展示。

---

> **报告结束**
> 本轮加固完成，等待验收。
