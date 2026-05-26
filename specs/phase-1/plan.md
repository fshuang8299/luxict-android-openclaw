# Implementation Plan: Phase 1 — Phone System Control

**Branch**: `develop` | **Date**: 2026-05-26 | **Spec**: `specs/phase-1/overview.md`

## 1. Summary

Phase 1 在保持上游 Android 端架构不变的前提下，完成两项工作：
1. **camera.snap** 新增 saveToGallery 参数
2. **device.* 命令空间** 扩展 10 条设备控制命令

不创建新文件结构，完全沿用上游的三层架构模式：Protocol Constants → Registry → Dispatcher → Handler。

## 2. 上游架构分析（遵循）

### 2.1 命令定义层 — Protocol Constants

文件：`protocol/OpenClawProtocolConstants.kt`

每条命令是一个枚举值，命名空间前缀统一管理：

```kotlin
enum class OpenClawDeviceCommand(val rawValue: String) {
  Status("device.status"),
  Info("device.info"),
  Permissions("device.permissions"),
  Health("device.health"),
  // ↑ 上游已有 ↑
  // ↓ Phase 1 新增 ↓
  ScreenBrightness("device.screen.brightness"),
  ScreenTimeout("device.screen.timeout"),
  AudioVolume("device.audio.volume"),
  AudioMode("device.audio.mode"),
  Wifi("device.wifi"),
  Bluetooth("device.bluetooth"),
  AirplaneMode("device.airplane"),
  PowerMode("device.power.mode"),
  PowerPolicy("device.power.policy"),
  PowerCpu("device.power.cpu"),
  ;

  companion object {
    const val NamespacePrefix: String = "device."
  }
}
```

### 2.2 命令注册层 — Registry

文件：`node/InvokeCommandRegistry.kt`

新增 InvokeCommandSpec 条目，availability 统一为 `Always`：

```kotlin
InvokeCommandSpec(name = OpenClawDeviceCommand.ScreenBrightness.rawValue),
InvokeCommandSpec(name = OpenClawDeviceCommand.ScreenTimeout.rawValue),
// ... 依次类推
```

### 2.3 命令分发层 — Dispatcher

文件：`node/InvokeDispatcher.kt`

在 `device.*` 分支下追加新路由：

```kotlin
// Device commands
OpenClawDeviceCommand.Status.rawValue -> deviceHandler.handleDeviceStatus(paramsJson)
OpenClawDeviceCommand.Info.rawValue -> deviceHandler.handleDeviceInfo(paramsJson)
// ↑ 上游已有 ↑
// ↓ Phase 1 新增 ↓
OpenClawDeviceCommand.ScreenBrightness.rawValue -> deviceHandler.handleScreenBrightness(paramsJson)
OpenClawDeviceCommand.ScreenTimeout.rawValue -> deviceHandler.handleScreenTimeout(paramsJson)
// ...
```

### 2.4 业务逻辑层 — Handler

#### 方案 A：扩展 DeviceHandler（推荐）

将所有设备控制命令放在同一个 `DeviceHandler` 中，保持与上游 `device.*` 命名空间的一致性。新增方法以 `handle*` 命名：

```kotlin
class DeviceHandler(
  private val appContext: Context,
  // ... 上游注入不变
) {
  // --- 上游已有（只读）---
  fun handleDeviceStatus(): GatewaySession.InvokeResult { ... }
  fun handleDeviceInfo(): GatewaySession.InvokeResult { ... }

  // --- Phase 1 新增（读写控制）---
  fun handleScreenBrightness(paramsJson: String?): GatewaySession.InvokeResult { ... }
  fun handleScreenTimeout(paramsJson: String?): GatewaySession.InvokeResult { ... }
  fun handleAudioVolume(paramsJson: String?): GatewaySession.InvokeResult { ... }
  fun handleAudioMode(paramsJson: String?): GatewaySession.InvokeResult { ... }
  fun handleWifi(paramsJson: String?): GatewaySession.InvokeResult { ... }
  fun handleBluetooth(paramsJson: String?): GatewaySession.InvokeResult { ... }
  fun handleAirplaneMode(paramsJson: String?): GatewaySession.InvokeResult { ... }
  fun handlePowerMode(paramsJson: String?): GatewaySession.InvokeResult { ... }
  fun handlePowerPolicy(paramsJson: String?): GatewaySession.InvokeResult { ... }
  fun handlePowerCpu(paramsJson: String?): GatewaySession.InvokeResult { ... }
}
```

**理由**：上游 `DeviceHandler` 已经持有 `appContext`，扩展它不需要改构造函数、注入、测试框架。所有控制命令同属 `device.*` 命名空间，放在同一个 Handler 里最自然。

#### 方案 B：拆分 ControlHandler（备选，暂不使用）

如果未来控制命令膨胀到 20+ 条，可以考虑拆出独立的 `DeviceControlHandler`。Phase 1 阶段 9 条命令，不值的引入额外抽象层。

## 3. 命令设计规范

每条控制命令遵循统一的请求/响应格式：

### 读取命令

```
请求: device.screen.brightness（无 params）
响应: {"brightness": 128, "max": 255}
```

### 设置命令

```
请求: device.screen.brightness {"brightness": 200}
响应: {"brightness": 200, "previous": 128}
```

### 切换命令（开关类）

```
请求: device.wifi {"enabled": true}
响应: {"enabled": true, "previous": false}
```

### 错误响应（统一格式）

```json
{"error": "INVALID_PARAM: brightness must be 0-255"}
```

## 4. 权限策略

| 功能 | 所需权限 | 处理方式 |
|------|---------|---------|
| 屏幕亮度 | `WRITE_SETTINGS` | 需用户手动授权（系统设置弹窗） |
| 屏幕超时 | `WRITE_SETTINGS` | 同上 |
| 音量控制 | 无（系统 API） | 无需额外权限 |
| 响铃模式 | 无 | `AudioManager` API |
| WiFi 开关 | `CHANGE_WIFI_STATE` | Manifest 声明，运行时授权 |
| 蓝牙开关 | `BLUETOOTH_CONNECT` / `BLUETOOTH_ADMIN` | API 31+ 需要运行时授权 |
| 飞行模式 | `WRITE_SETTINGS` | 需手动授权 |
| 性能/功耗 | 无 | 系统 API（PowerManager） |

## 5. 开发顺序（依赖关系）

```
Phase 1 整体  ─┬─ saveToGallery（camera.snap）──── 独立，可最先做
               │
               ├─ 屏幕控制  ─┬─ 亮度调节 ─── 独立
               │             └─ 屏幕超时 ─── 独立
               │
               ├─ 声音与模式 ─┬─ 音量控制 ─── 独立
               │              └─ 响铃模式 ─── 独立
               │
               ├─ 网络控制  ─┬─ WiFi ──────── 独立
               │             ├─ 蓝牙 ──────── 独立
               │             └─ 飞行模式 ──── 独立
               │
               └─ 性能与功耗 ─┬─ 性能模式 ──── 独立
                              ├─ 功耗调节 ──── 独立
                              └─ CPU 调度 ──── 独立
```

所有功能**无依赖关系**，可按任意顺序独立开发。建议从 `saveToGallery` 开始，先验证整个开发+审查流程。

## 6. Constitution Check

| 宪章要求 | 符合情况 |
|---------|---------|
| 基于上游定制 | ✅ 完全沿用上游三层架构模式 |
| Android 原生 | ✅ 使用标准 Android API（Settings.System、AudioManager、WifiManager 等） |
| 一次只改一事 | ✅ 每个功能单独 commit |
| 真机验证 | ✅ 每个功能需真机验证 |
