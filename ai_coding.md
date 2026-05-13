# DESIGN.md

个性化界面定义

```
https://github.com/VoltAgent/awesome-design-md/tree/main
```

# AI_CONTEXT.md

```md
# 项目技术规范

## 基础
- 桌面应用，使用 Electron + React (Vite 构建)
- 样式：Tailwind CSS，主题定义在 tailwind.config.ts
- 组件库：Shadcn/ui（组件源码在 src/components/ui），不要直接修改内部实现，通过包装组件扩展
- 图标：lucide-react，若缺少则使用 @heroicons/react
- 动效：framer-motion

## 动效约定
- 页面路由切换：AnimatePresence + motion.div，使用 opacity + y 偏移，时长 0.2s ease-out
- 悬浮/点击反馈：whileHover={{ scale: 1.02 }}, whileTap={{ scale: 0.98 }}
- 列表项入场：使用 staggerChildren，delay 0.05s
- 避免在动画中使用 width/height 变化，统一使用 transform

## Electron 安全与性能（必须严格遵守）
- 启用 contextIsolation，禁用 nodeIntegration
- 使用 preload 脚本通过 contextBridge 暴露安全 API
- 所有 IPC 通道名称定义在 shared/ipc-channels.ts
- 渲染进程禁止直接访问 Node.js 或 Electron 原生模块
- 大计算/文件操作放在主进程，通过 invoke/handle 调用
- 窗口创建使用 BrowserWindow 的 backgroundColor 属性避免白屏闪烁
```
