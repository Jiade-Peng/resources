# DESIGN.md

个性化界面定义

```
https://github.com/VoltAgent/awesome-design-md/tree/main
```

# AI_CONTEXT.md

```md
# 项目技术规范

## 1. 基础
- 桌面应用，使用 Electron + React (Vite 构建)
- 样式：Tailwind CSS，主题定义在 tailwind.config.ts
- 组件库：Shadcn/ui（组件源码在 src/components/ui），不要直接修改内部实现，通过包装组件扩展
- 图标：lucide-react，若缺少则使用 @heroicons/react
- 动效：framer-motion

## 2. 动效约定
- 页面路由切换：AnimatePresence + motion.div，使用 opacity + y 偏移，时长 0.2s ease-out
- 悬浮/点击反馈：whileHover={{ scale: 1.02 }}, whileTap={{ scale: 0.98 }}
- 列表项入场：使用 staggerChildren，delay 0.05s
- 避免在动画中使用 width/height 变化，统一使用 transform

## 3. Electron 安全与性能（必须严格遵守）
- 启用 contextIsolation，禁用 nodeIntegration
- 使用 preload 脚本通过 contextBridge 暴露安全 API
- 所有 IPC 通道名称定义在 shared/ipc-channels.ts
- 渲染进程禁止直接访问 Node.js 或 Electron 原生模块
- 大计算/文件操作放在主进程，通过 invoke/handle 调用
- 窗口创建使用 BrowserWindow 的 backgroundColor 属性避免白屏闪烁

## 4. 目录设计
my-electron-app/
├── electron/
│   ├── main.ts          # 主进程（窗口创建、菜单、IPC 处理）
│   ├── preload.ts       # 预加载脚本（暴露安全 API）
│   └── tsconfig.json
├── renderer/
│   ├── index.html
│   ├── src/
│   │   ├── main.tsx     # React 入口
│   │   ├── App.tsx      # 根组件
│   │   ├── components/  # Shadcn 组件（已安装的）
│   │   ├── lib/         # utils.ts (cn 函数)
│   │   └── styles/      # globals.css (含 Tailwind)
│   └── tsconfig.json
├── tailwind.config.js
├── postcss.config.js
├── components.json      # Shadcn 配置
└── electron-vite.config.ts
├── shared/            
│   └── ipc-channels.ts # 统一存放 IPC 通道名和公共 Type
├── tailwind.config.ts  # 建议改为 .ts 以匹配规范
├── electron-vite.config.ts
└── ...

## 5. 数据架构与 API 契约

### 5.1 单词对象模型 (Word Object Schema)
所有的词典查询结果必须遵循以下数据结构：

typescript
interface WordData {
  word: string;
  phonetic: {
    uk: string; // 英式音标
    us: string; // 美式音标
  };
  part_of_speech: string;
  definition: string; // 核心释义
  // 对应产品规范中的“释义即 Tab”逻辑，如果是多释义，建议使用数组
  definitions?: Array<{
    sense: string;
    simple_examples: Example[];
    complex_examples: Example[];
  }>;
  simple_examples: Example[]; // 简单例句（优先展示）
  complex_examples: Example[]; // 长难例句（后置展示）
}

interface Example {
  english: string;
  chinese: string;
}
```
# 产品设计规范 (Product Design Specification)

## 1. 核心理念
*   **极简主义 (Minimalist)**：消除所有冗余干扰，确保用户核心注意力集中于单词的学习与记忆。
*   **高效输入**：通过多模态（文本、视觉、音频）输入方式，实现极速查词体验。

## 2. 输入方式 (Input Methods)
*   **文本输入**：提供置顶的全局搜索框，支持模糊匹配。
*   **拍照查词 (OCR)**：利用摄像头捕捉图像，通过 Electron 主进程调用 OCR 服务提取文本。
*   **语音输入 (STT)**：实时声纹识别及语音转文字，支持即刻语音交互。

## 3. 单词展示与交互规范 (Word Display & Interaction)
单词详情页包含音标、释义、例句等核心字典要素

*   **核心层级**：单词标题 -> 音标 -> 发音按钮（使用 Lucide `Volume2` 图标）
*   **释义 Tab 布局**：
    *   **区分逻辑**：不同释义通过独立的 **背景色块** 进行视觉区分（如 `bg-blue-50`, `bg-green-50` 等）。
    *   **交互逻辑（新增）**：采用“释义块即 Tab”的设计。默认展示第一个释义块及其下方的例句。用户点击其他释义色块时，下方例句区域将动态切换为对应释义的内容。
*   **例句组织**：
    *   **分级体系**：每个释义下的例句分为“简单例句”与“长难例句”。
    *   **展示逻辑**：简单句在前，长难句在后。切换释义 Tab 时，例句区域通过 `AnimatePresence` 实现平滑的切入切出效果。

## 4. UI 视觉与动效约定
*   **排版**：参考 Notion 风格的结构化信息布局，严格控制字间距与行高以降低视觉疲劳。
*   **配色方案**：定义一套柔和的语义化色块映射表，用于自动分配给不同的释义块。
*   **动效辅助**：
    *   **释义切换**：使用 `framer-motion` 实现 0.2s 的 `opacity` 和 `y` 偏移效果。
    *   **入场动画**：整个单词详情页入场时使用 `staggerChildren` 效果。

## 5. 技术实现对齐
*   **渲染进程**：负责 Tab 切换逻辑、多语言排版及 `framer-motion` 动效。
*   **主进程**：处理 OCR 图像识别、语音流数据处理以及持久化生词本的存储。
*   **数据流**：所有 IPC 通道名称必须严格定义在 `shared/ipc-channels.ts` 中。

```

# 分阶段分任务执行

## 第一阶段：脚手架与目录初始化
```md
根据 @AI_CONTEXT.md 的目录结构要求，@DESIGN.md的设计要求，使用 electron-vite 创建项目基础脚手架。确保 electron/、renderer/ 和 shared/ 目录正确，并配置好 Tailwind CSS 和 TypeScript。
```

## 第二阶段：核心 UI 组件（Shadcn/ui + Tailwind）
```md
参考 @AI_CONTEXT.md 中的产品设计规范，实现单词详情页的 React 组件。

使用 Shadcn/ui 作为基础。
实现‘释义块即 Tab’的逻辑，支持点击切换例句。
按照规范中的‘配色方案’为不同释义分配背景色。
加入 @framer-motion 动效约定中的入场和切换动画。
```

## 第三阶段：IPC 桥接与安全通信
```md
根据 @AI_CONTEXT.md 的安全规范，在 shared/ipc-channels.ts 中定义查词、OCR 和语音识别的通道名。然后在 preload.ts 中通过 contextBridge 暴露对应的 API 接口。严禁在渲染进程直接调用 Node.js 模块。”\
```

## 第四阶段：功能集成（OCR 与 STT）
```md
在主进程 main.ts 中实现 OCR 处理逻辑。当收到渲染进程的图片数据时，进行模拟处理并返回结果。注意遵守‘大计算放在主进程’的规范。
```
