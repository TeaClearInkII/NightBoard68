# FIXLOG — 本次 PR 修复的问题

> 与 [归档/CHANGELOG.md](归档/CHANGELOG.md)（功能更新）配套；本文件只记录 **bug 修复**。

## 蓝牙模式：长时间不用 / 切后台回来，第一次按键卡键连发

- **现象**：切到其他应用一段时间后回来，第一次点击键盘，某个键在电脑端一直连打到上限，停不下来。
- **根因**：HID 键盘报告是「当前按住键集合」，`keyUp` 先清集合再 `sync()` 重发；若发送失败（`host`/`hidDevice` 瞬时为空被静默 `return`、或 `sendReport` 抛异常被吞），**手机端认为已抬起、电脑端永远收不到抬起** → OS 认为键按住，自动连发；且蓝牙重连建立后不重发状态，抬起永远不会补发。
- **修复**（[HidKeyboard.kt](app/src/main/java/com/nightboard/keyboard68/HidKeyboard.kt)）：
  - `sync()` 发送失败 / 连接未就绪时置 `dirty` 标记，状态不丢；
  - 蓝牙重连建立（`onConnectionStateChanged` CONNECTED）时若 `dirty` 或仍有按键/修饰位 → 补发全量当前报告（已无按键则发空报告解除卡键）。

## 局域网模式：数字小键盘无输出

- **现象**：局域网模式下按数字小键盘完全没反应（蓝牙正常）。
- **根因**：Agent 的 KeyMap 缺少小键盘专用键码（0x53~0x63）映射，`KeyDown` 查表失败直接丢弃。
- **修复**（[NightBoardAgent.cs](pc-agent/NightBoardAgent.cs)）：补全小键盘 HID 码 → Set1 扫描码映射；输出方向/数字由电脑端 NumLock 决定（与蓝牙 HID 一致）。