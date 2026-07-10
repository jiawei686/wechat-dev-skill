---
name: wechat-dev-skill
description: >
  微信开发者工具（WeChat DevTools MCP）配套调试 skill。把"每次调试先扫 build / devtools / output / networks 四个来源有没有报错"
  固化成一条命令式流程，自动汇总报错清单并给出修复方向。
  Trigger phrases: 调试小程序, 检查报错, debug wechat, build报错, 网络报错, 微信开发者工具检查,
  扫一遍报错, wx-debug, 跑一下debug loop, 改完代码复查。
agent_created: true
---

# WeChat Dev Skill（wechat-devtools MCP 配套调试）

本 skill 是 `wechat-devtools` MCP 的"例行巡检"封装：一次性扫完四个错误来源，输出一份简报，
省去每次手调 4 个工具。**MCP 提供原子工具（肌肉），本 skill 提供固定流程（routine）——两者配合使用。**

## 与 MCP 的关系 & 如何配合

| | 仓库 | 角色 | 你得到什么 |
|---|---|---|---|
| **wechat-dev-mcp** | [wechat-dev-mcp/README.md](../../wechat-dev-mcp/README.md) | 原子工具层 | `check_health` / `game_touch` / `game_tap_grid` / `game_canvas_screenshot` / `restart_project` … 40+ 个可被 AI 调用的工具（"肌肉"） |
| **wechat-dev-skill**（本文件） | `~/.workbuddy/skills/wechat-dev-skill/` | 例行巡检流程 | 一条命令跑完「健康 → 编译 → 控制台 → 网络 → 截图 → 功能测试」并出简报（"routine"） |

**配合使用方式**：
1. 先装好 MCP 连接器（见 MCP 的 README「🚀 快速开始」：npx / npm i -g，再在客户端里配 `mcpServers`）。
2. 在 WorkBuddy 启用本 skill（已放在 `~/.workbuddy/skills/wechat-dev-skill/`）。
3. 调试时对 AI 说"跑一下微信调试巡检 / 检查报错 / debug 一下小程序"，AI 会自动调 MCP 工具、按下面流程执行。

> 一句话：MCP 让你"能调"，skill 让你"知道按什么顺序调、调完怎么汇总"。**两个都装，调试才顺。**

### 本 skill 每一步调度的 MCP 工具

| 巡检步骤 | 调用的 MCP 工具 |
|---|---|
| ① 健康 / 编译状态 | `check_health` |
| ② 编译报错 | `check_health`（compilationStatus）· 必要时 `build_npm` |
| ③ 控制台报错 | `get_console_logs` |
| ④ 网络报错 | `game_get_network_logs` / `get_network_logs` |
| ⑤ 画面截图 | `game_canvas_screenshot`（小游戏，走 CDP）/ `screenshot`（小程序） |
| ⑥ 功能测试（小游戏） | `game_tap_grid` / `game_touch` / `game_canvas_screenshot` |

## 适用前提

- `wechat-devtools` MCP 已连接且项目已 `launch` / `connect`。
- 若未连接：先用 `launch`（传 `projectPath`）或 `connect`（传 `wsEndpoint` + `projectPath`）。
- **小游戏 CDP-only 模式**：若 automator(9420) 不可用（手动从 UI 打开的工程默认不启用自动化），但 DevTools 以 `--remote-debugging-port=9222`
  启动且游戏 webview 存在，可仅传 `cdpEndpoint`(或让 server 自动发现)调用 `connect`。Server 会自动降级为 CDP-only 连接，足以支撑
  console/network/截图等全部调试能力（evaluate 触控类操作也走 CDP Runtime.evaluate）。
- 若 `check_health` 返回 `compilationStatus: "compiling"`，先 `wait_ready({timeout: 60000})` 再继续。

## 巡检流程（每次调试都按这个顺序跑）

### 1. devtools（运行状态）
调用 `check_health`，确认：
- `connected` 是否为 true
- `projectType`：`program`（小程序）还是 `game`（小游戏）—— 决定后面用哪套工具
- `pageReady` 是否为 true
- `compilationStatus`：若出现 `error` / 非 `compiling` 异常，记录并直接跳到"汇总"。

### 2. build（编译报错）
- 编译错误多数会体现在 `check_health` 的 `compilationStatus` 与控制台日志里。
- 若需单独确认编译产物，可调用 `build_npm`（小程序 npm 构建）并看返回。
- 关键：凡是 `compilationStatus` 非成功，必须在报错清单中标红。

### 3. output（控制台报错）
调用 `get_console_logs(level: "error")`，只取 error 级日志。
- 记录每条的 `level` / `text` / 来源页面（`pagePath` 若有）。
- 若无 error 级日志，明确标注"控制台无 error"。

### 4. networks（网络请求报错）
调用网络日志工具（MCP 中通常名为 `get_network_logs`；若实际名称不同，先用 ToolSearch 查 `wechat-devtools` 确认）。
过滤条件：
- 状态码非 2xx（4xx / 5xx）
- 或请求失败 / 超时 / 被 abort
- 记录 `url` / `status` / `method` / 失败原因。
- 若无失败请求，标注"网络请求全部正常"。

> 注：若运行环境里没有独立的 network 工具，退而求其次——从控制台日志中找 `fail` / `net::ERR` / `request` 相关 error 行，并在简报里注明"网络来源取自控制台推断"。

### 5. 画面截图（视觉确认）
报错扫完，再截一张当前画面做肉眼确认（尤其小游戏 canvas 渲染 / 黑屏问题必须看一眼）。
按 `projectType` 选通道：

- **program（小程序）**：用 MCP 的 `screenshot` 工具（automator 通道，正常可用）。
- **game（小游戏）**：automator 的 `screenshot()` 对小游戏会**挂死**（协议层不支持），必须走 **CDP**：
  - DevTools 须以 `--remote-debugging-port=9222` 启动（否则 `/json` 不暴露）；
  - 从 `http://127.0.0.1:9222/json` 找 url 含 `gamePage.html` 的 webview target，取其 `webSocketDebuggerUrl`；
  - 用 CDP `Page.captureScreenshot` 截图（已端到端验证可用，产出 PNG）。
  - 若 MCP 暴露了 `game_canvas_screenshot` 之类的游戏专属截图工具，优先用它（底层即走上述 CDP）。
  - ⚠️ **禁止对游戏 webview 调用 `Page.reload`**：实测会让游戏运行时崩溃，webview 直接从 `/json` 列表消失（连截图都截不到）。直接 `Page.captureScreenshot` 当前 surface 即可。
  - ✅ **要重开/恢复游戏，用 MCP 的 `restart_project` 工具**（底层 `cli open --auto-test --project <path>`，非阻塞 + `waitGameWebview` 轮询 gamePage.html）。这能干净地重启工程、编译并拉起运行时；崩溃后也能从外部救回。不要用脚本 reload。
- 截图存为本地 PNG，**直接 present 给用户**做视觉确认，并在简报里附路径。
- 若截图失败：在简报标 ⚠️ 并说明原因（如未开 CDP、webview 未找到），不要静默跳过。

### 6. 功能测试（小游戏：真实触控落子 / 交互验证）
扫完报错后，若用户要"验证游戏功能"，用**真实触控**走完整输入链路（不要只靠读状态）：

- 用 `game_tap_grid({ row, col })` 直接以**棋盘格**落子（推荐）：内部把格坐标换算成逻辑像素 `BOARD_X+c*CELL, BOARD_Y+r*CELL`，
  并按 cdp-client.js 的 `touchMap` 仿射变换发 CDP 触控，确保精确命中目标格；`verify:true`（默认）会截图差分确认棋子真的落在 (row,col)。
- 也可用底层 `game_touch({ x, y })` 传逻辑坐标（同样自动套用仿射变换）。坐标换算：从 game.js 取 `BOARD_X/BOARD_Y/CELL`，逻辑 = `BOARD_X+c*CELL, BOARD_Y+r*CELL`。
- 落子后 `game_canvas_screenshot` 截图，用像素分析（如 `verify-fix.mjs` / `board-vision` 思路）确认棋子真的渲染出来（黑棋暗像素增加）。
- 🔴 **测试点必须非对称（行≠列，即 r≠c，且远离画布中心）**：只验证"落子后有棋子出现"不够——**必须验证棋子出现在"点击的那个位置"**。
  - 反例教训：曾用 grid(7,7)/(3,3)/(11,11) 等**对角线对称点**做功能测试，全部"通过"，但漏掉了 `pointToGrid` 行列互换（r/c swapped）的 bug——因为对角线点在互换下坐标不变，恰好蒙对。
  - 正确做法：至少测一个 r≠c 的点（如 grid(2,10)：x=BOARD_X+10·CELL, y=BOARD_Y+2·CELL），截图后**校验棋子的实际像素位置 = 期望像素位置**（而非只看总数增加）。若棋子落在对角线镜像处（转置位置），即为坐标轴/行列映射 bug。
  - 通用原则：验证"输入→输出映射"时，测试输入要**破坏对称性**，否则对称的坐标/参数会掩盖 swap/镜像/转置类 bug。
- ⚠️ **小游戏 `evaluate` 上下文限制**：`game.js` 运行在独立 JS 上下文，CDP `Runtime.evaluate` 能看到注入的 `wx`，
  但**拿不到** `GameGlobal.__gomoku` 等游戏内部状态。所以"读游戏状态 / 调游戏函数"走不通，**功能测试必须靠真实触控 + 截图像素校验**。

> 坐标空间（实测确认，重要）：开发者工具模拟器里 CDP `Input.dispatchTouchEvent` 的 touchPoints
> 用的是「渲染后的手机屏幕像素」，与游戏逻辑坐标之间是**仿射变换**而非 1:1：
>   game 实际收到的逻辑坐标 R = (1/sx)·S − (ox/sx, oy/sx)
> 所以 MCP 必须反算：发送 S = (ox, oy) + (sx, sy)·L（L 为目标逻辑坐标）。该变换已固化在
> cdp-client.js 的 `touchMap`（默认值由本机标定：sx=sy=0.75, ox=37.75, oy=91.0）。
> 若模拟器缩放/窗口位置变了导致落点偏移，用 `set_touch_map` 或 `diagnostic-touch.mjs` 重新标定。
> 直接传裸逻辑坐标会系统性偏移——这正是最初"落子位置不对"的根因之一（另一根是 game.js 的 pointToGrid 行列互换，已修）。

## 输出格式（汇总简报）

跑完四步后，给一份固定结构的简报：

```
【微信 DevTools 巡检】
- devtools:  connected=true | projectType=program | pageReady=true | compile=success
- build:     ✅ 无编译错误 / ❌ 编译失败：<一句话>
- output:    ✅ 无 error / ❌ N 条 error：<逐条列出>
- networks:  ✅ 全部正常 / ❌ M 个失败请求：<url + status>
- screenshot: ✅ 已截图 <path> / ⚠️ 截图失败：<原因>
结论: 无报错可继续 / 需修复以下项（按优先级）
修复建议:
  1. ...
  2. ...
```

- 有报错：按"阻断级（编译/连接失败）> 控制台 error > 网络失败"排序给修复建议。
- 无报错：一句话"四来源均干净，可继续"。

## 改完代码后的 Debug Loop（写/改文件后必跑）

1. `check_health` → 确认无编译错误
2. `get_console_logs(level: "error")` → 抓最新 error
3. 有错 → `edit` / `write` 修 → 回到 1
4. 无错 → 提示用户"可预览/上传"

## 常见坑（来自项目 AGENTS.md，必须规避）

| 坑 | 正确做法 |
|---|---|
| 在小游戏上用页面工具 | `projectType=game` 时只用 `evaluate` / `call_wx_method` / `game_get_info` / `screenshot`，禁用 `get_page_data` / `get_element` / `tap_element` / `navigate_to` |
| 不等编译完就查 | `launch` 后先 `wait_ready` |
| 改完不 `check_health` | 每次写/改文件后必跑 `check_health` |
| `reLaunch` 当 `navigateTo` 用 | `reLaunch` 清空页面栈；保留历史用 `navigateTo` |
| 路径硬编码 | `projectPath` 一律用绝对路径 |

## 云函数调试（如涉及）

`call_cloud_function(name, data)` → `get_console_logs()` → 有错修代码 →
`build_npm()` + `cloud_functions_deploy(env, names)` 部署。
