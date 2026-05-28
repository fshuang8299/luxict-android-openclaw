# luxict flavor — fork 定制发行渠道

本目录是 luxict 发行渠道的源码集（Android product flavor source set）。

## 用途

依据 `specs/constitution.md` 原则 V「上游解耦优先」：
所有 fork 专属的代码、资源、配置必须放在本目录，**不得污染** `app/src/main/`。

## 目录结构

```
luxict/
├── java/ai/openclaw/app/luxict/   ← fork 专属 Kotlin 代码
├── res/values/                    ← 覆盖/补充上游字符串、品牌资源
├── res/values-zh/                 ← 中文翻译（fork 专属 i18n）
└── README.md                      ← 本文件
```

## 构建命令

```bash
./gradlew :app:assembleLuxictDebug    # 构建 luxict-debug
./gradlew :app:assembleLuxictRelease  # 构建 luxict-release
```

## BuildConfig 区分

代码中通过 `ai.openclaw.app.BuildConfig.IS_LUXICT_BUILD` 区分当前构建是否为 luxict 渠道。
仅在以下情况允许使用此判断：
- 必须改上游 `src/main/` 文件时，用于注入 luxict 专属默认值
- 上游代码已有 BuildConfig 入口、且改入口比搬代码更合理时

否则一律走 Java/资源覆盖机制（同名文件优先级：`luxict/` > `main/`）。

## 与上游同步

`git pull upstream/main` 时，`luxict/` 目录天然不冲突。
若 `main/` 发生冲突，记入 `specs/<phase>/unplanned/`。
