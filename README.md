# Squirrel（鼠须管）中英切换改造经验总结

## 最终架构

```
我们在设置里先把键盘上的 Ctrl 映射为 Ctrl，Win 映射为 Command

Karabiner 被配置为：
物理左 Ctrl 单击 -> Karabiner 合成「右 Ctrl 单击」-> flagsChanged 经过输入法 -> Rime 切换中英
物理左 Ctrl 按住 -> Karabiner lazy 发送左 Command -> Command 组合键（复制粘贴等）
物理左 Win -> Karabiner 发送左 Ctrl -> Ctrl 组合键（Ctrl+C 中断等）
```

为什么用「右 Ctrl」而不是别的键：**MacBook 键盘物理上不存在右 Ctrl**，它的出现只能来自 Karabiner 合成，语义干净零冲突；且修饰键事件（flagsChanged）在所有应用（包括终端）里都会经过输入法——这是整个方案成立的基石。

## 相关文件

| 配置谁                | 文件                                   | 目标路径（实际路径）                                     | 作用                                                                                   |
| ------------------ | ------------------------------------ | ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| Karabiner-Elements | karabiner/karabiner.json             | ~/.config/karabiner/karabiner.json             | 物理左 Ctrl 单击 -> Karabiner 合成「右 Ctrl 单击」；<br>物理左 Ctrl 按住 -> Karabiner lazy 发送左 Command |
| Rime               | Rime/default.custom.yaml             | ~/Library/Rime/default.custom.yaml             | 设置 右 Ctrl 为中/英文切换键；<br>加入万象语法模型（如果不想要可以去掉）                                           |
| Rime               | Rime/double_pinyin_flypy.custom.yaml | ~/Library/Rime/double_pinyin_flypy.custom.yaml | 输入法默认英文                                                                              |
| Rime               | Rime/squirrel.custom.yaml            | ~/Library/Rime/squirrel.custom.yaml            | 主题配色（与本次改造无关，一并备份）                                                                   |

## 从零配置指南（全新 Mac）

> 适用场景：一台什么都没装的新 Mac（没有 Squirrel / Rime / Karabiner-Elements），
> 从零复刻本方案。全程约 15 分钟，其中大头是 Karabiner 的权限授权。
> 已验证环境：macOS + Squirrel 1.1.2 + rime_ice + Karabiner-Elements 16.1.0。

### 安装鼠须管（Rime）

这个软件包已经包含了：

- Rime——核心输入法引擎，负责编码解析、候选生成、词典查询、选词逻辑）；
- Squirrel——macOS 输入法前端，负责键盘事件、候选窗口、系统交互、界面样式。

```shell
# 安装 Squirrel（鼠须管）
brew install --cask squirrel-app
```

装完后必须手动添加输入法：

1. 系统设置 → 键盘 → 键盘输入法 → 编辑 → 添加 → 选「简体中文」分类下的「鼠须管」；
2. 用输入法菜单（或 Ctrl+Space）切换到鼠须管。

此时能打字，但只有默认配置。

### 第 2 步：安装雾凇拼音输入法配置（rime_ice）

可以去 https://github.com/iDvel/rime-ice 查看安装步骤。

安装双拼的步骤大概是：

```shell
# 1. 安装 plum
cd ~
git clone https://github.com/rime/plum.git plum

# 2. 安装雾凇拼音到默认客户端（Squirrel）：
cd ~/plum
bash rime-install iDvel/rime-ice

# 3. 双拼用户额外执行，替换「double_pinyin_flypy」为你使用的方案：
bash rime-install iDvel/rime-ice:others/recipes/config:schema=double_pinyin_flypy

# 4. （可选）万象语法模型额外执行，替换「double_pinyin_flypy」为你使用的方案：
bash rime-install iDvel/rime-ice:others/recipes/grammar:schema=double_pinyin_flypy

# 5. 重新部署

# 方案名称
# rime_ice（雾凇拼音、全拼）
# double_pinyin（自然码双拼）
# double_pinyin_flypy（小鹤双拼）
# double_pinyin_mspy（微软双拼）
# double_pinyin_sogou（搜狗双拼）
# double_pinyin_abc（智能 ABC 双拼）
# double_pinyin_jiajia（拼音加加双拼）
# double_pinyin_ziguang（紫光双拼）
```

推荐安装万象语法模型，使用 n-gram 算法配合已经训练好的静态的 .gram 模型文件（这个模型 370MB 左右）来提高候选词准确率，和用户词典正交。

### 第 3 步：配置 Rime

```bash
# clone 本仓库
git clone <本仓库地址> ./Squirrel-Config-for-AI

# 把本仓库的 Rime 配置放到 ~/Library/Rime/ 下
cd Squirrel-Config-for-AI
cp Rime/default.custom.yaml \
   Rime/double_pinyin_flypy.custom.yaml \
   Rime/squirrel.custom.yaml \
   ~/Library/Rime/
```

此时 `switch_key` 已配好：

- `Control_R: commit_code`（右 Ctrl 单击切中英）；
- `Shift_L: commit_code`（左 Shift 单击切换中英）。

如果不想要 左 Shift 直接改为 `Shift_L: noop` 就行。

### 第 4 步：让 Rime 重载配置

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --reload
# 或者在菜单栏里找到鼠须管输入法，在下拉菜单里点“重新部署”，
# 或者按快捷键 control + option + `
```

验证：

1. 看 `~/Library/Rime/build/` 下是否生成了 `double_pinyin_flypy.prism.bin`、`double_pinyin_flypy.schema.yaml` 等编译产物（有 = patch 生效）；
2. 随便找个输入框，**单击左 Shift** 应能切换中英（此时还没装 Karabiner，这是纯 Rime 侧的能力，先确认这一层是好的）。

### 第 5 步：安装 Karabiner-Elements

```bash
brew install --cask karabiner-elements
```

首次启动会引导授权，授权完成后先**退出 Karabiner**（菜单栏图标 → Quit Karabiner-Elements），避免它用默认配置干扰下一步。

### 第 6 步：放入 karabiner.json 并重载

```bash
mkdir -p ~/.config/karabiner
cp Squirrel-Config-for-AI/karabiner/karabiner.json ~/.config/karabiner/karabiner.json
```

⚠️ 这是**整体覆盖**——如果这台 Mac 上已有自定义 Karabiner 规则，先备份原文件；然后重启 Karabiner-Elements：

```bash
pkill -x karabiner_console_user_server # 会自动重启并加载新配置
# 或者直接在 Karabiner-Elements 的 Quit, Restart 界面重启
```

### 第 7 步：端到端验证

| 操作                      | 预期效果                |
| ----------------------- | ------------------- |
| 单击物理左 Ctrl              | 切换中英                |
| 按住物理左 Ctrl + C / V      | 复制 / 粘贴（逻辑 Command） |
| 物理左 Win + C（终端里）        | 发送 Ctrl+C 中断信号      |
| 终端（iTerm2 等）里单击物理左 Ctrl | 只切换中英，无乱码、无字符泄漏     |
| 单击物理左 Win               | 无任何反应（不会激活菜单栏）      |

如果哪一条不对，用 Karabiner-EventViewer.app 看实际发出的按键事件。

## 迁移打字习惯（用户词典）到新 Mac

「哪些字/词排在前面」存在两个地方：**userdb**（自动调频 + 自造词，`~/Library/Rime/*.userdb/`）

和 **custom_phrase.txt**（自定义短语）。配置文件只管「怎么打」，这两个才管「习惯」，

从零配置指南装完的只是干净的新机器，习惯要单独迁。

### 旧 Mac 导出

本仓库也提供了一个 `sync/<id>` 目录，不定时更新。

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --sync
# 或菜单栏输入法图标 → 同步用户数据，效果一样
```

执行后生成快照目录 `~/Library/Rime/sync/<installation_id>/`，里面的关键文件：

- `rime_ice.userdb.txt` —— 打字习惯的**可读文本**（每行 `拼音<TAB>词	c=次数 d=距上次天数 t=时刻`），二进制 userdb 被导出成文本，天然跨机器、跨版本；
- `custom_phrase.txt` —— 自定义短语快照。

把 `~/Library/Rime/sync/<id>/` 目录搬到新 Mac。

### 新 Mac 导入

```bash
mkdir -p ~/Library/Rime/sync
# 把旧机的 sync/<id>/ 目录放到 ~/Library/Rime/sync/ 下
mv sync/<id>/ ~/Library/Rime/sync/
"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --sync
```

再次执行 sync 时，Rime 发现 sync 下有「别的机器的快照」（installation_id 不同），会把其中的词条**合并**进本机 userdb——不是覆盖，两台机器的习惯可以叠加。然后手动补上自定义短语（sync 快照里的 txt 不会自动合并）：

```bash
cp custom_phrase.txt ~/Library/Rime/
"/Library/Input Methods/Squirrel.app/Contents/MacOS/Squirrel" --reload
```

### 验证

在新 Mac 上打几个旧机上的高频词（查 `rime_ice.userdb.txt` 里 `c=` 大的），看是否排在候选前列。

### 备选

- 直接拷贝 userdb（LevelDB 数据库）

`killall Squirrel` 后整目录拷贝 `rime_ice.userdb/` 到新机同路径，重启即可。缺点：LevelDB 二进制格式，拷贝前必须停掉 Squirrel（否则 `.log` 里有未落盘数据），且机器间是覆盖不是合并。sync 文本快照没这些问题，优先用 sync。

- 多机长期同步（可选）

`~/Library/Rime/installation.yaml` 里可设 `sync_dir` 指向 iCloud / Dropbox 目录，两台机器各自 sync，就能走云端互相合并习惯。

## 维护备忘

- 改 karabiner.json 后不会自动热重载：必须重启；
- `ascii_composer/switch_key` 的切换只在「修饰键单独按下再单独松开」时触发，按住期间敲任何键都会取消——这是 Command 快捷键不误触发的根本保证；
- 如果不想要左 Shift 切换：`default.custom.yaml` 里把 `Shift_L: commit_code` 改成 `noop`；
- 如果默认想中文：`double_pinyin_flypy.custom.yaml` 里把 `reset: 1` 改成 `reset: 0`（如果删掉该行代表记住上次状态）。
