 1. 外层骨架的 CSS 方案

  rows/cols 用 flexbox 还是 CSS Grid？我倾向于：
  - rows 之间用纵向 flow
  - 同一 row 内的多 cols 用 display: flex 或 grid，列宽根据 bbox 比例计算

  是否符合你的预期？

  2. node_ids 到原始 HTML 片段的映射

  layout.json 中 node_ids: ["n3", "n56"] 这些 ID 具体对应 original.html 里的哪些 DOM 子树？我看到 dom_tree.json 有节点信息，是否有直接的 node_id → HTML fragment 映射表，还是需要我按 DOM 深度优先顺序自己解析编号？

  3. item 内部的改写程度

  col 内部的每个 item（如 type=table、type=grid），是：
  - A) 保持原始 HTML 片段不动，只套进新骨架
  - B) 也需要对 item 内部做语义化重写（把 float 布局改成 flex/grid，重命名 class）

  如果是 B，那 item 内部改写到什么粒度？

  4. 样式处理方式

  - 继续用单文件 <style> 块，还是允许内联 style？
  - 原始 CSS 中的视觉属性（字体、颜色、边框）需要精确保留，还是只要"视觉等价"即可？

  5. 验证标准

  通过 Playwright CDP 验证时：
  - 是做像素级截图 diff（容差多少）？
  - 还是比对关键元素的 computed style + bounding box？
  - 文字内容是精确 text match？

  6. skeleton_filled.html 的关系

  我注意到每个目录已有 skeleton_filled.html，它和你期望的最终产出有什么区别？是作为参考/baseline，还是它本身不够好需要替代？

  7. 批量处理策略

  是先做 1 个 sample 打磨流程确认正确性，还是直接写脚本批量处理 28 个？

```txt
 1.按照你的建议，2.遍历文件夹下是否有你需要的信息，没有就自己解析 3.深度改写，按照你觉得合理正确简洁的方式 4.允许但是按照好的html写法应该少用内联，精确保留视觉属性
  5.视觉属性（字体、颜色、边框）像素级，相对排版保证和原来大概一致（可以让vl判断） 6.忽略skeleton_filled.html
    7.先做1个，后续作为agent处理其他的
```

```markdown
# HTML Normalization Rewrite Specification

## 目标

将 `original.html`（杂乱的 float/position 老式布局 HTML）深度重写为语义化、结构清晰的现代 HTML，精确保留视觉属性和文字内容。

## 输入

| 文件 | 来源 | 用途 |
|------|------|------|
| `original.html` | 样本目录 | HTML 源码：解析结构和 CSS |
| `screenshot_original.png` | Playwright 实时渲染 | 渲染图：辅助分析布局结构 |

渲染图由 `rewrite.py` 在重写前自动生成，或手动执行 `node compare.js <sample>` 生成。

## 输出

- `rewrite.html` — 规范化重写后的 HTML

---

## 分析流程

### 第一步：看渲染图，理解整体布局

先看 `screenshot_original.png`，从视觉上理解页面结构：
- 几栏布局？（单栏文章 / 两栏左右分栏 / 三栏 / 网格卡片）
- 有无固定的 header / footer / sidebar
- 内容区域的主要分块（标题区、列表区、图片区、导航区等）
- 是否有绝对定位的浮层（叠加在背景图上的文字块等）

### 第二步：读 HTML 源码，解析 CSS

从 `<style>` 块提取所有 class 规则，分离：
- **视觉属性**（必须保留）：font-family、font-size、font-weight、color、background-color、border（style/width/color）、border-radius、box-shadow、padding、text-align、line-height、text-decoration、opacity、white-space、overflow、object-fit
- **布局属性**（需理解后用现代 CSS 替代）：float、display、position、top/right/bottom/left、width、height、margin、clear、vertical-align

### 第三步：结合渲染图和源码，确定布局方案

对照渲染图验证从 CSS 推断的布局结构：

| 视觉特征（渲染图） | CSS 线索 | 重写方式 |
|------------------|----------|---------|
| 左右并排的区域 | `float:left/right` + width | `display: flex` |
| 等宽网格排列 | `float:left` + 相同 width 的重复元素 | `display: grid` |
| 叠加在背景图上的文字 | `position:absolute` + top/left | 保留 `position:absolute` |
| 横排的标签/按钮 | `display:inline-block` 或 flex | `display: flex` |
| 纵向堆叠的段落 | 无特殊布局 | 保持文档流 |
| 图文环绕 | `float:left/right` 的 img + 文字 | 保留 float（这是 float 的正确用法） |

**重要**：渲染图是真相来源。如果 CSS 分析得出的布局结论与渲染图不一致，以渲染图为准。

### 第四步：提取内容

- 所有文本节点内容（一字不差）
- 所有 `<img>` 的 src 和 data-* 属性
- 所有 `<a>` 标签（含无 href 的）
- 所有 `<input>`、`<textarea>`、`<button>` 等表单元素
- `<br>` 换行标签

---

## 重写规则

### 1. HTML 语义化

- 使用 HTML5 语义标签：`<article>`、`<header>`、`<nav>`、`<section>`、`<figure>`、`<main>`、`<aside>`、`<footer>`
- 修正错误嵌套（如 `<li>` 在 `<h2>` 内、`<div>` 在 `<ul>` 内）
- 减少嵌套层级（消除纯粹用于布局的 wrapper div）
- 保留所有 `data-*` 属性

### 2. CSS 策略

- **所有样式写在 `<style>` 块中**，不使用内联 style
- **语义化 class 命名**：如 `.article-title`、`.product-card`（替代 `.h1_0_0_0_0`、`.z9rsl1s0q8`）
- **精确保留视觉属性**：
  - font-family、font-size、font-weight → 原值照搬
  - color、background-color → rgb 值精确保留
  - border（样式、宽度、颜色）→ 精确保留
  - border-radius、box-shadow → 精确保留
  - padding → 精确保留
  - line-height、text-align → 精确保留
- **布局属性用现代 CSS 替代**：
  - `float: left/right` 并排 → `display: flex`
  - `display: inline-block` 并排 → `display: flex`
  - 绝对定位 → 按需保留或改 flex
  - `clear: both` → 不需要（flex 不产生 float）

### 3. 尺寸和间距

- 保留根容器的显式 width
- 保留关键元素的 height（如固定高度的卡片）
- 保留 margin 和 padding 的值
- 空元素间隔符（空 `<p>`、空 `<div>`）用 margin 替代

### 4. 文本内容

- **所有文字与原始完全一致**（包括空格和特殊字符）
- 保留 `<br>` 换行标签
- 保留 `<a>` 标签（即使没有 href）

### 5. font-size 继承

- 不随意在根容器设置 font-size
- 原始 HTML 若未在某容器设置 font-size，则继承浏览器默认 16px
- 只在原始 CSS 明确设置了 font-size 的元素上保留该值

---

## 验证标准

### 必须通过

1. **文本一致性**：Playwright 提取的所有 textContent 与原始完全一致
2. **视觉属性像素级**：关键元素的 computed style（font、color、border）与原始一致

### 近似通过（VL 判断）

3. **相对排版**：整体布局、元素间距与原始大致一致
4. **多栏布局**：列宽比例、元素相对位置正确

---

## 处理流程总结

1. **看图**：读取 screenshot_original.png，理解页面视觉布局
2. **读码**：解析 original.html 的 `<style>` 和 `<body>`
3. **对照**：结合渲染图和 CSS 确定布局方案
4. **重写**：生成语义化 HTML + CSS
5. **验证**：Playwright 截图对比 + 文本/样式验证

```

  1. 流程关系
  rewrite.html 是中间产物还是最终给 Word 转换用的？也就是说 —— 是让 rewrite 输出的 HTML 本身就要受 Word 能力的约束，还是 rewrite 和 Word 转换是两个独立步骤？

  2. 多栏布局
  python-docx 支持的布局手段只有：表格（无边框表格模拟分栏）、分栏（Section columns，但只能等宽或固定宽度，不能精细控制每列内容）。对于原始 HTML 中的左侧栏+内容+右侧栏三栏布局，你是希望在 Word 中用表格保留多栏效果，还是接受降级为单栏纵向排列？

  3. 绝对定位 / 叠加
  比如 sample_000019 那种文字叠加在背景图上的设计，Word 中只能用 TextBox 或 "图片+文字环绕" 来近似。这种复杂情况是要尝试还原，还是可以跳过或降级？

  4. 最终目的
  转 Word 是为了什么场景？（阅读？编辑？打印？归档？）—— 这决定了排版精度的优先级。
