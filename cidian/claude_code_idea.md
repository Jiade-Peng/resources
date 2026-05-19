面是设计思路。先给整体骨架，再分能力展开，最后列出需要你拍板的决策点。

  ---
  一、总体架构

  核心思路：在主进程引入一个 意图路由 Agent，把现在的 searchWord 升级为多能力调度器。Agent 只做"识别意图 + 选择工具 + 整合返回"，具体能力下沉到独立的 service 模块。

  渲染端  ─┐
          ├─ IPC: agent:dispatch(input, context)  ──> 主进程 Agent
          │
          └<─ AgentResult (discriminated union)
                │
                ├─ kind: 'lookup'     ← word-search service（现有，复用）
                ├─ kind: 'translate'  ← translate service（新）
                ├─ kind: 'memorize'   ← wordbank + session service（新）
                ├─ kind: 'settings'   ← 语言切换 / 偏好（新）
                └─ kind: 'error'

  Agent 实现选择（建议）：不要急着上 tool-calling 真 Agent，用"轻量分类器 + 派发" 即可——单次 DeepSeek 调用让模型把输入归到 { intent, params }，主进程拿到结果直接走对应 handler。理由：

  - 当前能力 4 类，强类型可枚举，tool-calling 是过度设计。
  - 分类器调用便宜（短 prompt + JSON 输出）、可缓存、可写本地启发式快路径（如 "中→日切换" 这类短指令直接关键词命中，不走网络）。
  - 等出现"组合动作"（"查 X 然后加进我的记忆队列"）再升级到真 tool-calling agent，接口契约不变。

  ---
  二、能力清单 & 工具定义

  每个能力在主进程是一个独立 service；Agent 负责把自然语言映射到调用参数。

  ┌────────┬───────────────────────────────┬─────────────────────────────────────────┬───────────┐
  │  意图  │           触发示例            │             主进程 service              │ 输出 kind │
  ├────────┼───────────────────────────────┼─────────────────────────────────────────┼───────────┤
  │ 查词   │ taxonomy / 分类学             │ word-search.ts（现有）                  │ lookup    │
  ├────────┼───────────────────────────────┼─────────────────────────────────────────┼───────────┤
  │ 翻译   │ 翻译 "I went home" / 任意整句 │ translate.ts（新）                      │ translate │
  ├────────┼───────────────────────────────┼─────────────────────────────────────────┼───────────┤
  │ 背单词 │ 我想背4级的植物名词           │ wordbank.ts + memorize-session.ts（新） │ memorize  │
  ├────────┼───────────────────────────────┼─────────────────────────────────────────┼───────────┤
  │ 设置   │ 切换到中日词典 / 界面改成英文 │ settings.ts（新）                       │ settings  │
  └────────┴───────────────────────────────┴─────────────────────────────────────────┴───────────┘

  意图识别 prompt 大致输出：

  type IntentResult =
    | { intent: 'lookup',    word: string }
    | { intent: 'translate', text: string, srcLang?: string, dstLang?: string }
    | { intent: 'memorize',  level?: string, topic?: string, count?: number }
    | { intent: 'settings',  uiLang?: 'zh'|'en'|'ja', dictPair?: 'zh-en'|'zh-ja'|'en-zh'|'ja-zh' }
    | { intent: 'unknown' }

  unknown 兜底给现有 searchWord 的"模糊→候选"老逻辑，保持当前体验不退化。

  ---
  三、多语言（双维度，别合并）

  把"UI 语言"和"词典方向"拆开是关键：

  - UI 语言 (uiLang: 'zh'|'en'|'ja')：纯前端 i18n，建议用最小化方案（按 key 的 dict 文件）。不影响数据层。
  - 词典方向 (dictPair: 'zh-en'|'zh-ja'|'en-zh'|'ja-zh')：影响数据 schema、DB、查词 prompt、UI 展示。

  WordData schema 改造（最小破坏）：

  interface WordData {
    word: string
    srcLang: 'en'|'ja'|'zh'
    dstLang: 'zh'|'en'|'ja'
    phonetic: Record<string, string>   // en: {uk, us}; ja: {kana, romaji}; zh: {pinyin}
    partOfSpeech: string
    definition: string
    definitions: Array<{...}>          // 不变
  }

  DB：words 主键改成 (word, src_lang, dst_lang) 复合主键，老数据通过迁移补 src_lang='en', dst_lang='zh'。phonetic 列继续 JSON 字符串。definitions_cache 通过 FK 跟随。

  Prompt：FULL_LOOKUP_PROMPT 按 dictPair 选模板（中日 vs 中英），意图识别 prompt 不变。

  ---
  四、背单词

  这是新增工作量最大的一块。拆三层：

  (1) 词库来源 —— 混合策略（建议）

  ┌──────────┬──────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────┐
  │   来源   │                       适用                       │                                    实现                                     │
  ├──────────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
  │ 预置词表 │ CET-4 / CET-6 / TOEFL / GRE / N5–N1 等"标准考纲" │ 离线 JSON 打包进资源，启动时按需懒加载，写入 wordbank 表                    │
  ├──────────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
  │ 主题筛选 │ "植物 / 动物 / 商务 / 医疗"等子类                │ LLM 在选定考纲内做语义筛选，结果缓存到 wordlist_query 表（key=level+topic） │
  ├──────────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
  │ 纯生成   │ 没有预置考纲对应的请求                           │ 直接 LLM 生成 30–50 词，标 source: 'generated' 透明告知用户                 │
  └──────────┴──────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────┘

  具体到 "我想背4级的植物名称单词" → Agent 解析 { level: 'CET-4', topic: 'plants', count: ?默认20 } → 走"考纲内 LLM 筛选"路径。

  (2) 会话模型

  - memorize_sessions 表：id / created_at / wordlist_query_hash / status
  - memorize_progress 表：session_id / word / state ('new'|'learning'|'known'|'skipped') / last_reviewed_at
  - v1 不做 SRS（间隔重复），就是顺序播放卡片 + 标记。SRS 留给 v2 接 SM-2 算法。

  (3) UI 页面

  - 卡片视图：正面 word + 音标，背面释义+例句（复用 WordDetail 的内部块）。
  - 操作：左滑"还不会"、右滑"会了"、空格翻面（参考 Anki）。
  - 进度条 + "x / total" 计数。

  ---
  五、翻译

  最薄实现：单 DeepSeek 调用，prompt 固定为"翻译以下文本到 X 语言，仅输出译文"。

  - 不入库（短期价值低、命中率低）。
  - 输出 { srcText, dstText, srcLang, dstLang }。
  - UI：在搜索结果区上方显示一个译文卡片（不替换 WordDetail，因为单词查询和整句翻译是不同心智模型）。

  可选增强（v2）：自动语言检测、术语高亮、"译文中的某词"点击跳到查词。

  ---
  六、渲染端路由

  目前 App.tsx 是单页。引入轻量"页面状态机"，不要拉 react-router：

  type View =
    | { kind: 'home' }                              // 搜索框 + 最近查询
    | { kind: 'word', data: WordData }
    | { kind: 'translate', src: string, dst: string }
    | { kind: 'memorize', sessionId: string }
    | { kind: 'settings' }

  AgentResult 直接驱动 View 切换。AnimatePresence 在最外层做页面切换动效（已有 0.2s opacity+y 约定）。

  ---
  七、IPC 契约新增

  // shared/ipc-channels.ts
  AGENT_DISPATCH: 'agent:dispatch'

  // 主进程返回
  type AgentResult =
    | { kind: 'lookup',    data: SearchWordResult }
    | { kind: 'translate', src: string, dst: string, srcLang: string, dstLang: string }
    | { kind: 'memorize',  sessionId: string, words: WordData[], meta: { level?: string, topic?: string, source: 'curated'|'filtered'|'generated' } }
    | { kind: 'settings',  applied: { uiLang?: string, dictPair?: string } }
    | { kind: 'error',     message: string }

  老的 word:search 通道保留（直接精确查词时跳过 Agent，零延迟）；新通道 agent:dispatch 走自然语言路径。

  ---
  八、分阶段落地（建议顺序）

  ┌──────┬──────────────────────────────────────────────────────────────────┬────────────────────┐
  │ 阶段 │                               内容                               │      增量风险      │
  ├──────┼──────────────────────────────────────────────────────────────────┼────────────────────┤
  │ P1   │ 引入意图路由 + AgentResult 契约；lookup 走老逻辑、其他 kind 占位 │ 极低，向后兼容     │
  ├──────┼──────────────────────────────────────────────────────────────────┼────────────────────┤
  │ P2   │ 多语言（UI + dictPair），含 DB 迁移                              │ 中（schema 变更）  │
  ├──────┼──────────────────────────────────────────────────────────────────┼────────────────────┤
  │ P3   │ 背单词：预置 CET-4/6 词库 + 基础卡片页 + 顺序刷词                │ 中（新页面、新表） │
  ├──────┼──────────────────────────────────────────────────────────────────┼────────────────────┤
  │ P4   │ 翻译能力 + 译文卡片                                              │ 低                 │
  ├──────┼──────────────────────────────────────────────────────────────────┼────────────────────┤
  │ P5   │ SRS / 进度持久化 / 主题筛选缓存优化                              │ 低                 │
  └──────┴──────────────────────────────────────────────────────────────────┴────────────────────┘

  ---
  九、需要你拍板的决策点

  1. Agent 形态：轻量分类器（建议）vs 真 tool-calling agent？
  2. 背单词词库来源：预置 + LLM 筛选混合（建议）vs 全部 LLM 生成？预置词库容忍多大体积？（CET-4 ≈ 4500 词 ≈ 几百 KB JSON）
  3. 多语言广度 v1：只做中英 + 中日，还是一并把英日、日英也开？
  4. 翻译能力：v1 是否纳入？还是先专注查词 + 背单词？
  5. 路由方案：内置 View 状态机（建议）vs 引入 react-router？
  6. 背单词页交互：模仿 Anki 翻卡 + 滑动判定，还是更轻的"列表 + 打勾"？
  7. 历史 DB 迁移：现在用户量为零，是否直接 drop 重建表（最简）？
