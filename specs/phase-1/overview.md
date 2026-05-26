# Feature Specification: Phase 1 - Phone System Control

**Feature Branch**: `develop`

**Created**: 2026-05-26

**Status**: Draft

**Input**: User description: "完善上游 Android 端已有的功能，新增手机本地硬件控制能力"

## 功能总览

### A. 上游现有功能（基线）

#### 相机

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 拍照 | camera.snap | ⚠️ 需完善 | 缺少 saveToGallery |
| 录像 | camera.clip | ✅ 已有 | — |
| 相机列表 | camera.list | ✅ 已有 | — |

#### 设备信息

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 设备状态 | device.status | ✅ 已有 | 电池、存储、网络、热状态 |
| 设备信息 | device.info | ✅ 已有 | 型号、系统版本、App 版本 |
| 设备权限 | device.permissions | ✅ 已有 | 相机/麦克风/定位等权限状态 |
| 设备健康 | device.health | ✅ 已有 | 内存、电池健康、功耗状态 |

#### 定位

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 获取位置 | location.get | ✅ 已有 | — |

#### 通知

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 通知列表 | notifications.list | ✅ 已有 | — |
| 通知操作 | notifications.actions | ✅ 已有 | — |

#### 联系人

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 搜索联系人 | contacts.search | ✅ 已有 | — |
| 添加联系人 | contacts.add | ✅ 已有 | — |

#### 日历

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 查看日程 | calendar.events | ✅ 已有 | — |
| 添加日程 | calendar.add | ✅ 已有 | — |

#### 运动

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 活动识别 | motion.activity | ✅ 已有 | — |
| 计步器 | motion.pedometer | ✅ 已有 | — |

#### SMS 与通话记录

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 发送短信 | sms.send | ⚠️ 仅 thirdParty | Play flavor 不可用 |
| 搜索短信 | sms.search | ⚠️ 仅 thirdParty | Play flavor 不可用 |
| 通话记录 | callLog.search | ⚠️ 仅 thirdParty | Play flavor 不可用 |

#### 系统

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 系统通知 | system.notify | ✅ 已有 | — |

#### Web UI

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 显示页面 | canvas.present | ✅ 已有 | — |
| 隐藏页面 | canvas.hide | ✅ 已有 | — |
| 页面导航 | canvas.navigate | ✅ 已有 | — |
| 执行 JS | canvas.eval | ✅ 已有 | — |
| 屏幕截图 | canvas.snapshot | ✅ 已有 | — |

#### 照片

| 模块 | 命令 | 状态 | 说明 |
|------|------|------|------|
| 最新照片 | photos.latest | ✅ 已有 | — |

### B. 上游功能完善（本阶段目标）

| 模块 | 命令 | 当前问题 | 目标 |
|------|------|---------|------|
| 相机拍照 | camera.snap | 可拍照，不保存到相册 | 新增 saveToGallery 参数，拍照后保存到 Pictures/OpenClaw/ |

### C. 新增设备控制能力

#### 屏幕控制

| 模块 | 命令 | 当前状态 | 目标 |
|------|------|---------|------|
| 亮度调节 | device.screen.brightness | 未实现 | 可读取和设置屏幕亮度（0-255） |
| 屏幕超时 | device.screen.timeout | 未实现 | 可读取和设置屏幕超时时间 |

#### 声音与模式

| 模块 | 命令 | 当前状态 | 目标 |
|------|------|---------|------|
| 音量控制 | device.audio.volume | 未实现 | 可读取和设置媒体/铃声/闹钟音量 |
| 响铃模式 | device.audio.mode | 未实现 | 切换静音/震动/响铃模式 |

#### 网络控制

| 模块 | 命令 | 当前状态 | 目标 |
|------|------|---------|------|
| WiFi 开关 | device.wifi | 未实现 | 读取和切换 WiFi 开关状态 |
| 蓝牙开关 | device.bluetooth | 未实现 | 读取和切换蓝牙开关状态 |
| 飞行模式 | device.airplane | 未实现 | 读取和切换飞行模式 |

#### 性能与功耗控制

| 模块 | 命令 | 当前状态 | 目标 |
|------|------|---------|------|
| 性能模式 | device.power.mode | 未实现 | 切换省电/均衡/高性能模式 |
| 功耗调节 | device.power.policy | 未实现 | 设置功耗限制策略 |
| CPU 调度 | device.power.cpu | 未实现 | 读取和设置 CPU 调度策略 |

## 成功标准

- 每个新增命令可被 AI 通过 Gateway 调用并返回正确结果
- 所有控制命令在真机上验证通过
- saveToGallery 功能在主流 Android 版本（12-16）上一致可用

## 边界约定

- 本次不涉及无障碍服务（Accessibility Service），App 自动化操控放到 Phase 2
- 每个功能的详细设计在各自的 plan.md 中展开
- 部分功能需系统级权限（如 WRITE_SETTINGS），测试时手动授权

## 变更记录

| 版本 | 日期 | 变更内容 | 确认人 |
|------|------|---------|--------|
| 1.0.0 | 2026-05-26 | 初始版本 | huangfusheng, sunxuewen |
