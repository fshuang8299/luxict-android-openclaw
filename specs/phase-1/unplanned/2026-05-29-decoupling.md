# Phase 1 — 计划外改动补办记录

**日期**：2026-05-29
**触发原因**：在 Phase 1 实施期间，working tree 累积了若干未经 spec → plan → tasks 流程的改动。依据 `specs/constitution.md` 原则 V 第 4 条「可追溯性」，统一在此补办登记。
**已完成解耦**：所有改动已通过 luxict product flavor + BuildConfig 重新组织，上游 `src/**` 与 `app/src/main/` 中的硬编码值均已恢复或参数化。

## 改动清单

| # | 改动 | 涉及文件 | 解耦方式 | 上游回归路径 |
|---|------|---------|---------|------------|
| U1 | **chat.send 超时配置化** | `app/src/main/.../chat/ChatController.kt`（30_000 → `BuildConfig.CHAT_TIMEOUT_MS`） | 上游文件加 BuildConfig 入口；play/thirdParty=30_000，luxict=120_000 | 向上游提"hook injection PR"：把硬编码超时改为 BuildConfig 入口（上游会接，对上游 0 影响） |
| U2 | **chat.send 语音流超时配置化** | `app/src/main/.../NodeRuntime.kt`（30_000 → `BuildConfig.VOICE_TIMEOUT_MS`） | 同上；play/thirdParty=30_000，luxict=45_000 | 同 U1 |
| U3 | **versionCode/Name 自动生成（仅 luxict）** | `app/build.gradle.kts` | defaultConfig 维持硬编码；luxict flavor 覆盖为 UTC 日期生成 | 仅 luxict 渠道使用，不影响上游 |
| U4 | **去除 `-dev` 后缀（仅 luxict）** | `app/src/main/.../node/ConnectionManager.kt`（加 `BuildConfig.STRIP_DEV_SUFFIX` 开关） | 上游文件加 BuildConfig 入口；play/thirdParty 保留上游 `-dev` 行为 | 向上游提 hook injection PR 加 BuildConfig 开关 |
| U5 | **i18n UI 字符串（英文 + 中文）** | `app/src/luxict/res/values/strings.xml`（149 条新增）<br>`app/src/luxict/res/values-zh/strings.xml`（中文翻译） | 完全放 luxict flavor，main/ 仅保留 `app_name` | 若上游有引用 R.string.\* 的 UI 代码，再决定是否搬回 main |

## 已回退的违宪改动

| 改动 | 原因 |
|------|------|
| `src/gateway/protocol/schema/protocol-schemas.ts`：PROTOCOL_VERSION 3 → 4 | 违反原则 V「零修改区」；该 bump 不含任何 schema 实际变化；保留会与上游官方服务端握手失败 |
| `app/src/main/.../GatewayProtocol.kt`：GATEWAY_PROTOCOL_VERSION 3 → 4 | 与上述同步回退 |

## 验证

- [ ] `./gradlew :app:assembleLuxictDebug` 通过
- [ ] `./gradlew :app:assemblePlayDebug` 通过（验证未破坏上游契约）
- [ ] `./gradlew :app:assembleThirdPartyDebug` 通过
- [ ] luxict APK 真机验证：chat.send 超时 120s、versionName 为当日 UTC 日期、无 `-dev` 后缀、中文系统下显示中文

## 后续行动

1. **提 2 个 hook injection PR 给上游**：
   - PR A：`ChatController.kt`、`NodeRuntime.kt` 超时改为 BuildConfig 入口
   - PR B：`ConnectionManager.kt` 加 `STRIP_DEV_SUFFIX` 开关
2. **上游合并后**，移除本目录登记 + 移除 constitution「上游修改例外清单」对应行
3. **未上游接受期间**，每次 `git pull upstream/main` 若这 3 个文件冲突，使用 `git rerere` 复用历史解决方案
