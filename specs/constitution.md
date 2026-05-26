# luxict android openclaw Constitution

## 核心原则

### I. 基于上游的定制化
本项目是 openclaw/openclaw 的 fork，专注于 Android 端的定制优化。
原则：基于上游稳定版本二次开发，不做与上游核心架构背道而驰的修改。
说明：上游核心代码（TypeScript 服务端、Gateway 协议、扩展系统）保持干净，修改集中在 `apps/android/` 目录。

### II. Android 原生优先
所有功能修改以 Android 原生（Kotlin/Jetpack Compose）方式实现，不依赖第三方闭源 SDK。
原则：使用 Android SDK + Jetpack 官方库，避免引入非必要的第三方依赖。

### III. 一次只改一件事
原则：一个 commit 只改一个功能点。重构和功能修改不混在同一次提交中。
说明：复杂功能先写 spec → plan → tasks，按 task 逐个实现、逐个提交。

### IV. 真机验证
原则：所有 Android 端修改必须通过真机（ADB 连接）验证后才能提交。
说明：模拟器无法覆盖相机、MediaStore 等硬件相关功能的真实行为。

## 技术栈与约束

### Android 端（修改目标区域）

| 项目 | 实际值 |
|------|--------|
| **语言** | Kotlin 2.2.21 |
| **构建工具** | Gradle 8.x + Android Gradle Plugin 9.1.0 |
| **最小 SDK** | 31（Android 12） |
| **目标 SDK** | 36（Android 16） |
| **编译 SDK** | 36 |
| **应用 ID** | `ai.openclaw.app` |
| **版本** | 2026.3.24（versionCode 2026032400） |
| **目标 ABI** | armeabi-v7a, arm64-v8a, x86, x86_64 |
| **Java 兼容** | Java 17（source + target） |
| **代码检查** | ktlint 14.0.1，warningsAsErrors=true |

### 关键依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| Jetpack Compose BOM | 2026.02.00 | UI 框架 |
| Compose Material3 | — | Material Design 3 UI |
| CameraX（core/camera2/lifecycle/video） | 1.5.2 | 相机拍照、录像 |
| Kotlinx Coroutines | 1.10.2 | 异步编程 |
| Kotlinx Serialization | 1.10.0 | JSON 序列化 |
| OkHttp | 5.3.2 | 网络请求 |
| ExifInterface | 1.4.2 | 图片 EXIF 方向读取 |
| CommonMark | 0.27.1 | Markdown 渲染 |
| Bouncy Castle | 1.83 | 加密库 |
| dnsjava | 3.6.4 | DNS-SD 服务发现 |
| Google Play Code Scanner | 16.1.0 | 二维码扫描 |

### 构建命令

| 命令 | 说明 |
|------|------|
| `./gradlew :app:assemblePlayDebug` | Play flavor，debug 构建（常用） |
| `./gradlew :app:assembleThirdPartyDebug` | thirdParty flavor，debug 构建（含 SMS/通话记录权限） |
| `./gradlew :app:assemblePlayRelease` | Play flavor，release 构建（需签名配置） |
| `adb install -r app/build/outputs/apk/play/debug/openclaw-*-play-debug.apk` | 安装到真机 |

### 产品 Flavor

- **play**: Google Play 版本，不含 SMS/通话记录权限（`OPENCLAW_ENABLE_SMS=false`）
- **thirdParty**: 第三方商店版本，含 SMS/通话记录权限

### 服务端（保持上游，不做修改）

- **语言**: TypeScript（ESM）
- **运行时**: Node 22+
- **包管理**: pnpm（workspace monorepo）
- **测试**: Vitest

## 开发工作流

### 分支策略

| 分支 | 角色 | 来源 | 生命周期 |
|------|------|------|----------|
| **`main`** | 上游镜像 | fork 时从 upstream 创建，定期同步 | 持续，与上游完全一致 |
| **`develop`** | 主开发分支 | feature 合入 + main 同步 | 持续 |
| **`feature/<name>`** | 功能开发 | develop | 完成后合回 develop，保留 |
| **`release/v<版本>`** | 版本发布 | develop 拉出 | 发布后保留，在 release 分支上打 tag |
| **`hotfix/<描述>`** | 紧急修复 | main | 修复验证后合入 develop，保留 |

### 上游同步

- 定期从 `upstream/main` 合并到 `main`
- 合并冲突由 AI 处理，负责人审核把关
- 同步完成后将 `main` 合并到 `develop`，确保开发基线包含上游最新代码

### 打 Tag 规则

- **位置**: 每次发版在 release 分支上打 tag
- **格式**: `v<主>.<次>.<修订>`（语义化版本）
- **频次**: 每次发版必打 tag，不遗漏
- **版本规则**:
  - 新功能发布 → 次版本 +1（如 `v1.0.0` → `v1.1.0`）
  - 重大 bug 修复 / 安全修复 → 修订号 +1（如 `v1.1.0` → `v1.1.1`）
  - 架构重构 / 破坏性变更 → 主版本 +1（如 `v1.1.0` → `v2.0.0`）
  - 小 bug 修复积累到下一次正式发布

### SDD 流程

1. **Spec-Driven Development**: 每个功能先写 spec → plan → tasks，逐份对齐后再动手
2. **Commit 规范**:
   - 格式: `<type>(<scope>): <描述>`
   - 类型: `feat` / `fix` / `docs` / `refactor` / `chore` / `test`
   - 范围: `android`（Android 端修改）
   - 末尾追加 trailer: `AI-assisted: <工具> (<模型>)`
   - 示例: `feat(android): add saveToGallery support to camera.snap`
3. **提交前确认**: 每次 git commit 前先向项目负责人展示最终内容，确认后方可提交
4. **构建验证**: `cd apps/android && ./gradlew :app:assemblePlayDebug` 编译通过后再提交

### 代码审查（Code Review）

每个 feature 分支合入 develop 前，必须经过以下审查流程：

| 轮次 | 审查者 | 职责 |
|------|--------|------|
| 第1轮 | AI review（开发者自选工具） | 代码规范、逻辑错误、安全风险 |
| 第2轮 | AI review（可更换另一款工具） | 覆盖率检查、边界条件、性能问题 |
| 第3轮 | 人工 review（项目负责人） | 最终把关，确认全部问题已处理 |

**规则：**
- 每轮 review 产出的评审意见存入 `specs/<feature>/review/` 目录
- 每轮 review 提出的每个问题必须处理：修复或确认非问题并说明原因
- 问题处理记录要清晰可回溯
- 3 轮全部通过后，feature 方可合入 develop

## 治理规则

- Constitution 修改需经项目负责人（huangfusheng、sunxuewen）确认
- 上游（openclaw/openclaw）的架构决策优先于本项目内的偏好
- 新增第三方依赖需评估许可协议和安全性

## 变更记录

| 版本 | 日期 | 变更内容 | 确认人 |
|------|------|---------|--------|
| 1.0.0 | 2026-05-26 | 初始版本 | huangfusheng, sunxuewen |

