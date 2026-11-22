# R8.1 补证报告

依据：《SoulMate_R8阶段报告独立复审与R8.1补证整改任务书》（P0-01 ~ P1-08）
日期：2026-11-21 23:45（Asia/Shanghai）
性质：只补证、复跑测试、安全核验。**产品代码零改动**（见 §6 基线核对）。

---

## 0. 结论摘要

| 编号 | 项 | 状态 | 关键结果 |
|---|---|---|---|
| P0-01 | 补齐原始交付包 | ✅ | 本包含全部文档、脚本、原始日志、清单，哈希可复验 |
| P0-02 | 独立复跑 Mock PoC | ✅ | 基础 11/11 + 扩展 12/12，原始输出随包 |
| P0-03 | 核验 Shikigami 7/7 | ✅（并修正结论） | 4 条消息触发 **6 / 7 / 13** 次不等，见 §3 |
| P0-04 | 扩展真实 DeepSeek | ✅ | 5 次调用：流式/非流式/长回复/缓存复测/401，账本 4 exact + 1 unknown |
| P0-05 | 密钥安全核验 | ✅ | 明文仅 .env 与备份；**备份仍是旧 Key，待处置** |
| P1-06 | 四角色归因方案 | ⚠️ 部分 | 保留 M2；M1 未实测，仅给出可执行方案与代价 |
| P1-07 | UI 规范核对 | ✅ | R8_E_UI_SPEC.md 覆盖全部要求项 |
| P1-08 | 产品零改动基线 | ✅ | 14121 文件，`NO_DIFF` |

---

## 1. P0-01 交付包

`R8.1_bundle.zip` 结构见 `README.md`。要点：
- 所有 `.md` 文档 + `poc/*.py` + `evidence/*`（原始日志、清单、比对输出）
- `SHA256SUMS.txt` 覆盖除自身外全部载荷
- 不含 Key 明文（已扫描，见 §5）

## 2. P0-02 复跑结果

| 套件 | 用例 | 结果 | 输出 |
|---|---|---|---|
| `run_poc.py` | 11 | 11/11 | `evidence/poc_base.log` |
| `run_poc_ext.py` | 12 | 12/12 | `evidence/poc_ext.log` |

扩展用例覆盖（审查书点名项）：
- 随机/碎片 chunk 边界（7 字节硬切，切在 UTF-8 中间）
- 慢客户端（每 10ms 读 10 字节）
- 客户端 RST 取消 → `incomplete/client_aborted`
- 上游 401 / 402 / 429 / 500（含**非 JSON 错误体**）原样透传
- usage 缺失 → `unknown/no_final_usage`
- 代理 kill 后重启 → 无遗留 `open`
- 只读库（SQLite 写失败场景可复现）

## 3. P0-03 Shikigami 兼容：结论修正

原报告写"4 条消息 → 7 次上游请求"。补测三轮后必须修正：

| 轮次 | 上游请求数 | 每条分布 |
|---|---|---|
| A | 7 | — |
| B | 13 | [3,3,4,3] |
| C | 6 | [3,1,1,1] |

**修正结论：不存在固定倍率。** 数量取决于 agent 路径与后台调用阈值（情绪/亲密度/记忆）。
因此账本必须逐次记录；任何按消息数估算成本都会失真。

每条账本行含独立 `invocation_id`、时间戳、usage；`R8_C_MAPPING.json` 给出 4 条消息与上游请求的逐条对应。
后台调用**无法归属 persona**（无传播通道），这是 M2 的已知限制。

## 4. P0-04 扩展真实调用

| 用例 | 状态 | usage | 备注 |
|---|---|---|---|
| stream 短 | 200 | 35 / 29 / 64 | reasoning 27，正文"收到" |
| non-stream 短 | 200 | 35 / 32 / 67 | reasoning 30 |
| stream 长 | 200 | 59 / 495 / 554 | 496 事件，正文 316 字 |
| 长回复复测（缓存） | 200 | 59 / 600 / 659 | `cache_hit=0`，未命中 |
| 无效 Key | 401 | 无 usage | 记 `unknown`，**未伪造 exact** |

观察：`deepseek-flash` 的推理 token 占比很高（长回复 300/495、复测 465/600），
产品层 `max_tokens` 必须留足空间，否则正文为空。

缓存未命中：两次相同 prompt 发送，`prompt_cache_hit_tokens` 均为 0。说明该模型/账号下短间隔重复请求不保证命中缓存，
**不能把缓存命中当作默认优化**。

## 5. P0-05 密钥核验（详见 `evidence/key_security.md`）

| 项 | 结果 |
|---|---|
| 手机 `.env` | 新 Key，指纹 `038869f33613`，权限 600 |
| 备份 `.env.bak.20261121` | **旧 Key**，指纹 `6c780c1abe94`，权限 600 |
| 本地明文副本 | 曾出现 `.env.new`，已删除，复扫 0 命中 |
| 证据包 / ZIP | 0 命中 |
| 手机日志 / history | 未发现 Key |

待处置：备份中的旧 Key（删除备份，或吊销旧 Key）。
进程环境不可见 `DEEPSEEK_API_KEY` 属正常（dotenv 在进程内注入），已用"修改时间 + 重启后真实对话"替代证明。

## 6. P1-08 基线核对

```
python compare_baseline.py manifest evidence/manifest_r8.txt        -> files=14121
python compare_baseline.py compare evidence/manifest_baseline_recomputed.txt evidence/manifest_r8.txt
   base_entries=14121 new_entries=14121 only_in_base=0 only_in_new=0 identical=14121
   RESULT: NO_DIFF
```

纳入规则：遍历 `D:\SoulMate` 全部文件，排除 `.git/build/.gradle/node_modules/.venv/venv/__pycache__/.pytest_cache/.idea/.dsh-office-*`。
清单为 `sha256  相对路径`，可独立复算。

手机运行态变更（不属于工作区文件）：`runtime/.env` 两个变量、后端进程重启、`.env.bak.20261121` 新增。

## 7. P1-06 四角色归因（部分完成）

保留 M2 为安全默认。M1（每 persona 独立 runtime/profile/provider client）未实测，给出方案与代价：
- 每 persona 一个 Shikigami 副本，`default_llm` 指向不同 preset，其 `base_url` 指向不同代理端口；
- Gateway 按 persona 路由到对应端口；后台调用因同实例发起，自然归到该 persona；
- 代价：4× 进程内存与端口；手机端可行性未验证。

未做的最小 PoC 需再跑一轮（两个副本、两个端口、验证端口 → persona 映射），本轮时间不足，如实标注。

## 8. 仍未覆盖

1. M1 的最小可运行 PoC（两个实例的端口归因）。
2. 真实 402（余额不足）与 429（限流）分支：需要外部条件，未构造。
3. 长回复下的客户端取消传播到上游（Mock 覆盖，真实未测）。
4. 手机端运行计量代理的性能与稳定性。
