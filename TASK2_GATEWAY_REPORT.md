# Task 2 完成报告 — Gateway 最小版

> 依据：《SoulMate DSH 整改意见与 Task 2 启动任务书》第二部分
> 执行日期：2026-11-15
> 结论：**三个接口全部实现并真实运行；49 个自动化测试 + 34 项真实端到端验证全部通过**

---

## 0. 报告摘要（任务书 §16 模板）

| 项 | 内容 |
|---|---|
| **任务编号** | T2 · Gateway 最小版 |
| **本次目标** | 在真实 Shikigami + 真实 LLM 环境中，通过自建 FastAPI Gateway 完成人格列表读取和一次完整 SSE 流式聊天 |
| **修改文件** | 新增 `gateway/` 全套源码（20 个 Python 文件）+ 3 个验证脚本 + 3 份文档；更新 `API_SPEC.md`、`DSH_TASKS.md`、`scripts/check_doc_consistency.py` |
| **执行环境** | Windows / Python 3.13.15 / FastAPI 0.163.1 / httpx / Shikigami v0.12.0 / DeepSeek `deepseek-flash` |
| **是否更新 DSH_TASKS.md** | ✅ 已更新（T2 → 完成，含 15 条完成记录） |
| **是否满足验收标准** | **是** |

---

## 1. 任务书 §8：开发前必须核对的上游真实行为

**这一步先做，没有照抄 API_SPEC。** 完整记录见 `gateway/UPSTREAM_VERIFICATION.md`。

### 核对方式

```powershell
cd gateway
..\..\runtime\venv\Scripts\python.exe probe_upstream.py --persona example_luna
```

该脚本**不经过任何 Gateway 代码**，直连真实 Shikigami。

### 7 个与 `API_SPEC.md` 假设不符的实测结果

| # | API_SPEC 原假设 | 实测 | 已处理 |
|---|---|---|---|
| 1 | `state_update` 由上游提供 | ❌ 上游 SSE **完全没有状态事件** | Gateway 在 `message_end` 后主动查 `/sessions/{id}/status` |
| 2 | 可按 HTTP 状态码判断错误 | ❌ 上游用 **HTTP 200 + `{"error":...}`** | 解析 SSE `error` 字段 |
| 3 | `Origin` 可透传 | ❌ 跨源 `Origin` / `Sec-Fetch-Site: cross-site` → **403** | Adapter 统一剥离 |
| 4 | `/chat_history` 正序 | ❌ **倒序**（实测 id 7,6,5,4,3） | 记录；历史接口属 T4 |
| 5 | 单次 LLM TTFT ≈ 4s | 实测 **1.00 / 1.69s**（波动） | 心跳按"空闲 15s"触发，不绑死 TTFT |
| 6 | `avatar` 可直接下发 | ❌ 是相对路径 `avatars/Luna.png` | 转 `/v1/assets/*`（代理未实现，见限制） |
| 7 | `/sessions` 可作人格列表 | ❌ 只含已加载人格 | 用 `/profiles` |

### 10 条对实现的硬性约束（已全部落实）

1. 剥离 `Origin` / `Sec-Fetch-Site`
2. 错误看 SSE 内容
3. `done:true` 拆成 `message_end` + `done`
4. `state_update` Gateway 自己补，失败降级
5. `/chat_history` 反转（T4）
6. `avatar` 转 URL
7. 心跳 `: ping`，不伪装成 `text_delta`
8. 失败回滚 `DELETE /sessions/{id}/messages/by-id/{mid}`
9. 回滚失败记录，不静默
10. 同 persona 串行化

---

## 2. 实现内容

### 2.1 目录结构（按任务书 §9）

```
gateway/
├── app/
│   ├── main.py                    # FastAPI + 异常归一化 + lifespan
│   ├── config.py                  # 环境变量 + 随机令牌
│   ├── api/{health,personas,chat}.py
│   ├── adapters/shikigami/
│   │   ├── client.py              # HTTP/SSE（剥离头、超时、连接池）
│   │   ├── mapper.py              # 上游 → Gateway 契约
│   │   └── capabilities.py        # 上游能力探测
│   ├── schemas/{common,persona,chat}.py
│   ├── services/
│   │   ├── health_service.py      # 健康聚合 + LLM 三态
│   │   ├── chat_service.py        # 流式编排
│   │   └── idempotency.py         # 幂等状态机 + 并发闸门
│   └── security/auth.py           # 本机令牌
├── tests/                         # 49 个 pytest
├── probe_upstream.py              # §8 核对脚本
├── e2e_check.py                   # 真实端到端（26 项）
├── e2e_concurrency.py             # 并发/取消/心跳（8 项）
├── UPSTREAM_VERIFICATION.md
├── README.md
└── requirements.txt
```

### 2.2 三个必须实现的接口

#### `GET /v1/health`

- 聚合 `/version` + `/api/engine_warnings`
- LLM 三态：`true` / `false` / **`null(unknown)`**
  - `observed`：最近一次真实对话结果（5 分钟内）
  - `configured`：配置里 `has_key=false` → 判定不可用
  - `unknown`：配置齐全但从未请求过 → **不伪报 true**
- 严格超时（3s）+ 缓存（5s / 60s）
- 实测响应 48 ms

#### `GET /v1/personas`

- 数据源 `/profiles`（**非** `/sessions`）
- `load_into_chat=false` 的人格也会列出
- `base_prompt` / `style_constraint` **不下发**
- `avatar` 转 `/v1/assets/...`
- `is_current` 来自 `/sessions`（唯一提供该字段的接口），失败则省略

#### `POST /v1/chat/stream`

6 类事件 + 心跳 + 状态补充 + 幂等 + 并发 + 回滚 + 取消，全部实现。

---

## 3. 真实执行记录

### 3.1 自动化测试（mock 上游，快速确定）

```powershell
cd gateway
..\..\runtime\venv\Scripts\python.exe -m pytest -q
```

```
collected 49 items
tests\test_chat_sse.py .........                                         [ 18%]
tests\test_health.py .........                                           [ 36%]
tests\test_idempotency.py ............                                   [ 61%]
tests\test_personas.py ..........                                        [ 81%]
tests\test_upstream_failures.py .........                                [100%]
============================= 49 passed in 7.11s ==============================
```

### 3.2 真实上游行为核对

```powershell
..\..\runtime\venv\Scripts\python.exe probe_upstream.py
```

关键输出：
```
/profiles -> 4 个人格
/sessions/{id}/status -> emotion_state + affinity_state + available_emotions
Origin=跨源 -> HTTP 403
Sec-Fetch-Site: cross-site -> HTTP 403
SSE: 3 种事件（user_ack / token / done）
不存在的 session -> HTTP 200 + {"error":"No active session","done":true}
DELETE by-id -> 200 {"ok":true}，total 10->9
DELETE 不存在 -> 404 {"detail":"Message not found"}
```

### 3.3 真实端到端（真实 Shikigami + 真实 LLM）

```powershell
..\..\runtime\venv\Scripts\python.exe e2e_check.py
```

```
合计 26 项：通过 26，失败 0
```

关键数据：

| 指标 | 实测值 |
|---|---|
| health 耗时 | **48 ms** |
| health 初始 LLM 状态 | `available=null, confidence=unknown`（诚实） |
| personas 数量 | 4（与上游 `/profiles` 一致） |
| **真实 LLM TTFT** | **1.00 s** |
| 总耗时 | 1.24 s |
| 事件数 | 14（1 start + 11 delta + end + state + done） |
| 事件序列 | `message_start → text_delta×11 → message_end → state_update → done` |
| 回复 | 「晚安。累了就早点放下吧，我在。」 |
| state_update | 非降级，真实状态（sad/tired, energy=61.7） |
| 幂等重放 | **0.04 s**，事件仅 `message_end + done`，未再调上游 |
| 幂等冲突 | 返回 `IDEMPOTENCY_CONFLICT` |
| 对话后 health | `available=true, confidence=observed` |

### 3.4 真实并发 / 取消 / 心跳

```powershell
..\..\runtime\venv\Scripts\python.exe e2e_concurrency.py
```

```
合计 8 项：通过 8，失败 0
```

| 项 | 实测 |
|---|---|
| 同人格并发 | 两个请求**同时竞争**，恰好一个 `PERSONA_BUSY`，另一个正常完成 |
| 客户端取消 | 历史一致（user 后紧邻 assistant），锁已释放 |
| 心跳 | 默认 15s 下 TTFT 1-2s 不触发（正确）；`SSE_HEARTBEAT_INTERVAL=0.5` 时**实测收到 `: ping`** |

**心跳原始报文取证**（`SSE_HEARTBEAT_INTERVAL=0.5`）：

```
data: {"type":"message_start",...}

: ping

data: {"type":"text_delta","delta":"夜"}
```

心跳是 SSE 注释帧，**未混入文本**。

---

## 4. 实现过程中发现并修正的问题

| # | 问题 | 严重度 | 处理 |
|---|---|---|---|
| 1 | **客户端取消时回滚会破坏数据**：上游无取消 API，断开后仍生成并落库 assistant。回滚 user 会留下"没有提问的孤立 assistant" | 🔴 高 | 改为**取消不回滚**，记录 `rollback_detail`，写入 README 已知限制 |
| 2 | 错误体嵌套层次不一致：`make_error()` 返 `{"error":{...}}`，但异常处理器期望 error 体 | 🟠 中 | 新增 `error_body()`，api 层统一使用 |
| 3 | **并发测试假阳性**：`httpx.AsyncClient` 默认 `trust_env=True`，Windows 上初始化 **2–6 秒**，导致两个"并发"请求实际错开数秒 | 🟠 中 | 测试改为 `trust_env=False` + 复用 client；Gateway 代码本就设了 `trust_env=False` |
| 4 | 取消测试断言过时（原断言"必须回滚"） | 🟡 低 | 改为验证"历史一致性" |
| 5 | `get_history` 有 `if False else` 的丑陋写法 | 🟡 低 | 改为直接调用 |
| 6 | `personas.py` 用 `client._get_json` 私有方法 | 🟡 低 | client 新增公开 `get_sessions()` |
| 7 | `main.py` 用 `__import__("fastapi")` 动态导入 | 🟡 低 | 改为正常 import |
| 8 | `Settings` 为 `frozen=True` 导致测试无法覆盖超时/心跳 | 🟡 低 | 改为普通 dataclass |

> 第 1、3 项是**真实缺陷**，不是笔误；其余是代码质量修正。

---

## 5. 对照任务书 §15 完成定义

| 条件 | 结果 | 证据 |
|---|---|---|
| 三个接口真实运行，不只是代码 | ✅ | `e2e_check.py` 真实 HTTP |
| 通过真实 Shikigami + 真实 LLM 完成一次 Gateway SSE 流式对话 | ✅ | TTFT 1.00s，14 事件 |
| 心跳有测试证据 | ✅ | mock 精确测试 + 真实 `: ping` 报文 |
| 状态补充有测试证据 | ✅ | 非降级 + 降级两条路径 |
| 同人格并发有测试证据 | ✅ | mock + 真实双路竞争 |
| 幂等有测试证据 | ✅ | 12 单元测试 + 真实重放 0.04s |
| 错误处理有测试证据 | ✅ | 连接错误 / 业务错误 / 无 done 三类 |
| 回滚有测试证据 | ✅ | 成功 / 404 / 网络错误三种 |
| 关键自动化测试已运行并报告 | ✅ | 49 passed |
| 无密钥泄漏 | ✅ | 令牌文件已 gitignore；日志不打印令牌与正文 |
| 默认仅 localhost 监听 | ✅ | `127.0.0.1:7790` |
| 无明显资源泄漏 | ✅ | httpx 连接池在 lifespan 关闭 |
| 文档、代码、DSH_TASKS 一致 | ✅ | `check_doc_consistency.py` 5 项全过 |

---

## 6. 未通过项

**无。**

---

## 7. 已知限制（详见 `gateway/README.md` §7）

| # | 限制 | 影响 |
|---|---|---|
| 1 | 幂等仅内存实现 | Gateway 重启后记录丢失，重发会重复生成 |
| 2 | 上游无取消 API | 客户端取消后上游仍生成并落库；Gateway 不回滚（避免破坏数据） |
| 3 | 取消可能残留孤儿 user 消息 | 若上游未生成完，历史里会有无回复的提问 |
| 4 | `state_update` 需额外一次上游请求 | `message_end` 后多一次 RTT |
| 5 | **`/v1/assets/*` 头像代理未实现** | `avatar_url` 目前不可直接访问 |
| 6 | 未实现历史/记忆/事件接口 | 属 T4/T5 |
| 7 | 仅支持单 worker | 多 worker 会各自持有幂等表 |

---

## 8. 下一步建议

1. **T2 验收**：建议重点检查 `gateway/UPSTREAM_VERIFICATION.md`（§8 核对是否到位）与 `e2e_concurrency.py` 的并发证据。
2. **T3（Android 基础工程）**：Gateway 契约已稳定，可开始。需要注意 OkHttp `readTimeout ≥ 120s`（TTFT 1-2s + 长回复）。
3. `/v1/assets/*` 头像代理建议排在 T3 一起做（Android 要显示头像）。
4. 若 T2 验收通过，可考虑把 `state_update` 的额外查询改为**与 `message_end` 并行**，省一次 RTT。

---

> **报告结束**
> Task 2 完成，等待验收。**未开始 T3。**
