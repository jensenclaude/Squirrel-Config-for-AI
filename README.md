# Squirrel（鼠须管）中英切换改造经验总结

> 环境：macOS + Squirrel 1.1.2 + 雾凇拼音（rime_ice，小鹤双拼）+ Karabiner-Elements 16.1.0 + iTerm2
> 目标：把中英切换从左 Shift 单击改为**点按左 Command**，且 Cmd+C/V 等快捷键零干扰、终端无字符泄漏
> 完成日期：2026-08-21

## 最终架构

```
物理左 Ctrl 点按 ──→ Karabiner 合成「右 Ctrl 单击」──→ flagsChanged 经过输入法 ──→ Rime 切换中英
物理左 Ctrl 按住 ──→ Karabiner lazy 发送左 Command ──→ Cmd 组合键（复制粘贴等）
物理左 Command ──→ Karabiner 发送左 Ctrl ──→ Ctrl 组合键（Ctrl+C 中断等）
右侧同理互换：右 Command ↔ 右 Ctrl（外接键盘场景）
```

另支持 **Control ↔ Command 互换**（2026-08-21 追加）：
- 物理左 Ctrl = 逻辑 Command：按住是 Cmd 组合键（物理 Ctrl+C 就是复制），**单独点按切换中英**
- 物理左 Command = 逻辑左 Ctrl：纯互换，无点按动作（Rime 侧 Control_L 是 noop，点按无副作用）
- 右侧对称互换；Rime 侧无需改动（`Control_L: noop` + `Control_R: commit_code` 恰好兼容互换后语义）

为什么用「右 Ctrl」而不是别的键：**MacBook 键盘物理上不存在右 Ctrl**，
它的出现只能来自 Karabiner 合成，语义干净零冲突；且修饰键事件（flagsChanged）
在所有应用（包括终端）里都会经过输入法——这是整个方案成立的基石。

## 相关文件（本目录 config-backup/ 内有副本）

| 文件 | 实际路径 | 作用 |
|---|---|---|
| karabiner.json | ~/.config/karabiner/karabiner.json | 左 Command 点按 → 合成 right_control；按住 → 左 Ctrl；物理左/右 Ctrl ↔ Cmd 互换 |
| default.custom.yaml | ~/Library/Rime/default.custom.yaml | `Control_R: commit_code` 切换键；`Shift_L: commit_code` 保留 |
| double_pinyin_flypy.custom.yaml | ~/Library/Rime/double_pinyin_flypy.custom.yaml | `switches/@0/reset: 1` 默认英文 |
| squirrel.custom.yaml | ~/Library/Rime/squirrel.custom.yaml | 主题配色（与本次改造无关，一并备份） |

## 踩过的四个坑（按时间顺序，每个都是架构级限制）

### 坑 1：`Super_L: commit_code` 直配 → Cmd+C/V 误触发切换

直觉改法是在 `ascii_composer/switch_key` 里配 `Super_L: commit_code`（macOS 的 Command = Rime 的 Super）。
但读 Squirrel 源码（`SquirrelInputController.swift` keyDown 分支）发现：

```swift
// Let client apps handle Command shortcuts.
if modifiers.contains(.command) {
  break   // ← 带 Command 的组合键根本不转发给 librime
}
```

于是 librime 只看到 `Super_L↓ … Super_L↑`，中间的 C/V 被 Squirrel 吞掉了，
Rime 以为你「单独单击了 Command」→ 每次 Cmd+C/V 都切换一次。
**结论：纯 Rime 配置无法用 Command 做切换键，必须在 Squirrel 下游/上游想办法。**

### 坑 2：合成功能键 F18 → 英文模式下失效 + 输出不可见字符

方案改为 Karabiner 点按 Cmd 时合成 F18，Rime 用 `key_binder` 绑 `F18 → toggle ascii_mode`。
结果：中文模式正常切换，**英文模式下不切换且输出 `` 字符**。

原因：librime 的处理器链里 `ascii_composer` 排在 `key_binder` 之前，
英文（ascii）模式下它把无修饰键直接 `kRejected` 透传给应用：

```cpp
if (ascii_mode) {
  if (!ctx->IsComposing()) {
    return kRejected;  // ← key_binder 根本没机会处理
  }
}
```

中途试过 `Ctrl+F18`（带修饰键会被放行）→ 变成「不切换 + 系统提示音」：
Squirrel 转发按键前检查事件的字符属性，**带 Ctrl 的功能键字符属性为空，直接不转发**。
最终修法是把 `key_binder` 在方案里前置到 `ascii_composer` 之前（`double_pinyin_flypy.custom.yaml` 的 `engine/processors` 重排）——这个 hack 后来随 F18 方案一起废弃了，但思路值得记住。

### 坑 3：终端（iTerm2）绕过输入法处理功能键 → 漏出 `` 和 `<F18>`

F18 + processor 前置后普通应用正常了，但 iTerm2 里点按 Cmd 会输出 `` 字符，
新开的终端里还能看到 `<F18>` 字面字符串。

原因：终端类应用的职责是把按键编码成字节流发给 shell，**功能键不问输入法直接处理**。
F18 被编码成转义序列/私用区伪字符（U+F7xx）写进命令行。
而修饰键的 flagsChanged 事件在终端里也会经过输入法——这正是「原生 Shift 切换在终端从不出问题」的原因。
**结论：任何「合成功能键给输入法」的方案在终端里天生走不通；必须合成修饰键。**

### 坑 4：Karabiner 非 lazy 透传 Cmd → 同一次点按双触发

右 Ctrl 方案上线后，日志显示每次点按触发两次切换（间隔仅 ~4ms，机器事件级重复）：

```
19:10:37.588886  updated option: ascii_mode
19:10:37.593198  updated option: ascii_mode   ← 同一次点按
```

原因：`to: [left_command]` 原样透传 Cmd 按下/抬起，macOS 对「单独点按 Cmd」
有激活菜单栏焦点的特殊行为，扰动输入法会话导致事件重复投递。

修法：给透传的 Cmd 加 `lazy: true`——**按住期间真的按了别的键才发出 Cmd**，
单独点按时系统完全看不到 Cmd，只发右 Ctrl 单击。副作用是点按 Cmd 菜单栏也不闪了，纯赚。

## 排查工具箱（这次用过的）

| 工具/位置 | 用途 |
|---|---|
| Karabiner-EventViewer.app | 直接看 Karabiner 实际发出的按键事件（验证合成是否正确） |
| `/var/folders/*/T/rime.squirrel/rime.squirrel.*.log.INFO.*` | Rime 运行日志，`updated option: ascii_mode` = 一次切换，时间戳可查双触发 |
| `~/Library/Rime/build/` | Rime 编译产物，改完配置必须检查这里确认 patch 真的生效 |
| Squirrel 源码 `sources/SquirrelInputController.swift` | 按键转发的第一现场（github.com/rime/squirrel） |
| librime 源码 `src/rime/gear/ascii_composer.cc` | 切换判定逻辑（单独按下再松开才触发） |

## 维护备忘

- **改 karabiner.json 后不会自动热重载**：必须 `pkill -x karabiner_console_user_server`（会自动重启并加载新配置）
- **改 Rime 配置后**：`"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --reload` 重新部署，然后看 `~/Library/Rime/build/` 确认
- Rime 日志里的 `starting engine` 是会话重建（切应用/重部署），不是错误
- `ascii_composer/switch_key` 的切换只在「修饰键单独按下再单独松开」时触发，按住期间敲过任何键都会取消——这是 Cmd 快捷键不误触发的根本保证
- 如果不想要左 Shift 切换：`default.custom.yaml` 里把 `Shift_L: commit_code` 改成 `noop`
- 如果默认想中文：`double_pinyin_flypy.custom.yaml` 里把 `reset: 1` 改成 `reset: 0`（或删掉该行 = 记住上次状态）
