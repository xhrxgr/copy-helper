# AGENTS.md — 抄写助手项目上下文

> 本文件用于跨对话保留上下文，方便切换对话时快速恢复工作状态。修改代码后请同步更新此文件。

## 项目概览

- **名称**：抄写助手 (copy-helper)
- **源仓库**：https://github.com/xhrxgr/copy-helper
- **本地路径**：`e:\工程开发\抄写助手`
- **在线体验**：https://xhrxgr.github.io/copy-helper/抄写助手.html
- **作者意图**：高二学生为抄写长作文不串行而做的工具
- **技术栈**：纯 HTML + CSS + JavaScript，单文件，无框架无依赖
- **主文件**：`抄写助手.html`（约 1776 行，HTML/CSS/JS 全在一个文件）

## 核心功能

- 自定义每行字数（5~100），字号自动适配屏幕宽度
- 阅读模式：左半屏点击=上一句，右半屏=下一句；键盘 ← → 空格切换
- 段落提示：进入新段落时绿色提示
- 进度条：可拖动，绿色标记显示段落起始位置
- 深色主题，横竖屏自适应，支持移动端安全区域

## 项目结构

```
抄写助手/
├── 抄写助手.html   # 主程序（唯一代码文件）
├── README.md       # 项目说明
├── LICENSE         # MIT
└── AGENTS.md       # 本文件
```

## 关键全局变量（脚本顶层 let 声明）

| 变量 | 类型 | 含义 |
|---|---|---|
| `lines` | string[] | 按字数切分后的所有行 |
| `paragraphMap` | number[] | 每行对应的段落索引（与 lines 等长） |
| `currentIndex` | number | 当前阅读到的行索引 |
| `currentParagraph` | number | 当前段落索引 |
| `isReading` | boolean | 是否在阅读模式 |
| `showControls` | boolean | 是否显示底部按钮 |
| `charPerLine` | number | 每行字数（5-100） |
| `currentFontSize` | number | 当前字号（px） |
| `interactionsInitialized` | boolean | 监听器是否已绑定（防重复） |
| `touchState` / `dragState` / `mouseState` | object | 交互状态机 |

## 关键函数

- `splitText()` — 按字数切分文本，生成 lines/paragraphMap
- `calculateOptimalFontSize()` — 二分查找最优字号
- `startReading(fromRestart=false)` — 进入阅读模式；fromRestart=true 强制从头
- `restartReading()` — 重置到第 1 行并开始
- `backToEdit()` — 返回编辑模式
- `updateDisplay()` — 刷新阅读视图（当前行/前后行/进度条/段落标签）
- `prevLine()` / `nextLine()` — 翻行
- `getTotalParas()` — 安全获取总段落数（避免 Math.max(...[]) 返回 -Infinity）
- `updateProgressUI(p)` / `updateProgressFromPercentage(p)` — 进度条 UI 与位置同步

## 记忆功能（2026-07-25 新增）

- **存储 key**：`copyHelper:memory:v1`（localStorage）
- **保存内容**：text、charPerLine、lineHeight、showControls、currentIndex、currentParagraph、isReading、savedAt
- **保存时机**：输入(防抖500ms)、调整字数、翻行、拖动进度条、切换按钮、返回编辑、beforeunload
- **恢复时机**：页面加载时 `applyMemory(loadMemory())`
- **关键函数**：
  - `scheduleSaveMemory()` — 防抖保存
  - `saveMemory()` — 同步保存到 localStorage
  - `loadMemory()` — 读取并 JSON.parse
  - `applyMemory(memory)` — 恢复文本、设置、位置（不自动进入阅读模式）
  - `clearMemory()` — 清除（带 confirm 确认）
  - `updateStartButton()` — 根据 currentIndex 切换"开始"/"继续 (第 X 行)"按钮
  - `updateMemoryInfo()` — 显示"↻ 已记忆 MM/DD HH:MM · 第 X 行"绿色提示
- **UI 元素**：
  - `#startBtn` 文案会变成 `继续 (第 X 行)`（当 currentIndex>0）
  - `#restartBtn`（从头开始）在 currentIndex>0 时显示
  - `#clearMemoryBtn`（🗑️ 清除记忆）常驻设置面板
  - `#memoryInfo` 在输入区下方显示记忆提示

## 已修复的 Bug（2026-07-25）

1. **空数组 Math.max** — `Math.max(...paragraphMap)` 在空数组返回 -Infinity；新增 `getTotalParas()` 用循环替代
2. **innerHTML XSS** — `currentLine.innerHTML = '<span>...' + lines[i]` 会把 `<` `&` 当 HTML；改为 `createElement` + `textContent`
3. **Windows 换行符** — `splitText` 未处理 `\r\n`/`\r`；新增 `raw.replace(/\r\n/g,'\n').replace(/\r/g,'\n')`
4. **进度条越界** — `Math.floor(percentage * lines.length)` 在 p=1.0 时得 lines.length；改为 `Math.min(lines.length-1, ...)`
5. **重复监听器** — `initClickAreas`/`initProgressBar` 每次进阅读模式都绑定；新增 `interactionsInitialized` 守卫
6. **adjustCharCount 越界** — 增大字数后 lines 减少但 currentIndex 未变；`splitText` 末尾加 `if (currentIndex >= lines.length) currentIndex = lines.length-1`

## 测试方法

- **快速验证**：浏览器打开 `抄写助手.html`，输入文本→开始→翻行→返回→刷新→确认恢复
- **回归测试**：曾用 Node + vm + 手写 DOM mock 跑过 15 项测试（测试文件已删除，需要时可重新生成）
- **本地服务器**：`python -m http.server 8765` 然后访问 `http://localhost:8765/抄写助手.html`

## Git 工作流

- 主分支：`main`（来自上游）
- 修改应在本地提交，PR 回上游需用户决定
- 提交前确认 `抄写助手.html` 单文件可独立运行

## 待办 / 已知限制

- 记忆功能仅保存单个文档（无历史列表）；如需多文档切换可扩展为 `copyHelper:history:v1` 数组
- 无导入/导出功能（如需分享文本，用户需手动复制）
- 无字体大小手动调节（仅自动适配）；如需可加滑块
- 阅读模式无自动滚动/翻页定时器；如需可加 setInterval

## 开发约定

- 代码注释用中文
- 单文件架构，不拆分 CSS/JS（保持双击即开的特性）
- 修改后需同步更新本文件

## 提交记录

- `08da962` 2026-07-25 — Add memory feature and fix 6 bugs（已推送 origin/main，Pages 已生效）
- 下次：在 README 增加"更新日志"段落，说明本次新增的记忆功能（待提交）
