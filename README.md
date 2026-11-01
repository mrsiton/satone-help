[README.md](https://github.com/user-attachments/files/34230412/README.md)
[AGENT-INSTALL.md](https://github.com/user-attachments/files/34230416/AGENT-INSTALL.md)
# AGENT-INSTALL — 给自动化 agent 的安装 / 验证 / 回滚说明

本文件是机器可读的安装流程，供 AI agent 直接执行。人类请看 `README.md`。
本包只做三件事：装 BepInEx、放一个插件、可选地放皮肤与角色卡。**不修改游戏本体文件，不写注册表，不动系统目录。**

---

## 0. 事实（先核对）

- 游戏：`Chill with You : Lo-Fi Story`，Steam **AppID 3548580**，Unity **2022.3.62f2**，Mono 后端，x86_64，窗口 2560×1600（UI 参考分辨率 1728×1080）。
- 默认安装路径：`D:\SteamLibrary\steamapps\common\Chill with You Lo-Fi Story`
- 找不到就解析：`C:\Program Files (x86)\Steam\steamapps\libraryfolders.vdf`（每条 `"path"` 下拼 `steamapps\common\Chill with You Lo-Fi Story`），逐个 `Test-Path`。
- 模组形态：**BepInEx 5.4.23.2 x64 (Mono)** + 单插件 `SatoneHelp.dll`（net472）。
- 装好后游戏根目录**新增**三项：`winhttp.dll`、`doorstop_config.ini`、`BepInEx\`。除此之外不应有任何改动。

---

## 1. 安装

```powershell
$G = "D:\SteamLibrary\steamapps\common\Chill with You Lo-Fi Story"   # 按需替换
# 1) 结束游戏（找不到进程可忽略）
taskkill /IM "Chill With You.exe" /F 2>$null
# 2) 复制三件套到游戏根目录（覆盖）
Copy-Item .\winhttp.dll          $G -Force
Copy-Item .\doorstop_config.ini  $G -Force
Copy-Item .\BepInEx              $G -Recurse -Force
```

安装后必须存在：

```
<GAME>\BepInEx\core\BepInEx.dll
<GAME>\BepInEx\core\0Harmony.dll
<GAME>\BepInEx\plugins\SatoneHelp\SatoneHelp.dll
<GAME>\winhttp.dll
<GAME>\doorstop_config.ini
```

可选数据（缺了会自动生成默认值）：

```
<GAME>\BepInEx\SatoneHelp\skin\*.png            皮肤（round/round_small/pill/circle/bubble_icon/sent）
<GAME>\BepInEx\SatoneHelp\character\satone_card.txt   角色卡
<GAME>\BepInEx\SatoneHelp\avatars\{satone,player}.png 头像
```

**先记下安装前状态**（回滚依据）：`Get-ChildItem $G | Select-Object Name` —— 正常应只有游戏本体文件，不含 `BepInEx`、`winhttp.dll`、`doorstop_config.ini`。

---

## 2. 启动

```powershell
Start-Process "steam://run/3548580"
# 若 Steam 未运行/卡住，改：Start-Process "C:\Program Files (x86)\Steam\steam.exe" -ArgumentList '-applaunch','3548580'
```

已知现象，**不是故障**：
- 游戏会先起一个进程跑到 `AssetLoad` 就退出，再由第二个进程继续到 `RoomScene`（Player.log 里能看到两次 `awake`）。
- 从启动到进入房间约 20–40 秒。
- 若 Steam 卡在 `LaunchApp changed task to CheckShaderDepotManifest`：
  `steam.exe -shutdown` → 等 10 秒 → 删 `D:\SteamLibrary\steamapps\shadercache\3548580` → 重开 Steam → 再 `-applaunch`。

---

## 3. 验证（按顺序，全部应通过）

### 3.1 日志关键字

```
文件： <GAME>\BepInEx\SatoneHelp\satone-help.log
必须出现（按时间顺序）：
  awake ok; datadir=
  scene RoomScene (Single)
  game ui found: Paremt/PCPlatform/Canvas/UI (bar=MostFrontArea)
  chat button parented to LeftIcons (children now 6)
  ui built; font=Base_XX screen=2560x1600
  remote models: N (...)
允许缺失：dsh cli:（未装 DSH CLI 时）
```

一次性排查命令：

```powershell
$log = "$G\BepInEx\SatoneHelp\satone-help.log"
Select-String -Path $log -Pattern 'ui built','parented to LeftIcons','remote models' | Select-Object -Last 6
```

`BepInEx\LogOutput.log` 里应无 `Exception`/`NullReferenceException`。

### 3.2 界面

- 底部按钮栏顺序：`故事 / 特别 / 聊天 / 转换心情 / 环境 / 音乐`；「聊天」按钮图标与本体的「聊天记录」按钮同族（旋转 90°），悬停时出现白化 + 青/品红故障环 + 1.05 倍放大 + 本体提示框「聊天」。
- 点击后出现深色直角窗口：标题左「聊天」、右侧「已连接/未连接」+ 齿轮 + ×，中间消息区，底部输入框 + 📄 + ↑ + 六点缩放。
- 沉浸模式（UI 自动隐藏）下按 **F8** 可开合窗口。

### 3.3 功能冒烟

```
1) 设置齿轮 → 面板出现（接口地址/API Key/模型/随机消息 关·开 + 频率滑条 + 四个动作键）
2) 点「测试连接」→ 状态行出现成功或失败原因（失败不影响其它功能）
3) 输入一句话 → 回车或点 ↑ → 她的回复以多条短气泡依次出现
4) 点 📄 → 系统文件对话框 → 选任意文件 → 复制到 <GAME>\BepInEx\SatoneHelp\workspace\
5) 滚轮在消息区上下滚动（内容超过窗口高度时）
```

---

## 3.5 自动启动 DSH（v1.3 起）

> 术语：**DSH = DeepSeek Harness 桌面客户端**，主程序名 `DeepSeek Harness.exe`。
> 模组拉起的是**这个桌面客户端**（它自带本地网关 `127.0.0.1:8326`），不是 `dsh` 命令行。

可执行文件解析顺序（`SatoneHelp.Dsh.FindExe`）：

1. `config.dshPath`（显式指定，最高优先级）；
2. **正在运行的 `DeepSeek Harness` 进程的真实路径**
   （`Process.GetProcessesByName("DeepSeek Harness")` → `MainModule.FileName`；最可靠，装到哪都能找到）；
3. 常见安装位置（按序取第一个存在的）：
   `%LOCALAPPDATA%\Programs\DeepSeek Harness\DeepSeek Harness.exe`、
   `%LOCALAPPDATA%\Programs\deepseek-harness\DeepSeek Harness.exe`、
   `%PROGRAMFILES%\DeepSeek Harness\DeepSeek Harness.exe`、
   `%PROGRAMFILES(X86)%\DeepSeek Harness\DeepSeek Harness.exe`。

模组加载后约 8 秒检查一次网关（`config.apiBase` 的 host:port，默认 `127.0.0.1:8326`），
并**始终先打印解析结果**：`dsh exe: <path>` 或 `dsh exe: not found`。

- 能连上 → 日志 `dsh gateway already up`，不启动任何进程。
- 连不上且 `autoStartDsh=true`（默认）→ 用 `Process.Start(path, UseShellExecute=true, WorkingDirectory=exe 目录)` 拉起，
  日志 `dsh launched: <path>`。
- `autoStartDsh=false` → 日志 `dsh auto-start disabled`。
- 都找不到 → 日志 `dsh not found; set config.dshPath or start DSH manually`（不报错、不阻塞游戏）。

agent 侧判断依据（grep 关键字）：`dsh exe: ` / `dsh gateway already up` / `dsh launched: ` /
`dsh not found` / `dsh auto-start disabled` / `dsh launch failed`。

本机实测（可作为期望值参考）：

```
10:12:32.596 dsh exe: C:\Users\Administrator\AppData\Local\Programs\DeepSeek Harness\DeepSeek Harness.exe
10:12:32.611 dsh gateway already up
```

## 4. 配置（config.json）

路径：`<GAME>\BepInEx\SatoneHelp\config.json`，首次启动自动生成，界面改动会写回。

| 字段 | 默认 | 说明 |
|---|---|---|
| `apiBase` | `http://127.0.0.1:8326` | OpenAI 兼容根地址；会自动补 `/v1/chat/completions` |
| `apiKey` | 空 → 自动读 `%USERPROFILE%\.dsh\openai-gateway\api-key` | 直连官方时填 `sk-...` |
| `model` | `deepseek-account/deepseek-flash` | 点界面「模型」循环切换，列表来自 `GET {apiBase}/v1/models` |
| `randomEnabled` | `false` | 随机消息开关 |
| `randomMinutes` | `5` | 0.5–30 |
| `playerName` | `你` | 称呼 |
| `autoStartDsh` | `true` | 游戏启动时网关不通则拉起 DSH |
| `dshPath` | `""` | DSH 主程序路径；留空自动查找 |
| `toolsEnabled` | `true` | 是否允许 `list_dir/read_file/write_file/run_cmd` |

改 `config.json` 后**重启游戏**生效（界面上的保存按钮会立即生效）。
直连 DeepSeek 官方：`apiBase=https://api.deepseek.com`、`model=deepseek-chat`、填自己的 key。

---

## 5. 数据与安全边界

```
<GAME>\BepInEx\SatoneHelp\workspace\   工具的唯一可写目录（write_file 越界会被拒绝）
history.json      当前会话（下线时被提炼成记忆后删除）
memory.md         长期记忆（≤1500 字，注入 system prompt）
birthday.txt      当日是否已祝福
satone-help.log   运行日志
```

- 插件**只读**两处外部资源：`%USERPROFILE%\.dsh\openai-gateway\api-key`、`%USERPROFILE%\.dsh\our-free-model\catalog.json`；不写。
- `run_cmd` 走 `cmd.exe /c`，工作目录是 `workspace\`，超时 15 秒，输出截断 4000 字符。
- 不触碰游戏存档（`%LOCALAPPDATA%Low\nestopi\Chill With You\SaveData\`）——生日是**只读反射**读内存里的 `ProfileData`。

---

## 6. 回滚

```powershell
# 完全回滚：删三项即可（游戏本体文件从未被修改）
Remove-Item "$G\BepInEx" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$G\winhttp.dll","$G\doorstop_config.ini" -Force -ErrorAction SilentlyContinue

# 只停用模组（保留 BepInEx 与数据）
Move-Item "$G\BepInEx\plugins\SatoneHelp" "$G\BepInEx\plugins\SatoneHelp.disabled" -Force

# 只清数据
Remove-Item "$G\BepInEx\SatoneHelp" -Recurse -Force
```

回滚验证：`Test-Path "$G\BepInEx"` 为 `False`，且 `Get-ChildItem $G` 与安装前记下的清单一致。

---

## 7. 故障对照表

| 现象 | 判定 | 处理 |
|---|---|---|
| 日志只有 `awake ok` 没有 `ui built` | 卡在场景加载/游戏自杀 | 再启动一次；仍失败查 `Player.log` |
| 日志停在 `AssetLoad` | 游戏双进程启动的第一次 | 正常，等第二个进程 |
| 有 `ui built` 但没按钮 | 被「隐藏 UI」或沉浸模式隐藏 | 按 F8；或点一下游戏画面唤回 UI |
| 窗口打开但「未连接」 | 网关未启动/key 失效 | 设置里「测试连接」，检查 DSH 是否在跑 |
| 点按钮无反应 | 输入未聚焦（窗口未激活） | 先点一下游戏窗口再点按钮 |
| Steam 卡 `CheckShaderDepotManifest` | Steam 缓存异常 | 见 §2 |
| 中文显示成方块 | 字体未取到 | 日志搜 `font=`，应为 `Base_SC` 或 `Base_JP` |

---

## 8. 卸载后残留检查

```
不应残留： <GAME>\BepInEx  <GAME>\winhttp.dll  <GAME>\doorstop_config.ini
可保留：   <GAME>\BepInEx\SatoneHelp\（若想留记忆/皮肤，先备份再删 BepInEx）
```
