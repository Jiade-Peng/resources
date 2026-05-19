技术栈

  - 主进程：Electron 42 + TypeScript + better-sqlite3
  - 渲染进程：React 18 + Vite + Tailwind CSS + Shadcn/ui + framer-motion + lucide-react
  - 构建工具：electron-vite

  目录结构

  electron/        主进程
    main.ts        IPC 注册 + 窗口管理
    preload.ts     contextBridge 暴露 electronAPI
    services/
      database.ts     SQLite (words / definitions_cache 两表 + 90天/2000条清理)
      word-search.ts  三级搜索 + DeepSeek 调用 + .env 加载
  shared/
    ipc-channels.ts   IPC 通道常量 + WordData / SearchWordResult 类型
  renderer/src/
    App.tsx              入口（搜索框 + 候选 Tag + 详情）
    components/search/   SearchBox、SuggestedTags
    components/word/     WordDetail（释义 Tab + 关键词高亮）
    components/ui/       Shadcn 基础组件

  核心架构（来自 AI_CONTEXT.md）

  - 安全：contextIsolation 开 / nodeIntegration 关；API Key 仅主进程通过 .env 读取，渲染进程绝不接触
  - 本地优先三级搜索（searchWord）：
    a. SQLite 精确命中 → source: 'local'，零网络
    b. 未命中调 DeepSeek 意图识别 → suggestedWords
    c. 取候选首词，本地缓存优先，否则全量拉取并 saveWord 持久化 → source: 'fuzzy'
  - 数据契约：snake_case 仅在主进程 ↔ DeepSeek 之间使用，渲染进程只见 camelCase WordData
  - 视觉联动：每个义项按 index % 5 绑定一个 theme_color（blue/emerald/violet/amber/rose），存库时冻结，渲染时正则匹配关键词上色
