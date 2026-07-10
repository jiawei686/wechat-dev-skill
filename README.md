# wechat-dev-skill

微信开发者工具（WeChat DevTools）调试 Skill —— 把“每次调试先扫 build / devtools / output / networks 四个来源有没有报错”固化成一条命令式流程，自动汇总报错清单并给出修复方向。

本 Skill **依赖** [`wechat-dev-mcp`](https://github.com/jiawei686/wechat-dev-mcp) 提供的能力（连接 DevTools、截图、落子、重开项目等 MCP 工具）。两者配合使用：MCP 负责“动手连真机/模拟器”，Skill 负责“按流程排查问题”。

## 它解决什么问题

调试微信小程序 / 小游戏时，报错往往分散在四个地方，人工逐个翻很费时间：

- **build**：编译错误、语法错误、依赖问题
- **devtools**：运行时控制台报错、异常堆栈
- **output**：构建产物、`wxss` 编译、`app.json` 校验
- **networks**：请求失败、接口超时、域名校验

本 Skill 把“扫这四个来源 → 汇总 → 定位 → 修复建议”固化成标准动作，并直接调用 MCP 工具去连模拟器、重开项目、截图核对 UI。

## 与 MCP 的关系 & 如何配合

| 你在这个 Skill 里要做的事 | 调用的 MCP 工具（来自 wechat-dev-mcp） |
|---|---|
| 连上正在运行的 DevTools | `connect` |
| 重开 / 切换到指定项目（触发重新编译） | `restart_project` |
| 抓控制台 / 报错日志 | `get_console_logs` / `get_system_info` |
| 看当前页面结构、定位元素 | `get_element` / `get_page_stack` / `page_scroll_to` |
| 在模拟器上点某个坐标（小游戏落子等） | `game_tap_grid`（自动做坐标仿射补偿） |
| 截屏核对画面是否对 | `screenshot` |

> 小游戏触控坐标有一个关键的**仿射变换**：模拟器收到的坐标 = `1.3333 × 你发的 − (50.33, 121.33)`，所以 MCP 侧会把逻辑坐标反算成 `0.75 × L + (37.75, 91)` 再下发，无需你手动换算。`game_tap_grid` 已内置这个补偿。

## 安装

1. 先装好 `wechat-dev-mcp` 并在客户端（Cursor / Claude Code / Codex / OpenCode 等）里连上它（配置示例见 MCP 仓库的 `examples/`）。
2. 把本 Skill 放到客户端的 skills 目录，例如：
   - WorkBuddy 用户级：`~/.workbuddy/skills/wechat-dev-skill/`
   - 项目级（任意客户端）：`<项目>/.workbuddy/skills/wechat-dev-skill/`
   - 其它客户端按其自身的 skills 目录放置（Claude Code / Codex / OpenCode / Cursor / Windsurf / Cline / Zed 等）
3. 确保 SKILL.md 与 README.md 在同一目录，重新加载客户端即可。

## 典型调试流程

1. `restart_project` 重开项目，触发干净编译。
2. 用 Skill 的标准动作扫 build / devtools / output / networks 四类报错。
3. 根据汇总清单逐条定位：编译类看 build，运行类看 devtools 控制台，请求类看 networks。
4. 改完后 `restart_project` 重新编译，必要时 `game_tap_grid` + `screenshot` 做功能验证（小游戏推荐用非对称点，如 (2,10)、(10,3) 来抓 r/c 互换类 bug）。

## 文件

- `SKILL.md` —— Skill 本体（工作流、工具映射、坐标说明）
- `README.md` —— 本说明
