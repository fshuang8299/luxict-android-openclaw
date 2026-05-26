# Implementation Plan: saveToGallery for camera.snap

**Branch**: `feature/save-to-gallery` | **Date**: 2026-05-26 | **Spec**: `specs/phase-1/overview.md`

## 1. Summary

在 CameraCaptureManager.snap() 中新增 `saveToGallery` 布尔参数。当为 true 时，在 JPEG 压缩完成后，将原始字节通过 `MediaStore.Images.Media` API 写入 `Pictures/OpenClaw/`。保存失败不阻塞 base64 返回，仅记录日志。

## 2. Technical Context

| 项目 | 内容 |
|------|------|
| **目标文件** | `apps/android/app/src/main/java/ai/openclaw/app/node/CameraCaptureManager.kt` |
| **修改范围** | snap() 方法（行 97-166）+ 新增 saveToGallery() 方法 |
| **所需 API** | `MediaStore.Images.Media.EXTERNAL_CONTENT_URI`（Android 10+, API 29+） |
| **运行时权限** | 无（minSdk=31，MediaStore 无需额外权限） |
| **线程要求** | MediaStore 写入必须在 `Dispatchers.IO` |
| **编译 SDK** | compileSdk=36，所有 API 可用 |

## 3. Implementation Design

### 3.1 Data Flow (变更后)

```
snap(params)
  ↓
parseJsonParamsObject(params)
  → val saveToGallery = parseSaveToGallery(params) ?: false    ← 新增
  ↓
takeJpegWithExif() → (bytes, orientation, tempFile)            ← 改为保留 tempFile
  ↓
decode → rotate → scale → compress → base64
  ↓
if (saveToGallery) {
    withContext(Dispatchers.IO) {                               ← IO 线程
        saveToGallery(context, result.bytes, fileName)
    }
}
  ↓
return Payload(base64)                                          ← 保存失败不影响返回
```

### 3.2 saveToGallery() 方法实现

```kotlin
private suspend fun saveToGallery(
    context: Context,
    jpegBytes: ByteArray,
    fileName: String,
): Boolean = withContext(Dispatchers.IO) {
    val values = ContentValues().apply {
        put(MediaStore.Images.Media.DISPLAY_NAME, fileName)
        put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
        put(MediaStore.Images.Media.RELATIVE_PATH, "${Environment.DIRECTORY_PICTURES}/OpenClaw")
        put(MediaStore.Images.Media.IS_PENDING, 1)  // 防止写入过程中出现在相册
    }

    try {
        val uri = context.contentResolver.insert(
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            values,
        ) ?: return@withContext false.also {
            Log.e(TAG, "saveToGallery: insert returned null for $fileName")
        }

        context.contentResolver.openOutputStream(uri)?.use { outputStream ->
            outputStream.write(jpegBytes)
        } ?: return@withContext false.also {
            Log.e(TAG, "saveToGallery: openOutputStream null for $fileName")
            // 清理已插入的空记录
            context.contentResolver.delete(uri, null, null)
        }

        // 清除 IS_PENDING 标记，让照片出现在相册
        val updateValues = ContentValues().apply {
            put(MediaStore.Images.Media.IS_PENDING, 0)
        }
        context.contentResolver.update(uri, updateValues, null, null)

        Log.i(TAG, "saveToGallery: saved $fileName")
        true
    } catch (e: Exception) {
        Log.e(TAG, "saveToGallery: failed to save $fileName", e)
        false
    }
}
```

### 3.3 snap() 方法需要修改的部分

```kotlin
suspend fun snap(paramsJson: String?): Payload =
    withContext(Dispatchers.Main) {
        // ... 上游已有代码（权限检查、相机初始化）...

        val saveToGallery = parseSaveToGallery(params) ?: false  // ← 新增

        val provider = context.cameraProvider()
        // ... 拍照、旋转、压缩、返回 base64 ...
        val (bytes, orientation, tempFilePath) = capture.takeJpegWithExif(context.mainExecutor())  // ← 改为 Triple

        // ... 原有的 decode / rotate / scale / compress 逻辑 ...

        // ← 新增：保存到相册
        if (saveToGallery) {
            val fileName = "OpenClaw_${timestamp()}.jpg"
            val saved = withContext(Dispatchers.IO) {
                saveToGallery(context, result.bytes, fileName)
            }
            if (!saved) {
                Log.w(TAG, "snap: saveToGallery failed for $fileName")
            }
        }

        Payload(...)
    }
```

### 3.4 takeJpegWithExif() 修改

原函数在读完文件后删除临时文件（行 420: `file.delete()`），需要改为**不删除，返回文件路径**，让 snap() 自行决定何时删除。

```kotlin
private suspend fun ImageCapture.takeJpegWithExif(executor: Executor): Triple<ByteArray, Int, File> =
    suspendCancellableCoroutine { cont ->
        val file = File.createTempFile("openclaw-snap-", ".jpg")
        val options = ImageCapture.OutputFileOptions.Builder(file).build()
        takePicture(options, executor, object : ImageCapture.OnImageSavedCallback {
            override fun onError(exception: ImageCaptureException) {
                file.delete()
                cont.resumeWithException(exception)
            }
            override fun onImageSaved(outputFileResults: ImageCapture.OutputFileResults) {
                try {
                    val exif = ExifInterface(file.absolutePath)
                    val orientation = exif.getAttributeInt(...)
                    val bytes = file.readBytes()
                    // 不删除文件，让调用者处理
                    cont.resume(Triple(bytes, orientation, file))
                } catch (e: Exception) {
                    file.delete()
                    cont.resumeWithException(e)
                }
            }
        })
    }
```

### 3.5 文件命名

```kotlin
private fun generateFileName(): String {
    val now = java.time.LocalDateTime.now()
    return "OpenClaw_${now.format(java.time.format.DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"))}.jpg"
}
```

### 3.6 参数解析

```kotlin
private fun parseSaveToGallery(params: JsonObject?): Boolean? =
    parseJsonBooleanFlag(params, "saveToGallery")
```

## 4. Thread Safety Analysis

| 操作 | 当前线程 | 说明 |
|------|---------|------|
| snap() | Dispatchers.Main | 不变，CameraX 要求主线程 |
| takeJpegWithExif() | Main executor | 不变 |
| saveToGallery() | Dispatchers.IO | 新增，MediaStore 写入 |
| 文件删除 | IO | 移入 snap() 的 finally 块 |

## 5. Error Handling

| 场景 | 行为 | 日志级别 |
|------|------|---------|
| MediaStore.insert() 返回 null | 记日志，返回 false | ERROR |
| openOutputStream() 返回 null | 删除空记录，记日志 | ERROR |
| 写入过程中 IOException | catch 后记日志，返回 false | ERROR |
| 文件名冲突 | MediaStore 自动处理 | — |
| 存储空间不足 | IOException，catch 后记日志 | ERROR |

## 6. Android Version Compatibility

| Android 版本 | API Level | 注意事项 |
|-------------|-----------|---------|
| 12 | 31 | minSdk，IS_PENDING 可用 |
| 13 | 33 | RELATIVE_PATH 行为不变 |
| 14 | 34 | 无变化 |
| 15 | 35 | 无变化 |
| 16 | 36 | targetSdk，无变化 |

**无需处理的情况**：
- WRITE_EXTERNAL_STORAGE 权限（API 29+ 不需要）
- READ_MEDIA_IMAGES 权限（我们只写入不读取，不需要）

## 7. Verification

```bash
cd apps/android
./gradlew :app:assemblePlayDebug
adb install -r app/build/outputs/apk/play/debug/openclaw-*-play-debug.apk

# 测试 saveToGallery=false（默认行为，不保存）
# 测试 saveToGallery=true（保存到相册）
adb logcat -s CameraCaptureManager
```
