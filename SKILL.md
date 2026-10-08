---
name: ppt-agent
description: 专业 PPT 演示文稿全流程 AI 生成助手。模拟顶级 PPT 设计公司的完整工作流（需求调研 -> 资料搜集 -> 大纲策划 -> 策划稿 -> 设计稿），输出高质量 HTML 格式演示文稿。当用户提到制作 PPT、做演示文稿、做 slides、做幻灯片、做汇报材料、做培训课件、做路演 deck、做产品介绍页面时触发此技能。即使用户只说"帮我做个关于 X 的介绍"或"我要给老板汇报 Y"，只要暗示需要结构化的多页演示内容，都应该触发。也适用于用户说"帮我把这篇文档做成 PPT"、"把这个主题做成演示"等需要将内容转化为演示格式的场景。
---

# PPT Agent -- 专业演示文稿全流程生成（精简版）

> 精简版说明：保留 26 套风格、完整 6 步工作流与全部生成脚本；去掉了示例输出与冗余流程模板，核心 Prompt 模板见 `references/prompts.md`（5 套合一版）。

## 核心理念

1. **先调研后生成** -- 用真实数据填充内容，不凭空杜撰
2. **策划与设计分离** -- 先验证信息结构，再做视觉包装
3. **内容驱动版式** -- Bento Grid 卡片式布局，每页由内容决定版式
4. **全局风格一致** -- 先定风格再逐页生成，保证跨页统一

---

## 环境感知

开始工作前自省 agent 拥有的工具能力：

| 能力 | 降级策略 |
|------|---------|
| **信息获取**（搜索/URL/文档/知识库） | 全部缺失 -> 依赖用户提供材料 |
| **图片生成** | 缺失 -> 纯 CSS 装饰替代 |
| **文件输出** | 必须有 |
| **脚本执行**（Python/Node.js） | 缺失 -> 跳过自动打包和 SVG 转换 |

**原则**：检查实际可调用的工具列表，有什么用什么。

---

## 路径约定

| 变量 | 含义 | 获取方式 |
|------|------|---------|
| `SKILL_DIR` | 本 SKILL.md 所在目录的绝对路径 | 触发 Skill 时读取 SKILL.md 的目录 |
| `OUTPUT_DIR` | 产物输出根目录 | 用户当前工作目录下的 `ppt-output/`（首次使用时 `mkdir -p` 创建） |

---

## 6 步 Pipeline

### Step 1: 需求调研 [必须等用户回复]

**执行**：使用 `references/prompts.md` Prompt #1（需求调研 7 问）
1. 搜索主题背景资料（3-5 条）
2. 一次性向用户提问（场景/受众/期望行动/叙事结构/内容侧重/说服要素/执行细节，页数少时可精简为 3 题）
3. **等待用户回复**后整理为需求 JSON

### Step 2: 资料搜集

根据主题规划查询并行搜索，每组结果摘要总结。产物：搜索结果集合 JSON。

### Step 3: 大纲策划

**执行**：使用 `references/prompts.md` Prompt #2（大纲架构师，金字塔原理）
自检：页数符合要求 / 每 part >= 2 页 / 要点有数据支撑。产物：`[PPT_OUTLINE]` JSON。

### Step 4: 内容分配 + 策划稿 [建议等用户确认]

**执行**：使用 `references/prompts.md` Prompt #3
- 将搜索素材精准映射到每页
- 每页：主卡片 40-100 字 + 数据亮点 + 辅助要点
- 确定 page_type / layout_hint / cards[] 结构（布局参考 `references/bento-grid.md`）
- 每个内容页至少 3 张卡片 + 2 种 card_type + 1 张数据卡片

向用户展示策划稿概览，建议确认后再进入 Step 5。产物：`OUTPUT_DIR/planning.json`（可用 `scripts/planning_validator.py` 校验）。

### Step 5: 风格决策 + 设计稿生成

#### 5a. 风格决策

阅读 `references/styles/index.md` 主索引（含决策矩阵），按主题匹配 26 种预置风格之一：

| 板块 | 风格 ID |
|------|--------|
| 暗色专业（7） | dark_tech / xiaomi_orange / luxury_purple / nocturne_violet / cyberpunk_neon / chrome_y2k / noir_film |
| 浅色高级（8） | blue_white / fresh_green / minimal_gray / mocha_editorial / medical_pulse / earth_concrete / champagne_gold / liquid_glass |
| 活力鲜明（4） | vibrant_rainbow / kindergarten_pop / bauhaus_block / candy_pastel |
| 东方文化（3） | royal_red / sakura_wabi / ink_jade |
| 自然/复古（4） | botanic_forest / safari_savanna / retro_70s / gov_authority |

**必读**：`references/typography.md`（排版铁律）。产物：`OUTPUT_DIR/style.json`。

#### 5b. 逐页 HTML 设计稿生成

**执行**：使用 `references/prompts.md` Prompt #4 + `references/bento-grid.md`

每页 Prompt = Prompt #4 模板 + 风格 JSON + 该页策划稿 JSON + 该页内容文本。

**核心设计约束**：
- 画布 1280x720px，overflow:hidden
- 所有颜色通过 CSS 变量引用，禁止硬编码
- 视觉元素必须是真实 DOM 节点，图形优先内联 SVG
- 遵守 `references/pipeline-compat.md` 管线兼容性规则

**分批策略**：按 Part 分批，每批 3-5 页，写入 `OUTPUT_DIR/slides/`。

**资源路由**：`layout_hint→references/layouts/`、`page_type→references/page-templates/`、`card_type→references/blocks/`、`chart_type→references/charts/`。

### Step 6: 后处理 [必做]

```
slides/*.html --> preview.html --> svg/*.svg --> presentation.pptx
```

```bash
pip install python-pptx lxml Pillow 2>/dev/null
python3 SKILL_DIR/scripts/html_packager.py OUTPUT_DIR/slides/ -o OUTPUT_DIR/preview.html
python3 SKILL_DIR/scripts/html2svg.py OUTPUT_DIR/slides/ -o OUTPUT_DIR/svg/
python3 SKILL_DIR/scripts/svg2pptx.py OUTPUT_DIR/svg/ -o OUTPUT_DIR/presentation.pptx --html-dir OUTPUT_DIR/slides/
```

- `preview.html` -- 浏览器翻页预览
- `presentation.pptx` -- 可编辑 PPTX（右键"转换为形状"）
- 若 Node.js 不可用，跳过 SVG/PPTX 步骤，仅输出 preview.html

---

## 质量自检

| 维度 | 检查项 |
|------|-------|
| 内容 | 每页 >= 2 信息卡片 / >= 60% 内容页含数据 / 章节有递进 |
| 视觉 | 全局风格一致 / 卡片不重叠 / 文字不溢出 |
| 技术 | CSS 变量统一 / `pipeline-compat.md` 禁止清单检查 |

## Reference 文件索引

| 文件 | 何时阅读 |
|------|---------|
| `references/prompts.md` | 每步生成前（5 套 Prompt 模板合一版）|
| `references/styles/index.md` | Step 5a 风格决策 |
| `references/styles/{dark,light,vibrant,cultural,natural}.md` | 选定风格后 |
| `references/typography.md` | Step 5b 排版 |
| `references/charts/index.md` | 数据可视化时 |
| `references/layouts/*.md` | layout_hint 路由 |
| `references/blocks/*.md` | card_type 路由 |
| `references/page-templates/*.md` | page_type 路由 |
| `references/principles/*.md` | 设计原则自检 |
| `references/bento-grid.md` | 布局决策 |
| `references/method.md` | 方法论 |
| `references/pipeline-compat.md` | 管线兼容性 |
| `references/cli-cheatsheet.md` | 脚本调用速查 |
