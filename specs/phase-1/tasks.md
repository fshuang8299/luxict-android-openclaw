# Tasks: Phase 1 — Phone System Control

**Input**: Design documents from `specs/phase-1/plan.md`

**Prerequisites**: plan.md (required), spec/overview.md (required)

## Path Conventions

- **Protocol**: `apps/android/app/src/main/java/ai/openclaw/app/protocol/OpenClawProtocolConstants.kt`
- **Registry**: `apps/android/app/src/main/java/ai/openclaw/app/node/InvokeCommandRegistry.kt`
- **Dispatcher**: `apps/android/app/src/main/java/ai/openclaw/app/node/InvokeDispatcher.kt`
- **Handler**: `apps/android/app/src/main/java/ai/openclaw/app/node/DeviceHandler.kt`
- **Camera**: `apps/android/app/src/main/java/ai/openclaw/app/node/CameraCaptureManager.kt`
- **Tests**: `apps/android/app/src/test/java/ai/openclaw/app/node/`

---

## T1: save-to-gallery

- [ ] T1.1 Refactor `takeJpegWithExif()`: return `Triple<ByteArray, Int, File>` in `CameraCaptureManager.kt`
- [ ] T1.2 Add `saveToGallery()` method using MediaStore API
- [ ] T1.3 Add `parseSaveToGallery()` in `CameraCaptureManager.kt`
- [ ] T1.4 Integrate saveToGallery into `snap()`: parse param → call → fail-open
- [ ] T1.5 Write unit tests
- [ ] T1.6 Build: `./gradlew :app:assemblePlayDebug`
- [ ] T1.7 Deploy: `adb install -r <apk>`
- [ ] T1.8 Verify: saveToGallery=false no photo added; saveToGallery=true photo in Pictures/OpenClaw/
- [ ] T1.9 Review Round 1-3 + archive records

---

## T2: screen-control

- [ ] T2.1 Add `ScreenBrightness`, `ScreenTimeout` to ProtocolConstants enum
- [ ] T2.2 Register 2 commands in InvokeCommandRegistry
- [ ] T2.3 Add 2 when{} branches in InvokeDispatcher
- [ ] T2.4 Implement `handleScreenBrightness()`: Settings.System read/write
- [ ] T2.5 Implement `handleScreenTimeout()`: Settings.System read/write
- [ ] T2.6 Add WRITE_SETTINGS permission handling
- [ ] T2.7 Write unit tests
- [ ] T2.8 Build + deploy + verify on device
- [ ] T2.9 Review Round 1-3 + archive records

---

## T3: audio-control

- [ ] T3.1 Add `AudioVolume`, `AudioMode` to ProtocolConstants enum
- [ ] T3.2 Register 2 commands in registry
- [ ] T3.3 Add 2 when{} branches in dispatcher
- [ ] T3.4 Implement `handleAudioVolume()`: AudioManager get/set stream volume
- [ ] T3.5 Implement `handleAudioMode()`: AudioManager ringer mode
- [ ] T3.6 Write unit tests
- [ ] T3.7 Build + deploy + verify on device
- [ ] T3.8 Review Round 1-3 + archive records

---

## T4: network-control

- [ ] T4.1 Add `Wifi`, `Bluetooth`, `AirplaneMode` to ProtocolConstants enum
- [ ] T4.2 Register 3 commands in registry
- [ ] T4.3 Add 3 when{} branches in dispatcher
- [ ] T4.4 Implement `handleWifi()`: WifiManager + CHANGE_WIFI_STATE
- [ ] T4.5 Implement `handleBluetooth()`: BluetoothAdapter + BLUETOOTH_CONNECT
- [ ] T4.6 Implement `handleAirplaneMode()`: Settings.Global + Intent broadcast
- [ ] T4.7 Write unit tests
- [ ] T4.8 Build + deploy + verify on device
- [ ] T4.9 Review Round 1-3 + archive records

---

## T5: power-control

- [ ] T5.1 Add `PowerMode`, `PowerPolicy`, `PowerCpu` to ProtocolConstants enum
- [ ] T5.2 Register 3 commands in registry
- [ ] T5.3 Add 3 when{} branches in dispatcher
- [ ] T5.4 Implement `handlePowerMode()`: PowerManager.setPowerSaveMode
- [ ] T5.5 Implement `handlePowerPolicy()`: power saving hints
- [ ] T5.6 Implement `handlePowerCpu()`: CPU frequency/scheduling queries
- [ ] T5.7 Write unit tests
- [ ] T5.8 Build + deploy + verify on device
- [ ] T5.9 Review Round 1-3 + archive records
