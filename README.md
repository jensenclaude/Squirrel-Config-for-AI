# Squirrel（鼠须管）中英切换改造经验总结

> 环境：macOS + Squirrel 1.1.2 + 雾凇拼音（rime_ice，小鹤双拼）+ Karabiner-Elements 16.1.0 + iTerm2
> 目标：把中英切换从左 Shift 单击改为**点按左 Command**，且 Cmd+C/V 等快捷键零干扰、终端无字符泄漏
> 完成日期：2026-08-21

全新 Mac 复刻本方案请直接看 **[从零配置指南](#从零配置指南全新-mac)**；踩坑记录从[这里](#踩过的四个坑按时间顺序每个都是架构级限制)开始。

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

## 从零配置指南（全新 Mac）

> 适用场景：一台什么都没装的新 Mac（没有 Squirrel / Rime / Karabiner-Elements），
> 从零复刻本方案。全程约 15 分钟，其中大头是 Karabiner 的权限授权。
> 已验证环境：macOS + Squirrel 1.1.2 + rime_ice + Karabiner-Elements 16.1.0。

### 第 0 步：准备

- 可选但推荐：安装 [Homebrew](https://brew.sh)（下面的命令都用 brew，手动下载安装也行）
- 把本仓库 clone 到任意目录（下文用 `$REPO` 指代）：

```bash
git clone <本仓库地址> ~/Squirrel-Config-for-AI
cd ~/Squirrel-Config-for-AI   # 即 $REPO
```

### 第 1 步：安装 Squirrel（鼠须管）

```bash
brew install --cask squirrel
```

装完后**必须手动添加输入法**（brew 装完不会自动启用）：

1. 系统设置 → 键盘 → 键盘输入法 → 编辑 → 添加
2. 选「简体中文」分类下的「鼠须管」
3. 用输入法菜单（或 Ctrl+Space）切换到鼠须管

此时能打字，但只有默认配置（明月拼音简化字）。

### 第 2 步：安装雾凇拼音（rime_ice）

用东风破（plum）安装到 `~/Library/Rime`：

```bash
mkdir -p ~/Library/Rime
cd ~/Library/Rime
bash <(curl -L https://github.com/rime/plum/raw/master/rime-install) iDvel/rime-ice:others/recipes/full
```

装完先**不要**急着部署——下一步还要放 custom 配置，一次部署就够。

### 第 3 步：放入本仓库的 Rime 配置

```bash
cd $REPO
cp config-backup/default.custom.yaml \
   config-backup/double_pinyin_flypy.custom.yaml \
   config-backup/squirrel.custom.yaml \
   ~/Library/Rime/
```

三个文件各自的作用见上文「相关文件」表格。要点：

- `default.custom.yaml` 里 `schema_list` 只保留了 `double_pinyin_flypy`（小鹤双拼），
  想加别的方案就往 `schema_list` 里追加
- 此时 `switch_key` 已配好 `Control_R: commit_code`（右 Ctrl 单击切中英）和
  `Shift_L: commit_code`（左 Shift 切中英）

### 第 4 步：部署 Rime 并验证

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --reload
```

⚠️ 两个易踩的点：
- **路径含空格**（`Input Methods`），必须带引号，裸敲会报「找不到文件」
- **Finder 默认隐藏 `/Library`（资源库）目录**，找不到时按 `Cmd+Shift+G` 输入
  `/Library/Input Methods` 直达；嫌命令麻烦也可以直接点菜单栏输入法图标 → **重新部署**，
  效果等价

验证：

1. 看 `~/Library/Rime/build/` 下是否生成了 `double_pinyin_flypy.prism.bin`、
   `double_pinyin_flypy.reverse.bin` 等编译产物（有 = patch 生效）
2. 随便找个输入框，**单独点按左 Shift** 应能切换中英（此时还没装 Karabiner，
  这是纯 Rime 侧的能力，先确认这一层是好的）

### 第 5 步：安装 Karabiner-Elements

```bash
brew install --cask karabiner-elements
```

首次启动会引导授权（这是全新 Mac 上最容易卡住的地方）：

1. 启动 Karabiner-Elements，弹出权限引导窗口
2. 按提示到 系统设置 → 隐私与安全性 → 输入监控，允许 Karabiner 相关组件
   （可能要求重启一次 Karabiner 或注销重登）
3. 授权完成后先**退出 Karabiner**（菜单栏图标 → Quit Karabiner-Elements），
  避免它用默认配置干扰下一步

### 第 6 步：放入 karabiner.json 并重载

```bash
mkdir -p ~/.config/karabiner
cp $REPO/config-backup/karabiner.json ~/.config/karabiner/karabiner.json
```

⚠️ 这是**整体覆盖**——如果这台 Mac 上已有自定义 Karabiner 规则，先备份原文件。

然后启动 Karabiner-Elements（或对已运行的实例执行重载）：

```bash
pkill -x karabiner_console_user_server   # 会自动重启并加载新配置
```

### 第 7 步：端到端验证

| 操作 | 预期效果 |
|---|---|
| 单独点按物理左 Ctrl | 切换中英 |
| 按住物理左 Ctrl + C / V | 复制 / 粘贴（逻辑 Cmd） |
| 物理左 Command + C（终端里） | 发送 Ctrl+C 中断信号 |
| 终端（iTerm2 等）里点按左 Ctrl | 只切换中英，**无乱码、无字符泄漏** |
| 单独点按物理左 Command | 无任何反应（不会激活菜单栏） |

如果哪一条不对，用 Karabiner-EventViewer.app 看实际发出的按键事件，
用 Rime 日志看是否 `updated option: ascii_mode`（见下文「排查工具箱」）。

### 一键脚本（第 2/3/6 步可合并）

```bash
REPO=~/Squirrel-Config-for-AI   # 按实际路径改
mkdir -p ~/Library/Rime && cd ~/Library/Rime
bash <(curl -L https://github.com/rime/plum/raw/master/rime-install) iDvel/rime-ice:others/recipes/full
cp "$REPO"/config-backup/{default.custom.yaml,double_pinyin_flypy.custom.yaml,squirrel.custom.yaml} ~/Library/Rime/
"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --reload
mkdir -p ~/.config/karabiner && cp "$REPO"/config-backup/karabiner.json ~/.config/karabiner/
pkill -x karabiner_console_user_server || true
```

（Squirrel 和 Karabiner 的安装与系统授权无法脚本化，仍需手动完成。）

## 迁移打字习惯（用户词典）到新 Mac

「哪些字/词排在前面」存在两个地方：**userdb**（自动调频 + 自造词，`~/Library/Rime/*.userdb/`）
和 **custom_phrase.txt**（自定义短语）。配置文件只管「怎么打」，这两个才管「习惯」，
从零配置指南装完的只是干净的新机器，习惯要单独迁。

### 旧 Mac（导出）

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --sync
```

（或菜单栏输入法图标 → 同步用户数据，效果一样）

执行后生成快照目录 `~/Library/Rime/sync/<installation_id>/`，里面的关键文件：

- `rime_ice.userdb.txt` —— 打字习惯的**可读文本**（每行 `拼音<TAB>词	c=次数 d=距上次天数 t=时刻`），
  二进制 userdb 被导出成文本，天然跨机器、跨版本
- `custom_phrase.txt` —— 自定义短语快照

用 U 盘 / scp / 网盘把整个 `~/Library/Rime/sync/` 目录搬到新 Mac。

### 新 Mac（导入，在完成从零配置指南第 4 步之后）

```bash
mkdir -p ~/Library/Rime/sync
# 把旧机的 sync/<id>/ 目录放进 ~/Library/Rime/sync/ 下
"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --sync
```

再次执行 sync 时，Rime 发现 sync 下有「别的机器的快照」（installation_id 不同），
会把其中的词条**合并**进本机 userdb——不是覆盖，两台机器的习惯可以叠加。

然后手动补上自定义短语（sync 快照里的 txt 不会自动合并）：

```bash
cp /path/to/旧机快照/custom_phrase.txt ~/Library/Rime/
"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --reload
```

### 验证

在新 Mac 上打几个旧机上的高频词（查 `rime_ice.userdb.txt` 里 `c=` 大的），
看是否排在候选前列。

### 备选：直接拷贝 userdb（不推荐但能用）

`killall Squirrel` 后整目录拷贝 `rime_ice.userdb/` 到新机同路径，重启即可。
缺点：LevelDB 二进制格式，拷贝前必须停掉 Squirrel（否则 `.log` 里有未落盘数据），
且机器间是覆盖不是合并。sync 文本快照没这些问题，优先用 sync。

### 多机长期同步（可选）

`~/Library/Rime/installation.yaml` 里可设 `sync_dir` 指向 iCloud / Dropbox 目录，
两台机器各自 sync，就能走云端互相合并习惯，不用每次 U 盘搬。

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
- **改 Rime 配置后**：`"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --reload` 重新部署（路径含空格要带引号；或直接点菜单栏输入法图标 → 重新部署），然后看 `~/Library/Rime/build/` 确认。注意 build 产物的时间戳只在源文件有改动时才会更新——配置没变时 reload 不重写产物，属正常现象
- Rime 日志里的 `starting engine` 是会话重建（切应用/重部署），不是错误
- `ascii_composer/switch_key` 的切换只在「修饰键单独按下再单独松开」时触发，按住期间敲过任何键都会取消——这是 Cmd 快捷键不误触发的根本保证
- 如果不想要左 Shift 切换：`default.custom.yaml` 里把 `Shift_L: commit_code` 改成 `noop`
- 如果默认想中文：`double_pinyin_flypy.custom.yaml` 里把 `reset: 1` 改成 `reset: 0`（或删掉该行 = 记住上次状态）
