# AI_CONTEXT.md — SnapDoc 图片转文档 App

> 本文档是整个项目的**唯一权威上下文**，用于指导 AI 完成 App 的设计与开发。
> 任何 AI 在开发本项目前必须完整阅读本文档，并严格遵循其中的设计规范、技术约束与开发流程。
> 当前迭代目标：**iOS App**。但所有架构与代码组织必须满足未来平滑扩展到 **Android 与 Windows**。

---

## 1. 产品概述

### 1.1 一句话定义

SnapDoc 是一款「拍照/上传图片 → AI 识别为 HTML → 流式动态渲染 → 一键导出多种文档格式」的移动端工具 App。

### 1.2 核心用户流程（Happy Path）

```
启动 App
  → 首页（拍照按钮 + 相册上传按钮 + 历史记录）
  → 拍照或选择 1~N 张图片
  → 图片预览确认（可裁剪/旋转/排序/删除）
  → 点击「开始识别」
  → 客户端上传图片，后端调用视觉大模型，以 SSE 流式返回 HTML
  → 客户端在 WebView 中动态渲染逐步到达的 HTML（打字机式增量渲染）
  → 识别完成 → 底部浮出「导出」操作栏
  → 用户选择目标格式：Word / PDF / PPT / Excel / HTML / Markdown
  → 后端基于 HTML + Playwright 实时渲染转换 → 返回文件
  → 客户端保存到本地 / 调用系统分享面板
```

### 1.3 功能清单

| 优先级 | 功能 | 说明 |
|---|---|---|
| P0 | 拍照输入 | 调用系统相机，支持连拍多页 |
| P0 | 相册上传 | 多选图片，支持 jpg/png/heic/webp |
| P0 | AI 识别为 HTML | 后端视觉大模型 OCR + 版面还原，输出语义化 HTML |
| P0 | 流式渲染 | SSE 流式下发 HTML，WebView 增量动态渲染 |
| P0 | 导出 PDF / HTML | Playwright 渲染 HTML → `page.pdf()`；HTML 直接落盘 |
| P0 | 导出 Word (.docx) | HTML → docx 转换管线（见 §5.4） |
| P1 | 导出 Excel (.xlsx) | 提取 HTML 中表格 → xlsx |
| P1 | 导出 PPT (.pptx) | HTML 分节 → Playwright 截图/结构化转换 → pptx |
| P1 | 历史记录 | 本地保存识别任务与结果，可重新导出 |
| P1 | 图片预处理 | 裁剪、旋转、多页排序 |
| P2 | 结果编辑 | 在 WebView 内对识别结果做轻量编辑（contenteditable） |
| P2 | 多语言 OCR | 中英混排优先，其他语言跟随模型能力 |

### 1.4 非功能要求

- **流畅**：识别中 UI 不卡顿，流式渲染帧率稳定；所有页面转场使用系统级动画。
- **简洁**：单手可完成全流程；首页到导出不超过 4 次点击。
- **可离线降级**：无网络时可拍照暂存，联网后继续识别。
- **隐私**：图片仅用于本次识别，后端处理完成后按 TTL 清理（默认 24h）。

---

## 2. 界面设计思路

### 2.1 设计风格来源

设计语言取自 [awesome-design-md 的 Notion 设计体系](https://github.com/VoltAgent/awesome-design-md/tree/main/design-md/notion)。
选择理由：Notion 的「文档工具」气质与本产品高度一致 —— 白纸质感、克制的编辑几何（矩形按钮 + 12px 圆角卡片）、温暖的炭灰文字、柔和的 pastel 点缀色，天然传达"你的图片正在变成一份干净的文档"。

### 2.2 设计令牌（Design Tokens）— 全局唯一来源

所有 UI 代码**必须**引用以下令牌，禁止硬编码颜色/字号/圆角。跨平台迁移时只需移植这一份令牌表。

#### 颜色

```yaml
colors:
  # 品牌与主操作
  primary:          "#5645d4"   # 签名紫，仅用于主 CTA（开始识别 / 导出）
  primary-pressed:  "#4534b3"
  on-primary:       "#ffffff"
  link-blue:        "#0075de"   # 行内链接，不与 primary 混用

  # 表面
  canvas:           "#ffffff"   # 页面背景 / 卡片
  surface:          "#f6f5f4"   # 分区底色 / 输入框静默态
  surface-soft:     "#fafaf9"
  hairline:         "#e5e3df"   # 1px 分隔线 / 卡片描边
  hairline-strong:  "#c8c4be"

  # 文字（Notion 暖炭灰体系）
  ink:              "#1a1a1a"   # 主文字
  charcoal:         "#37352f"   # 正文强调
  slate:            "#5d5b54"   # 次级文字
  steel:            "#787671"   # 三级文字 / 说明
  muted:            "#bbb8b1"   # 禁用 / 占位

  # Pastel 点缀（用于格式徽标、卡片提示）
  tint-peach:       "#ffe8d4"   # PPT
  tint-rose:        "#fde0ec"   # PDF
  tint-mint:        "#d9f3e1"   # Excel
  tint-lavender:    "#e6e0f5"   # Word
  tint-sky:         "#dcecfa"   # HTML
  tint-yellow:      "#fef7d6"   # Markdown / 提示条

  # 语义
  success:          "#1aae39"
  warning:          "#dd5b00"
  error:            "#e03131"

  # 深色场景（相机取景页）
  brand-navy:       "#0a1530"
  on-dark:          "#ffffff"
  on-dark-muted:    "#a4a097"
```

#### 字体与排版

- 字体：**Inter**（Notion Sans 的开源基底），中文回退 `PingFang SC` / 系统默认。
- 移动端字阶（由 Notion 桌面字阶按移动端缩放）：

| Token | 大小/字重/行高 | 用途 |
|---|---|---|
| `display`   | 28px / 600 / 1.2, 字距 -0.5px | 首页大标题 |
| `heading-1` | 22px / 600 / 1.3 | 页面标题 |
| `heading-2` | 18px / 600 / 1.4 | 卡片标题、分区标题 |
| `body`      | 16px / 400 / 1.55 | 正文 |
| `body-medium` | 16px / 500 / 1.55 | 强调正文、列表项标题 |
| `body-sm`   | 14px / 400 / 1.5 | 次级正文 |
| `button`    | 15px / 500 / 1.3 | 按钮文字 |
| `caption`   | 13px / 400 / 1.4 | 说明文字 |
| `caption-bold` | 13px / 600 / 1.4 | 徽标文字 |
| `micro`     | 11px / 600 / 1.4, 字距 +1px, 大写 | 格式标签（DOCX/PDF…） |

#### 圆角与间距

```yaml
rounded: { xs: 4, sm: 6, md: 8, lg: 12, xl: 16, full: 9999 }   # px
# 规则：按钮/输入框 = md(8)，卡片/底部面板 = lg(12)，徽标/胶囊标签 = full
# Notion 几何是"编辑级克制"：按钮是矩形圆角，绝不使用胶囊按钮

spacing: { xxs: 4, xs: 8, sm: 12, md: 16, lg: 20, xl: 24, xxl: 32, section: 48 }  # px
# 基准单位 4px，页面左右安全边距 = md(16)
```

#### 阴影层级

| 层级 | 值 | 用途 |
|---|---|---|
| 0 | 无阴影 + hairline 描边 | 默认卡片 |
| 1 | `rgba(15,15,15,0.04) 0 1px 2px` | 轻浮起 |
| 2 | `rgba(15,15,15,0.08) 0 4px 12px` | 功能卡片、导出面板 |
| 3 | `rgba(15,15,15,0.20) 0 24px 48px -8px` | 识别结果"纸张"卡片（签名视觉） |
| 4 | `rgba(15,15,15,0.16) 0 16px 48px -8px` | 弹窗、底部抽屉 |

#### 动效

- 时长 150–200ms，`ease-out`；页面转场跟随平台默认。
- 流式渲染新内容进入：透明度 0→1 + 4px 上移，120ms。

### 2.3 页面设计（共 5 个核心页面）

#### P1 首页 `HomePage`
- `canvas` 白底。顶部 `display` 标题「图片，变成文档。」+ `steel` 副标题。
- 中部主入口卡片（阴影层级 2，`rounded.lg`）：大号「拍照识别」primary 按钮 + 「从相册选择」secondary 描边按钮。
- 下方「支持导出」pastel 徽标横排：Word(lavender)、PDF(rose)、PPT(peach)、Excel(mint)、HTML(sky)、MD(yellow)，使用 `micro` 大写标签。
- 底部「最近记录」列表：`card-base`（hairline 描边、无阴影），缩略图 + 标题 + 时间 + 格式徽标。

#### P2 相机页 `CapturePage`
- 全屏取景，`brand-navy` 蒙层控件区，`on-dark` 文字。
- 底部：快门（白色圆形 72px）、已拍缩略图堆叠（右下角，带计数徽标）、「完成」按钮。
- 支持连拍多页；辅助线开关。

#### P3 图片确认页 `ReviewPage`
- 横向大图预览 + 底部缩略图条（可拖拽排序、左滑删除）。
- 工具条：裁剪 / 旋转。
- 底部固定 primary 按钮「开始识别（N 张）」。

#### P4 识别页 `RecognitionPage`（核心页面）
- 结构：顶部薄导航（返回 + 任务标题）→ 中部「纸张」卡片 → 底部操作区。
- 「纸张」卡片：白色 `canvas`、`rounded.lg`、**阴影层级 3**，内嵌 WebView，模拟一张正在被书写的纸 —— 本产品的签名视觉。
- 识别中：卡片顶部细进度条（primary 紫，不确定进度动画）+ HTML 增量流入渲染 + 尾部闪烁光标块。
- 识别完成：进度条淡出，底部浮起（4 层阴影、`rounded.lg` 顶部圆角）**导出操作栏**：六个格式按钮（pastel 底色徽标 + `micro` 标签），点击即触发导出。
- 导出中：对应按钮内嵌 spinner；完成后弹系统分享面板。
- 失败态：`error` 色内联提示 + 「重试」ghost 按钮。

#### P5 历史详情页 `HistoryDetailPage`
- 复用 P4 的纸张卡片（直接加载完整 HTML，无流式）+ 导出操作栏。

### 2.4 设计 Do / Don't（继承 Notion 规范）

- ✅ primary 紫**只**用于主 CTA；✅ 按钮 8px 矩形圆角；✅ 卡片 12px 圆角；✅ pastel 色只做点缀底色。
- ❌ 不用胶囊按钮；❌ 不把紫色用于正文或大面积背景；❌ 平铺文档卡片不加重阴影；❌ link-blue 与 primary 不混用。

---

## 3. 技术方案

### 3.1 跨平台选型：**Flutter**（已定，不再讨论）

| 候选 | 结论 | 理由 |
|---|---|---|
| **Flutter** | ✅ 采用 | iOS/Android/Windows 三端官方稳定支持；自绘 UI 保证三端像素一致，设计令牌一次实现；`camera`/`webview_flutter` 生态成熟；Dart 单语言 |
| React Native | ❌ | Windows 端（react-native-windows）维护投入大、生态弱 |
| Kotlin Multiplatform | ❌ | UI 层（Compose Multiplatform for iOS）尚不够稳，Windows 桌面弱 |
| SwiftUI 原生 | ❌ | 无法复用到 Android/Windows，违背核心诉求 |

### 3.2 总体架构

```
┌────────────────────────── Flutter App (iOS 首发) ──────────────────────────┐
│  presentation/  页面与组件（依赖 design tokens）                             │
│  application/   状态管理 Riverpod（任务状态机、流式缓冲）                     │
│  domain/        实体与用例（纯 Dart，零平台依赖）                            │
│  infrastructure/ API客户端(dio+SSE) / 本地存储(drift) / 平台服务(相机,文件,分享)│
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ HTTPS / SSE
┌──────────────────────────────▼──────────────────────────────────────────┐
│                    Backend: Python FastAPI (async)                       │
│  /recognitions   接收图片 → 调用视觉大模型 → SSE 流式返回 HTML 分片          │
│  /exports        HTML → Playwright(chromium) 实时渲染 → 各格式转换 → 文件    │
│  storage         对象存储(本地磁盘/S3兼容)，任务与文件 TTL 24h               │
│  依赖：playwright, python-docx, openpyxl, python-pptx, beautifulsoup4     │
└──────────────────────────────────────────────────────────────────────────┘
```

**架构铁律（跨平台的关键）**：
1. `domain/` 与 `application/` 层禁止 import 任何 `dart:io` 平台分支或插件；平台能力全部经 `infrastructure/` 的抽象接口注入。
2. 平台差异（相机、文件保存、分享）定义为接口 `CameraService` / `FileSaver` / `ShareService`，iOS 先实现，Android/Windows 后续各自实现即可，上层零改动。
3. 设计令牌集中在 `presentation/theme/tokens.dart` 单文件，是 §2.2 的唯一代码化载体。
4. 所有布局使用自适应约束（`LayoutBuilder` + 断点），为 Windows 宽屏预留：≥840px 时首页/识别页转为双栏。

### 3.3 客户端关键实现

#### 状态机（识别任务）

```
idle → picking → reviewing → uploading → streaming → completed
                                  │           │
                                  └── error ←─┘   （error 可 retry → uploading）
completed → exporting(format) → exported / exportError
```

用 Riverpod `AsyncNotifier` 承载；`streaming` 状态持有 `htmlBuffer`（StringBuffer）。

#### 流式渲染（核心难点，必须按此实现）

1. WebView（`webview_flutter`，iOS 为 WKWebView）**一次性**加载本地壳页面 `stream_shell.html`：内含基础排版 CSS（Inter 字体、Notion 风格的 h1~h6/table/blockquote 样式）与一个 `appendChunk(html)` JS 函数。
2. Dart 侧收到 SSE 分片后，**节流合并**（每 80ms 或累计 2KB 触发一次），调用 `runJavaScript("appendChunk(...)")` 注入。
3. `appendChunk` 内部维护**未闭合标签容错解析**：分片可能截断在标签中间，用增量 DOM 补丁（保留待定尾巴字符串，下次拼接后再解析）保证渲染永不破碎。
4. 自动滚动跟随最新内容；用户手动上滑后暂停跟随，出现「回到底部」浮钮。
5. 完成事件（SSE `event: done`）后移除光标、通知 Dart 层切换到 `completed`。

#### 本地存储

- `drift`（SQLite）：任务表 `tasks(id, title, createdAt, status, htmlPath, thumbPath)`。
- HTML 全文与缩略图存应用文档目录，DB 只存路径。

### 3.4 后端关键实现

#### 识别（图片 → HTML）

- 视觉大模型（如 GPT-4o / Qwen-VL / Gemini，通过环境变量 `VISION_MODEL_*` 配置，实现 `VisionProvider` 接口可替换）。
- Prompt 约束模型输出**纯 HTML 片段**（无 markdown 围栏、无 `<html>/<head>` 外壳）：语义化标签、表格用 `<table>`、公式用行内文本、保持原文版面顺序；多图按顺序拼接为多个 `<section data-page="n">`。
- 模型 token 流 → 直接透传为 SSE `event: chunk`，实现真流式。

#### 导出（HTML → 各格式，基于 Playwright 实时渲染）

后端维护一个 Playwright chromium 常驻实例（浏览器上下文池，并发≤4）。导出时将「完整 HTML + 打印级 CSS 模板」组装为完整页面：

| 格式 | 管线 |
|---|---|
| **PDF** | Playwright `page.pdf(format='A4', print_background=True)` —— 像素级还原 |
| **HTML** | 内联 CSS 后直接输出单文件 `.html` |
| **Word** | BeautifulSoup 解析 HTML 语义树 → `python-docx` 映射（h1~h6→标题样式、table→表格、img→内嵌图）；复杂不可映射块降级为 Playwright 局部截图插入 |
| **Excel** | 提取所有 `<table>` → `openpyxl`，每个表格一个 sheet；无表格时整页文字按行入 A 列 |
| **PPT** | 按 `<section data-page>`/`<h1|h2>` 切片 → 每片由 Playwright 定宽 1280×720 截图 → `python-pptx` 整页贴图（v1 策略，v2 再做结构化文本框） |
| **Markdown** | HTML → md（`markdownify`） |

导出为异步任务：`POST /exports` 立即返回 `exportId`，客户端轮询 `GET /exports/{id}`（或复用 SSE），完成后经 `GET /exports/{id}/file` 下载。

### 3.5 API 契约（v1，客户端后端共同遵守）

```
POST /v1/recognitions
  multipart/form-data: images[]（按页序）
  → 201 { "taskId": "uuid" }

GET /v1/recognitions/{taskId}/stream          (SSE)
  event: chunk   data: {"html": "<h1>标题</h1><p>..."}
  event: done    data: {"fullHtmlUrl": "/v1/recognitions/{taskId}/html"}
  event: error   data: {"code": "MODEL_ERROR", "message": "..."}

GET /v1/recognitions/{taskId}/html            → text/html（完整结果）

POST /v1/exports
  json: { "taskId": "uuid", "format": "pdf|docx|pptx|xlsx|html|md" }
  → 202 { "exportId": "uuid" }

GET /v1/exports/{exportId}
  → { "status": "processing|done|failed", "fileUrl": "...", "error": null }

GET /v1/exports/{exportId}/file               → 二进制文件（Content-Disposition）
```

错误码规范：`IMAGE_TOO_LARGE`(413) / `UNSUPPORTED_FORMAT`(422) / `MODEL_ERROR`(502) / `EXPORT_TIMEOUT`(504)。客户端所有错误 UI 文案从错误码映射，禁止直显后端 message。

### 3.6 工程目录

```
snapdoc/
├── AI_CONTEXT.md                # 本文档
├── app/                         # Flutter 工程
│   ├── lib/
│   │   ├── main.dart
│   │   ├── presentation/
│   │   │   ├── theme/tokens.dart        # §2.2 设计令牌唯一代码载体
│   │   │   ├── pages/{home,capture,review,recognition,history_detail}/
│   │   │   └── widgets/                 # PaperCard, FormatBadge, ExportBar...
│   │   ├── application/                 # Riverpod providers / 状态机
│   │   ├── domain/                      # 实体 + 用例（纯 Dart）
│   │   └── infrastructure/
│   │       ├── api/                     # dio + SSE 客户端
│   │       ├── db/                      # drift
│   │       └── platform/                # CameraService/FileSaver/ShareService 接口+iOS实现
│   ├── assets/stream_shell.html         # 流式渲染壳页面（含文档 CSS）
│   └── test/
├── server/                      # FastAPI 后端
│   ├── app/
│   │   ├── main.py
│   │   ├── api/{recognitions,exports}.py
│   │   ├── services/{vision_provider.py, playwright_pool.py, exporters/}
│   │   └── core/{config.py, storage.py}
│   ├── templates/print.css              # 导出用打印级 CSS
│   └── tests/
└── docs/PROGRESS.md             # 开发进度快照（AI 每完成一项必须更新）
```

---

## 4. 开发进程设计

> 执行规则：按 Phase 顺序开发；每个 Phase 有明确验收标准（DoD），**未达 DoD 不得进入下一 Phase**；
> 每完成一个任务，在 `docs/PROGRESS.md` 勾选并记录关键决策，保证任意 AI 可随时接手。

### Phase 0 — 工程脚手架（0.5 天）
- [ ] 初始化 Flutter 工程（iOS target ≥ 15），配置 Riverpod / dio / drift / webview_flutter / camera / image_picker / share_plus。
- [ ] 初始化 FastAPI 工程，安装 playwright 并 `playwright install chromium`。
- [ ] 落地 `tokens.dart`（完整实现 §2.2 全部令牌）+ 全局 Theme。
- **DoD**：两端可空跑；`tokens.dart` 单测校验令牌值与本文档一致。

### Phase 1 — 后端识别与流式链路（2 天）
- [ ] `VisionProvider` 抽象 + 一个真实模型接入 + 一个 `MockVisionProvider`（固定 HTML 按块延时下发，供前端联调）。
- [ ] `POST /recognitions` + SSE stream 端点 + 任务/文件 TTL 存储。
- **DoD**：`curl` 上传图片能收到流式 HTML 分片与 done 事件；单测覆盖分片与错误事件。

### Phase 2 — 客户端主流程 UI（3 天）
- [ ] P1 首页、P2 相机页、P3 确认页（含裁剪/旋转/排序）。
- [ ] P4 识别页：stream_shell.html + appendChunk 增量渲染 + 未闭合标签容错 + 自动滚动。
- [ ] 状态机与错误/重试态。
- **DoD**：对接 Mock Provider 跑通「拍照→流式渲染→完成」全流程，无掉帧、无破碎渲染。

### Phase 3 — 导出管线（3 天）
- [ ] Playwright 浏览器池 + `print.css`。
- [ ] 按序实现导出器：PDF → HTML → DOCX → XLSX → MD → PPTX（每个导出器独立文件 + 独立单测）。
- [ ] 客户端导出操作栏、进度、下载、系统分享。
- **DoD**：6 种格式在真机导出成功并可被 Office/浏览器正常打开；含表格样张验证 xlsx/docx 表格还原。

### Phase 4 — 历史记录与打磨（2 天）
- [ ] drift 持久化、首页最近记录、历史详情页。
- [ ] 动效打磨（§2.2 动效规范）、空态/加载态插画位、深浅色适配（v1 仅浅色，深色留 token 位）。
- [ ] 离线暂存与恢复上传。
- **DoD**：冷启动 < 2s；全流程 4 次点击内完成；无网络场景不崩溃且有明确提示。

### Phase 5 — 发布准备（1 天）
- [ ] iOS 权限文案（相机/相册）、App 图标、启动屏。
- [ ] 后端 Dockerfile + 部署脚本；图片与结果 TTL 清理任务。
- [ ] 端到端冒烟测试清单执行。
- **DoD**：TestFlight 可安装包 + 一键部署的后端。

### 后续 Phase（本迭代不做，但架构已预留）
- **P6 Android**：仅需实现 `platform/` 三个接口的 Android 版 + 相机权限适配。
- **P7 Windows**：实现接口 Windows 版；WebView 换 `webview_windows`（接口已抽象为 `StreamRenderer`，替换实现即可）；启用 ≥840px 双栏布局。
- **P8 结果编辑**：壳页面开启 contenteditable + 变更回传。

---

## 5. 开发规范与注意事项（AI 必读）

1. **令牌优先**：任何颜色/字号/圆角/间距必须来自 `tokens.dart`，PR 中出现魔法值视为不合格。
2. **接口冻结**：§3.5 API 契约为 v1 冻结版本，字段变更必须先更新本文档。
3. **平台隔离**：新增平台能力时先在 `infrastructure/platform/` 定义抽象接口，再写 iOS 实现；禁止在 UI/业务层写 `Platform.isIOS` 分支。
4. **流式健壮性**：SSE 必须处理断线重连（`Last-Event-ID` 续传由后端按分片序号支持）；WebView 注入必须节流。
5. **测试基线**：domain/application 层单测覆盖率 ≥ 80%；每个导出器必须有含中文、表格、图片的样张回归测试。
6. **提交粒度**：一个任务一个 commit，message 格式 `phaseN: <动词> <内容>`。
7. **进度同步**：每完成一个 checkbox，同步更新 `docs/PROGRESS.md`（状态 + 遇到的问题 + 决策），这是多 AI 协作的接力棒。

---

## 6. 真机测试与正式发布（iOS）

> 前置条件：开发电脑为 macOS 且已安装 **Xcode**（≥ 15）与 Flutter SDK；已注册 Apple ID。
> 真机调试可用免费 Apple ID；**TestFlight 与 App Store 发布必须加入 Apple Developer Program（$99/年）**。

### 6.1 一次性环境准备

```bash
xcode-select --install                 # 命令行工具
sudo xcodebuild -license accept        # 接受许可
flutter doctor                         # 确认 iOS toolchain / CocoaPods 全部 ✅
```

- Xcode → Settings → Accounts：登录 Apple ID，确认 Team 存在。
- App Store Connect（https://appstoreconnect.apple.com）→ 「用户与访问」确认权限；「App」页新建 App 前先在 https://developer.apple.com/account → Identifiers 注册 Bundle ID：`com.yourorg.snapdoc`（一旦发布不可更改，需与 Xcode 工程一致）。

### 6.2 真机测试

#### 方式 A：Flutter CLI 直接跑真机（日常开发）

1. iPhone 用数据线连接 Mac，首次需在手机上点「信任此电脑」。
2. 打开 `app/ios/Runner.xcworkspace`，在 Runner → Signing & Capabilities：
   - 勾选 **Automatically manage signing**，选择 Team；
   - Bundle Identifier 填 `com.yourorg.snapdoc`（免费账号需换一个唯一后缀）。
3. 运行：
   ```bash
   flutter devices                     # 确认设备被识别
   flutter run -d <device-id>          # Debug 模式，支持热重载
   flutter run --release -d <device-id>  # Release 模式，验证真实性能（流式渲染帧率必须在此模式验证）
   ```
4. 首次安装后，手机 设置 → 通用 → VPN与设备管理 → 信任开发者证书。
5. 无线调试：设备连接过一次后，Xcode → Window → Devices and Simulators → 勾选 *Connect via network*，之后可脱线 `flutter run`。

**真机联调后端注意**：手机无法访问 Mac 的 `localhost`。后端启动时监听 `0.0.0.0`，客户端 `API_BASE_URL` 配置为 Mac 局域网 IP（如 `http://192.168.x.x:8000`）；因 iOS ATS 默认禁止明文 HTTP，开发期在 `Info.plist` 中对该地址添加 `NSAppTransportSecurity` 例外（**发布版必须移除，正式后端必须 HTTPS**）。

#### 方式 B：TestFlight 内测（多人/远程测试，需付费开发者账号）

1. 按 §6.3 步骤 1–3 构建并上传构建版本。
2. App Store Connect → SnapDoc → TestFlight：
   - **内部测试**：添加团队成员（≤100 人），上传后几分钟即可测，无需审核；
   - **外部测试**：创建测试组 + 公开链接（≤10000 人），首个构建需通过一次轻量 Beta 审核（约 1 天）。
3. 测试者手机安装 TestFlight App → 打开邀请链接即可安装；崩溃与反馈在 TestFlight 页面查看。

#### 真机测试清单（每个 Release 候选必须过一遍）

- [ ] 相机拍照、多页连拍、相册多选（含 HEIC 图）；
- [ ] 弱网/断网下的上传重试与 SSE 断线重连；
- [ ] 流式渲染在 Release 模式无掉帧、无破碎 HTML；
- [ ] 6 种格式导出 → 系统分享面板 → 用「文件」/微信/邮件接收后能正常打开；
- [ ] 权限首次弹窗文案正确；拒绝权限后再次进入有引导；
- [ ] 深色系统外观下 UI 不错乱（v1 强制浅色也要验证）；
- [ ] 冷启动 < 2s（真机、Release 模式）。

### 6.3 正式发布到 App Store

#### 步骤 1：发布前配置

- `pubspec.yaml` 递增版本：`version: 1.0.0+1`（`1.0.0`=CFBundleShortVersionString，`+1`=构建号，**每次上传构建号必须递增**）。
- `Info.plist` 必须包含权限文案（缺失会被拒审）：
  - `NSCameraUsageDescription`：「用于拍摄需要识别的文档图片」
  - `NSPhotoLibraryUsageDescription`：「用于选择需要识别的图片」
  - `NSPhotoLibraryAddUsageDescription`：「用于保存导出的文档」（如有保存到相册功能）
- 移除开发期 ATS 明文例外；`API_BASE_URL` 切换为正式 HTTPS 域名（用 `--dart-define=API_BASE_URL=...` 注入，禁止硬编码）。
- App 图标（1024×1024 无透明通道）与启动屏就位。

#### 步骤 2：构建

```bash
cd app
flutter build ipa --release --dart-define=API_BASE_URL=https://api.snapdoc.example.com
# 产物：build/ios/archive/Runner.xcarchive 与 build/ios/ipa/*.ipa
```

签名问题排查时可改用 Xcode：Product → Archive → Organizer。

#### 步骤 3：上传构建

任选其一：
- **Xcode Organizer**：Distribute App → App Store Connect → Upload（最直观，首次发布推荐）；
- **Transporter.app**：Mac App Store 免费下载，登录后拖入 `build/ios/ipa/*.ipa` 即可上传；
- **CI 自动化（P6+ 再做）**：fastlane `pilot`/`deliver` + App Store Connect API Key。

上传后在 App Store Connect → TestFlight 等待「处理完成」（10–60 分钟），并回答出口合规（仅用 HTTPS → 选择「豁免加密」，可在 `Info.plist` 加 `ITSAppUsesNonExemptEncryption=false` 免每次询问）。

#### 步骤 4：App Store Connect 提交审核

1. 「App 信息」：名称（SnapDoc）、副标题、类目（效率）、隐私政策 URL（**必填**，需真实可访问网页）。
2. 「App 隐私」：声明数据收集——本产品上传用户图片到服务器识别，必须如实声明「用户内容（照片）— 用于 App 功能 — 不与身份关联 — 不用于追踪」。
3. 「定价与销售范围」：免费/付费与地区。
4. 新建版本 1.0.0：填写「此版本的新增内容」、上传截图（必需 6.7" 与 6.5" 两组，可用模拟器 `Cmd+S` 截取）、选择已处理完成的构建版本。
5. 「App 审核信息」：留联系人；因识别功能依赖后端，**必须保证审核期间正式后端在线**，并在备注中说明使用方法（可附一张测试用样例图说明）。
6. 提交审核 → 常规 1–3 天出结果；可选「审核通过后自动发布」或手动发布。

#### 常见被拒项自查（提交前必读）

- 权限弹窗文案含糊（Guideline 5.1.1）→ 用上面给出的具体文案；
- 隐私政策缺失或与「App 隐私」声明不一致；
- 审核期间后端不可用导致核心功能无法演示（2.1）→ 保证后端在线 + 审核备注写清楚；
- 有明文 HTTP 请求；
- 首版含「敬请期待」类占位功能。

#### 发布后

- 在 App Store Connect 关注崩溃率与评分；客户端接入 Flutter `PlatformDispatcher.onError` 全局兜底上报（P4 打磨项）。
- 热修复：不支持。任何修复都需递增构建号重新走 §6.3（可申请加急审核 Expedited Review，仅限严重问题）。
- 版本节奏：`x.y.z`——z=修复、y=功能、x=大版本；每次发版在 `docs/PROGRESS.md` 记录版本与变更。
