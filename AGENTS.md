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
7. **空行被计为段落（2026-07-25 二次修复）** — `splitText` 中空行也 `paraIndex++`，导致段落数虚高、编号不连续（开头空行会让第一段编号从 2 开始）；改为空行直接 `return` 跳过
8. **Unicode 段落分隔符未识别（2026-07-25 二次修复）** — 从 Word/网页复制的文本可能含 `\u2028`（行分隔符）、`\u2029`（段分隔符），`split('\n')` 无法识别导致整段文本被当成 1 段；在规范化换行符时增加 `.replace(/\u2028/g,'\n').replace(/\u2029/g,'\n')`
9. **全屏后系统手势误触翻页（2026-07-25 修复）** — `handleTouchStart`/`handleTouchEnd` 里 `e.preventDefault()` 会吞掉系统手势（下拉状态栏、上滑小白条），且 `.click-area` 覆盖整个屏幕高度（`top:0;height:100%`）导致边缘手势被劫持；修复：(a) `.click-area` 改为 `top:36px;height:calc(100%-72px)` 顶部底部各留 36px 死区；(b) 加 `touch-action:none` 用 CSS 替代 `preventDefault` 阻止默认行为；(c) 去掉 touchstart/touchend 的 `preventDefault`，监听器改 `passive:true`
10. **翻页多翻一页（2026-07-25 修复）** — 去掉 touchend 的 `preventDefault`（bug 9 修复）后，浏览器在 touchend 后会合成 mousedown/mouseup 事件，导致手机上 touchend 翻 1 页 + 合成的 mouse 事件再翻 1 页 = 多翻一页；修复：新增 `lastTouchTime` 全局变量，`handleTouchEnd` 末尾记录 `Date.now()`，`handleMouseDown` 开头检查 `Date.now() - lastTouchTime < 500` 则 return 忽略（因 `mouseState.isDown` 仍为 false，`handleMouseUp` 也会 return）

## 测试方法

- **快速验证**：浏览器打开 `抄写助手.html`，输入文本→开始→翻行→返回→刷新→确认恢复
- **回归测试**：曾用 Node + vm + 手写 DOM mock 跑过 15 项测试（测试文件已删除，需要时可重新生成）
- **本地服务器**：`python -m http.server 8765` 然后访问 `http://localhost:8765/抄写助手.html`

## Git 工作流

- 主分支：`main`（来自上游）
- 修改应在本地提交，PR 回上游需用户决定
- 提交前确认 `抄写助手.html` 单文件可独立运行

## 待办 / 已知限制

- ~~记忆功能仅保存单个文档（无历史列表）~~ → 已实现历史文档列表（2026-07-25）
- ~~无导入/导出功能~~ → 已实现 .txt 导入导出（2026-07-25）
- 无字体大小手动调节（仅自动适配）；如需可加滑块
- 阅读模式无自动滚动/翻页定时器；如需可加 setInterval

## 历史文档列表（2026-07-25 新增）

- **存储 key**：`copyHelper:history:v1`（localStorage，独立于 memory）
- **数据结构**：`[{id, title, text, charPerLine, lineHeight, currentIndex, currentParagraph, savedAt, lastReadAt}]`
- **自动保存**：`saveMemory` 时同步调用 `autoSaveToHistory()`，文本 ≥ 50 字才保存
  - 有 `currentDocId` → 更新对应文档
  - 无 `currentDocId` 但有相同文本 → 复用并更新
  - 全新文本 → 创建新文档，unshift 到列表
- **排序**：按 `lastReadAt` 降序
- **上限**：保留最新 50 篇（HISTORY_MAX）
- **标题**：第一行前 20 字（`makeDocTitle`）
- **UI**：左侧抽屉（`#historyDrawer` + `#drawerOverlay`），从左滑出
- **关键函数**：
  - `loadHistory()` / `saveHistory(history)` — 读写
  - `autoSaveToHistory()` — 自动保存（在 saveMemory 中调用）
  - `loadHistoryDoc(id)` — 加载文档到编辑区（更新 lastReadAt）
  - `deleteHistoryDoc(id)` / `clearAllHistory()` — 删除
  - `renderHistoryList()` — 渲染抽屉列表（带时间格式化"X 分钟前"）
  - `toggleHistoryDrawer(forceState)` — 打开/关闭抽屉
  - `escapeHtml(str)` — 标题转义（防 XSS）
- **memory 中新增字段**：`currentDocId` 跟踪当前文档在历史中的 ID
- **clearMemory 行为**：只清当前 memory，不影响历史列表

## 全屏功能（2026-07-25 新增）

- **API**：Fullscreen API + webkit 前缀兼容
- **按钮**：右上角 `#fullscreenBtn`（⛶ / ⤡）
- **关键函数**：`toggleFullscreen()` + `updateFullscreenBtn()`（监听 fullscreenchange）
- **限制**：Fullscreen API 需用户手势触发（已在 onclick 中），iOS Safari 部分支持

## 导入/导出 .txt（2026-07-25 新增）

- **导出**：`exportTxt()` — Blob + a.download，文件名用第一行前 20 字（过滤非法字符 `\/:*?"<>|`）
- **导入**：`importTxt(event)` — FileReader.readAsText(file, 'UTF-8')，限制 10MB
- **UI**：设置面板内 `⬇ 导出txt` / `⬆ 导入txt` 按钮
- **导入行为**：重置 `currentDocId=null`，让自动保存创建新文档

## UI 改动（2026-07-25）

- 右上角 `top-actions` 按钮组：全屏 ⛶ / 历史 📋 / 设置 ☰（替换原单一 ⚙️）
- 设置按钮动态图标：面板展开 ✕ / 收起 ☰（`toggleSettings` 更新）
- "按钮: 隐藏" → "底部按钮: 隐藏底部按钮 / 显示底部按钮"（消除歧义）
- time-display 位置右移至 `right: calc(9.5rem + safe-right)` 避开 3 按钮

## UI 改动（2026-07-25 二次优化）

- **横屏按钮遮挡修复**：在 `@media (orientation: landscape) and (max-height: 500px)` 中：
  - `.top-actions` 缩小：按钮 40px → 32px，gap 0.5rem → 0.25rem
  - `.time-display` 移到左上角（`left: 1rem`，`right: auto`），避免和按钮组挤在右上角
  - `.settings-panel` 加 `padding-right: 7rem` 让 wrap 出的内容避开按钮组
  - `.settings-group` gap 收紧到 0.5rem 0.8rem
- **"更多 ⋯" 二级菜单**：设置面板里的 导出/导入/清除当前记忆 3 个按钮收进下拉
  - `.more-menu-wrapper` 相对定位，`.more-menu` 绝对定位 top:100%+4px right:0
  - `toggleMoreMenu(forceState)` + 文档级 click 监听点击外部关闭
  - 危险操作"清除当前记忆"用 `.danger` 类（hover 时红色）
  - `toggleSettings` 收起面板时同步关闭更多菜单

## 待办 / 已知限制

- 无字体大小手动调节（仅自动适配）；如需可加滑块
- 阅读模式无自动滚动/翻页定时器；如需可加 setInterval
- 历史文档无手动重命名（仅用第一行前 20 字）；如需可扩展

## 开发约定

- 代码注释用中文
- 单文件架构，不拆分 CSS/JS（保持双击即开的特性）
- 修改后需同步更新本文件

## 提交记录

- `08da962` 2026-07-25 — Add memory feature and fix 6 bugs（已推送 origin/main，Pages 已生效）
- `52f5ccb` 2026-07-25 — docs: add changelog section to README（已推送 origin/main）
- `15ba288` 2026-07-25 — feat: history drawer, import/export txt, fullscreen, UI fixes（已推送 origin/main）
- `9fa0417` 2026-07-25 — fix: landscape button overlap, move import/export into More menu（已推送 origin/main）
- `759b6c9` 2026-07-25 — fix: paragraph split - handle Unicode separators and skip empty lines（已推送 origin/main）
- `27907b6` 2026-07-25 — fix: prevent fullscreen gesture mistouch - edge deadzone + touch-action（已推送 origin/main）
