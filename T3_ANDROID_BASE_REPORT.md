# Task 3 Android 基础工程 完成报告

> 执行日期：2026-11-17
> 依据：《SoulMate DSH 问题汇总与后续整改任务书》§4 允许范围
> 结论：**T3 基础工程完成并通过真机验证**；聊天闭环（T4）待开展

---

## 0. 摘要

| 项 | 内容 |
|---|---|
| **任务** | T3 · Android 基础工程 |
| **技术栈** | Kotlin 2.0.21 + Compose (BOM 2025.01) + Material 3 + Navigation Compose |
| **构建** | AGP 8.9.2 / Gradle 8.13 / JDK 17 / compileSdk 36 / minSdk 26 |
| **产物** | `app-debug.apk` **18.23 MB** |
| **测试** | **40 项单元/UI 测试全通过**（26 网络+映射 / 14 Compose UI） |
| **真机** | vivo V2536A / Android 16 (API 36) / arm64-v8a **安装并验证通过** |
| **发现的真 bug** | 2 个（都是真机测试才暴露的） |

---

## 1. 工程结构

```
android/
├── settings.gradle.kts          # 阿里云镜像优先（实测 2.5 MB/s），官方兜底
├── build.gradle.kts
├── gradle.properties
├── gradle/libs.versions.toml    # 版本目录
└── app/
    ├── build.gradle.kts
    └── src/
        ├── main/java/com/soulmate/app/
        │   ├── SoulMateApplication.kt   # 持有 AppContainer
        │   ├── AppContainer.kt          # 极简 DI（不引入 Hilt）
        │   ├── MainActivity.kt
        │   ├── core/
        │   │   ├── network/             # GatewayApi / GatewayClient / SseChatClient
        │   │   │                        # ApiResult / ChatStreamEvent / SafeApiCall
        │   │   ├── security/TokenStore.kt      # Android Keystore 加密
        │   │   └── preferences/SettingsRepository.kt  # DataStore
        │   ├── data/
        │   │   ├── model/GatewayDtos.kt  # Moshi DTO（精确映射 snake_case）
        │   │   ├── local/SoulMateDatabase.kt  # Room（仅 UI 缓存）
        │   │   └── repository/            # HealthRepository / PersonaRepository
        │   ├── feature/
        │   │   ├── health/       # 连接状态页
        │   │   ├── persona/      # 角色列表（只读）
        │   │   └── settings/     # 令牌 / 地址 / 主题
        │   └── ui/
        │       ├── theme/        # 色彩 / 排版 / 主题
        │       └── SoulMateApp.kt  # 导航宿主
        ├── test/java/...          # 40 项测试
        └── androidTest/java/...   # 预留（真机测试用 Robolectric 替代）
```

---

## 2. 网络层（任务书 §4 重点：超时分级）

### 2.1 三档超时，各司其职

| 用途 | connect | read | write | 说明 |
|---|---|---|---|---|
| 普通 JSON（health/personas） | 5s | **30s** | 10s | 后端本地，30s 足够 |
| SSE 聊天 | 5s | **120s** | 10s | 8 个心跳周期（网关 15s 一次） |

**为什么不设"总时限"**：长回复可能持续 60s+，硬性总时限会误杀正常请求。
真正的保护是心跳 + 读超时。

**为什么 SSE 用 120s**：AGENTS.md 坑 1 明确要求 ≥120s（上游首 token 最慢观测 43s）。

**为什么 SSE 不复用 jsonClient**：
- readTimeout 差异大（30s vs 120s）
- SSE 需 `retryOnConnectionFailure=false`——**防止 OkHttp 静默重发导致重复生成**

### 2.2 错误分类（UI 可解释的关键）

```kotlin
sealed class ApiError {
    ConnectTimeout   // 后端没起 / IP 不通 → "检查网关地址"
    ReadTimeout      // 后端过慢 / 上游卡住 → "稍后重试"
    Unauthorized     // 令牌缺失/错误 → "去设置填令牌"
    UpstreamUnavailable  // 网关在线但引擎挂了 → "检查上游"
    GatewayError     // 结构化错误（含 code）
    ParseError       // 契约不一致
    Network / Unknown
}
```

**实测证据**（真机断开后端时）：
```
连不上 Gateway
· 后端可能没启动。若后端跑在电脑上，请检查网关地址（设置页）与同一局域网。
· 详情：Failed to connect to /127.0.0.1:7790
[重试]
```

### 2.3 SSE 客户端

- 用 `okhttp-sse` 的 `EventSource`
- 事件解析成密封类 `ChatStreamEvent`（6 种）
- **未知事件类型被忽略**（向前兼容），不崩溃
- `callbackFlow` 封装，取消即断开，不泄漏
- 心跳 `: ping` 是注释帧，不进 `onEvent` 但保持连接

> T3 只建基础设施 + 单测；聊天 UI 属 T4。

---

## 3. 数据层（严格不越界）

### 3.1 Room —— 只做 UI 缓存

```kotlin
@Entity(tableName = "cached_personas")
data class PersonaEntity(...)
```

**可验证标准**（写在源码注释里）：删库重开 App，所有内容能从 Gateway 重建。

### 3.2 令牌 —— Android Keystore 加密

- AES-256/GCM 密钥由 Android Keystore 生成（不出系统密钥库）
- 令牌加密后 Base64 存 SharedPreferences
- **不参与备份**（`allowBackup=false` + `dataExtractionRules` 排除）

**真机验证**（`adb run-as` 直接读磁盘）：

```xml
<string name="gateway_token_b64">ZaAEDprayOFVaguSYwImGYs5RhpvqXWCFonQm8vqUXHZHvwvH8pIfF6Kjv9yCdmrH4mh9UrkVXTJ+kTSvPtxNQnKpg8B+Yo=</string>
```

**磁盘上找不到明文令牌** ✅（脚本用真实 token 做了子串比对）

### 3.3 网络安全配置

| 构建类型 | 明文 HTTP | 适用 |
|---|---|---|
| main（release） | **禁止**，仅放行 `127.0.0.1` / `localhost` / `10.0.2.2` | 生产 |
| debug | 允许（便于连电脑局域网 IP 调试） | 开发 |

---

## 4. 功能实现

### 4.1 连接状态页

展示：整体状态 / 上游（版本、延迟、告警）/ LLM（**三态**）/ Gateway（版本、运行时长）

**LLM 三态诚实显示**（关键设计）：

| available | 显示 | 判定依据文案 |
|---|---|---|
| `true` | 可用（绿） | 真实请求观测 |
| `false` | 不可用（红） | — |
| `null` | **未知（灰）** | 无证据（尚未发起过真实请求） |

真机实测：`status=OK, 上游=可达, LLM=未知` —— 没发过真实请求就不假装可用。

### 4.2 角色列表（只读）

4 个角色、17 种情绪标签、"当前"标记。
带**数据来源标注**：网络成功 → 无横幅；只有缓存 → 明确提示"不是最新数据"。

### 4.3 设置页

令牌（掩码回显 `••••ug8M`）/ 连接测试 / Gateway 地址 / 主题 / 自动刷新。

---

## 5. 测试

### 5.1 单元 + UI 测试：40 项全通过

| 测试类 | 数量 | 覆盖点 |
|---|---|---|
| `ChatEventParserTest` | 13 | 6 种事件 + **未知类型/非法 JSON 不崩溃** |
| `GatewayClientTest` | 5 | baseUrl 归一化 + **超时分级不变量** |
| `PersonaMappingTest` | 8 | 相对路径→绝对 URL、无头像不编造、标签编解码 |
| `HealthContentTest` | 14 | **LLM unknown 不显示"可用"**、错误分类文案、令牌缺失引导 |

Compose UI 测试用 **Robolectric 在 JVM 上跑**，不需要真机，CI 友好。

### 5.2 真机验证（vivo V2536A / Android 16）

用 `adb reverse tcp:7790 tcp:7790` 把手机 `127.0.0.1:7790` 转发到电脑网关。

| 场景 | 结果 |
|---|---|
| 首屏（无令牌） | 显示"尚未配置访问令牌"+ 引导 ✅ |
| 输入令牌（43 字符）→ 保存 | 回显 `••••ug8M` ✅ |
| 测试连接 | `✅ 连接成功：status=OK, 上游=可达, LLM=未知` ✅ |
| 连接页完整数据 | 上游 0.12.0 / 延迟 27ms / LLM 未知+原因 ✅ |
| 角色列表 | 4 个角色、当前标记、17 种情绪 ✅ |
| **断开后端后重启** | 连接页"连不上 Gateway"+ 定向引导；角色页展示缓存并标注 ✅ |
| 令牌落盘 | 密文，无明文 ✅ |

---

## 6. 真机测试发现并修复的 2 个真 bug

这两个都是 **UI 单测测不出来**、只有真实导航/真实刷新才暴露的。

### Bug 1：配置令牌后连接页仍显示"未配置"

**现象**：设置页保存令牌 → 切回连接页 → 仍显示"尚未配置访问令牌"。

**根因**：Navigation Compose 会缓存 ViewModel（`saveState`/`restoreState`），
`HealthViewModel.init` 只在首次创建时跑一次，`tokenMissing` 一直停在旧值。

**修复**：加 `onResumed()`，页面每次进入时同步令牌状态；`HealthScreen` 用 `LaunchedEffect(Unit)` 调用。

### Bug 2：刷新成功后仍显示"这是缓存"

**现象**：点刷新拿到最新数据，但"当前显示的是本地缓存"横幅不消失。

**根因**：`refresh()` 成功后 `replaceAll()` 写 Room → Room 的 Flow 触发 →
缓存观察器把 `showingCache` 又设回 `true`。**观察器和网络结果互相覆盖**。

**修复**：把 `showingCache: Boolean` 换成显式的 `dataSource: NONE/NETWORK/CACHE`；
缓存观察器只在 `dataSource == NONE`（冷启动、还没有任何数据）时填充，
一旦拿到网络结果就不再让缓存覆盖。

---

## 7. 已知问题与技术债

| # | 问题 | 影响 | 计划 |
|---|---|---|---|
| 1 | **Gateway 字段命名不统一** | `/v1/personas`、`/v1/health` 是 snake_case；`/v1/chat/stream` 是 camelCase | Android 端用 Moshi 精确映射兜住；**建议 T4 前统一为 camelCase** |
| 2 | **头像未渲染** | T3 用首字母圆形占位，未引入图片库 | T4 引入 Coil（约 300KB）；`avatar_url` 已是绝对 URL，接入成本低 |
| 3 | 聊天 UI 未做 | SSE 客户端已就绪，但无聊天页面 | 属 T4 范围 |
| 4 | `fallbackToDestructiveMigration` | 未来加表会清空缓存 | 缓存可从 Gateway 重建，可接受；加正式表时再写迁移 |
| 5 | 未写 `androidTest`（真机仪器测试） | 真机验证靠 adb 手动操作 | Compose UI 已用 Robolectric 覆盖；仪器测试留到 T4 |
| 6 | 网关地址改动需重启 App | `BuildConfig.GATEWAY_BASE_URL` 是编译期常量 | T4 改为运行时从 DataStore 读取 |

---

## 8. 对照任务书 §4 允许范围

| 允许项 | 状态 |
|---|---|
| Kotlin + Compose + Material 3 工程骨架、导航、主题、基础页面 | ✅ |
| Retrofit/OkHttp 网络层、health 连接状态、DataStore 偏好、基础 Room UI 缓存 | ✅ |
| 角色列表/角色信息只读展示骨架（字段以 Gateway 实际响应为准） | ✅ |
| 网络层单元测试、UI 基础测试、本地构建验证 | ✅ 40 项 |

| 完整聊天闭环前置条件 | 状态 |
|---|---|
| P0 幂等持久化 + 重启恢复 | ✅ 已于 T2-H 完成 |
| 取消/断流可解释、客户端不盲目重发 | ✅ 已于 T2-H 完成 |
| SSE 解析/超时/重连/错误展示经**真机**测试 | ⬜ **待 T4**（网络层已就绪，聊天 UI 未做） |
| 头像代理已验证 / 客户端可处理 `avatar_url=null` | ✅ 代理已验证；客户端 `absoluteAvatarUrl(null) == null` 有测试 |
| Room 只作 UI 缓存 | ✅ 源码注释 + 离线验证 |

---

## 9. 下一阶段建议

**T4 聊天闭环**可以直接开始，前置条件基本齐备。开工前建议先做两件小事：

1. **统一 Gateway 字段命名为 camelCase**（趁客户端刚起来，改动成本最低）
2. 引入 Coil 渲染头像

然后按 `DSH_TASKS.md` 的 T4 清单推进：SSE 聊天页、历史恢复、
`clientMessageId` 幂等、断线重连先查 `/v1/chat/status`。

---

> **报告结束**
