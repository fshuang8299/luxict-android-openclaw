# Technical Standards

## 1. 日志规范

### 1.1 Tag 命名

统一使用类名字符串常量，不使用 `TAG` 变量：

```kotlin
// ✅ 正确
android.util.Log.w("CameraCaptureManager", "snap: taking photo")

// ❌ 不推荐
private const val TAG = "CameraCaptureManager"
Log.w(TAG, "snap: taking photo")
```

理由：上游代码统一使用字符串字面量（CameraCaptureManager.kt:180-267），保持一致。

### 1.2 日志级别

| 级别 | 场景 | 格式 |
|------|------|------|
| `Log.v()` | 详细调试（Release 可裁减） | `"method: step detail"` |
| `Log.d()` | 开发调试（BuildConfig.DEBUG 受控） | `"method: debug info"` |
| `Log.i()` | 关键事件（功能启用/完成） | `"method: action completed: result"` |
| `Log.w()` | 预期内的异常/回退（不阻塞流程） | `"method: issue description"` |
| `Log.e()` | 非预期异常（需人工关注） | `"method: error description", exception` |

### 1.3 消息格式

```
"<方法名或阶段>: <具体描述>"
```

示例：

```kotlin
Log.w("DeviceHandler", "handleWifi: user declined permission, returning current state")
Log.e("CameraCaptureManager", "saveToGallery: MediaStore.insert failed", exception)
Log.i("DeviceHandler", "handlePowerMode: switched to power_save")
```

## 2. 错误处理规范

### 2.1 错误码格式

统一使用 `UPPER_SNAKE_CASE`，格式 `"前缀_具体错误"`：

```kotlin
return GatewaySession.InvokeResult.error(
    code = "WRITE_SETTINGS_UNAVAILABLE",
    message = "WRITE_SETTINGS_UNAVAILABLE: not granted",
)
```

错误码前缀需能快速定位来源（如 `CAMERA_*`、`DEVICE_*`、`LOCATION_*` 等）。

### 2.2 错误信息规则

- message 必须以 **error code 开头 + 冒号 + 空格 + 人类可读描述**
- 描述要告知调用方**发生了什么 + 如何修复**（如果有操作空间）

```kotlin
// ✅ 正确: 告知原因 + 修复方式
code = "WRITE_SETTINGS_UNAVAILABLE"
message = "WRITE_SETTINGS_UNAVAILABLE: screen brightness control requires WRITE_SETTINGS in Settings > Special access"

// ❌ 错误: 无修复指引
code = "FAILED"
message = "FAILED"
```

### 2.3 禁止静默 catch

上游已有反例（InvokeDispatcher.kt:215 `catch (_: Throwable) { }`），新增代码中禁止：

```kotlin
// ❌ 禁止
try {
    operation()
} catch (_: Exception) { }

// ✅ 正确
try {
    operation()
} catch (e: Exception) {
    Log.e("DeviceHandler", "handleXxx: operation failed", e)
    throw e  // 或返回 InvokeResult.error()
}
```

### 2.4 Fail-Open 原则

不影响主流程的次要操作失败时，记录日志但不阻塞主流程返回：

```kotlin
if (saveToGallery) {
    try {
        saveToGallery(context, bytes, fileName)
    } catch (e: Exception) {
        Log.e(TAG, "saveToGallery: failed", e)
        // 不 rethrow，不阻塞 base64 返回
    }
}
```

## 3. 测试策略

### 3.1 测试框架（沿用上游）

| 框架 | 版本 | 用途 |
|------|------|------|
| JUnit 4 | 4.13.2 | 测试运行器 |
| Kotest | 6.1.3 | 断言（kotest-assertions-core） |
| Robolectric | 4.16.1 | Android 环境模拟 |
| MockWebServer | 5.3.2 | HTTP 模拟 |
| kotlinx-coroutines-test | 1.10.2 | 协程测试 |

### 3.2 测试层级

| 层级 | 覆盖范围 | 工具 | 新增功能必须 |
|------|---------|------|-------------|
| **Unit** | Handler 方法级（不依赖 Android 环境） | JUnit + Kotest | ✅ 必写 |
| **Unit** | Handler 方法级（依赖 Android 组件） | Robolectric | ✅ 必写 |
| **Integration** | Gateway 交互 + InvokeResult 格式 | MockWebServer | 推荐 |
| **E2E** | 真机 ADB 执行 | 手动 | ✅ 必做 |

### 3.3 测试文件结构

```
app/src/test/java/ai/openclaw/app/node/
├── CameraCaptureManagerTest.kt   ← 新增
├── DeviceHandlerTest.kt           ← 新增
├── InvokeCommandRegistryTest.kt   ← 已有
└── ...
```

### 3.4 测试命名规范

```kotlin
// 类名: <被测类>Test
class DeviceHandlerTest {

    // 方法名: `<方法>__<场景>__<预期结果>`
    fun handleScreenBrightness__read__returnsCurrentValue() { }
    fun handleScreenBrightness__setValid__returnsNewValue() { }
    fun handleScreenBrightness__setInvalid__returnsError() { }
}
```

### 3.5 新增功能的测试要求

每个新的 device.* 命令至少包含：

1. **读取测试**: 验证返回 JSON 格式和字段
2. **设置测试**: 验证设置值后返回值正确
3. **错误测试**: 验证无效参数返回正确 error code
4. **权限测试**: 验证无权限时返回正确 error code

## 变更记录

| 版本 | 日期 | 变更内容 | 确认人 |
|------|------|---------|--------|
| 1.0.0 | 2026-05-26 | 初始版本 | huangfusheng, sunxuewen |
