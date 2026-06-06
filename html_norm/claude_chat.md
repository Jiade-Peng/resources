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
   1.按照你的建议，2.遍历文件夹下是否有你需要的信息，没有就自己解析 3.深度改写，按照你觉得合理正确简洁的方式 4.精确保留视觉属性 5.视觉属性（字体、颜色、边框）像素级，相对排版保证和原来大概一致（可以让vl判断） 6.忽略skeleton_filled.html
  7.先做1个，后续作为agent处理其他的
```
