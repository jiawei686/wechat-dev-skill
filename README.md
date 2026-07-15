# wechat-dev-skill

微信开发者工具调试 Skill —— 把"每次调试先扫 build / devtools / output / networks 四个来源有没有报错"固化成一句话流程，自动汇总报错清单并给出修复方向。

它**依赖** [`wechat-dev-mcp`](https://github.com/jiawei686/wechat-dev-mcp)：MCP 负责"动手连模拟器"，Skill 负责"按流程排查问题"。两个一起装上，调试时对 AI 说「调试一下这个项目」就行。

---

## 它解决什么

调微信小程序 / 小游戏，报错散在四个地方，人工逐个翻很费时间：

- **build**：编译 / 语法 / 依赖错误
- **devtools**：运行时控制台报错、异常堆栈
- **output**：构建产物、`wxss` 编译校验
- **networks**：请求失败、接口超时、域名校验

装上本 Skill 后，AI 会一次性把这四个来源扫完、汇总、定位，并直接调 MCP 去连模拟器、重开项目、截图核对。

---

## 安装（2 步）

1. 先装好 [`wechat-dev-mcp`](https://github.com/jiawei686/wechat-dev-mcp) 并在你的客户端（Cursor / Claude Code / Codex / WorkBuddy 等）里连上它。
2. 把本 Skill 放到你所用客户端的 skills 目录（各客户端位置不同，例如 `~/.workbuddy/skills/`、项目内 `.workbuddy/skills/`，或 Cursor / Claude Code 等的 skills 目录）。确保 `SKILL.md` 与 `README.md` 同目录，重新加载客户端即可。

---

## 怎么用

直接对 AI 说：

> 「调试一下 `/Users/你/项目` 这个小程序，把报错都找出来」

AI 会按标准动作扫四类报错 → 汇总清单 → 给出修复建议；改完后重开项目、必要时在模拟器上点一下做功能验证。

---

## 文件

- `SKILL.md` —— Skill 本体（工作流、工具映射、坐标说明）
- `README.md` —— 本说明
