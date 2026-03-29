---
name: iterate
description: Generator/Evaluator 自动迭代循环。扫描代码库，拟定评分标准，循环修改+独立 agent 评估直到达标。支持截图/URL/文字描述作为输入。
user-invokable: true
args:
  - name: target
    description: 要优化的目标（如"字体""报告布局""DCF 组件"），可附带截图路径或 URL
    required: false
---

# Generator/Evaluator 迭代循环

核心理念：把"改东西 → 自己看看 → 感觉还行"替换为"改东西 → 自动验证 → 独立评分 → 根据反馈再改"。Generator 和 Evaluator 必须是独立上下文，避免自评偏差。

## Phase 1：理解目标

### 1a. 接收用户输入

用户可以通过三种方式告诉你要优化什么：

- **截图路径**：用户提供图片文件路径（如 `~/Desktop/screenshot.png`），直接读取查看当前状态
- **URL**：用户提供页面地址（如 `localhost:3000`），用 Playwright 自动截图
- **纯文字描述**：用户用语言描述要优化的部分，从代码推断

### 1b. 探索代码库

1. 用 Glob/Grep/Read 找到相关文件，理解当前实现
2. 如果用户提供了 URL，用 Playwright 截图当前状态作为基线
3. 列出涉及的文件和当前实现概况

### 1c. 拟定评分标准

根据优化目标自动写 3-5 条评分维度，每项 1-10 分。

**如果目标涉及设计/UI/前端视觉，默认包含以下四条基础标准**（可根据具体目标增减）：

1. **设计质量**（Design Quality）— 设计是否给人一个有机整体的感觉，而非零件的拼凑？色彩、字体、布局、图像及其他细节共同营造出独特的氛围和个性。
2. **原创性**（Originality）— 是否有自主设计决策的痕迹，还是模板布局、组件库默认值和 AI 生成套路的堆砌？未经修改的现成组件或 AI 生成典型特征（白色卡片上的紫色渐变）会被判不合格。
3. **工艺**（Craft）— 技术执行层面：字体层级、间距一致性、色彩和谐、对比度。能力检验而非创意检验。
4. **功能性**（Functionality）— 独立于美学的可用性。用户能否理解界面用途，找到主要操作，顺畅完成任务？

**如果是非设计类目标**（如计算逻辑、prompt 质量、性能），根据具体目标拟定 3-5 条专属标准。

### 1d. 确定验证方式和结束条件

- **验证方式**：Playwright 截图 / 跑测试 / 对比输出文件 / 手算验证
- **结束条件**：默认"全部 ≥ 8/10 或 8 轮"，用户可调

## Phase 2：用户确认

将以上内容展示给用户确认：

```
修改范围：[文件列表 + 改动类型]

评分标准（每项 1-10）：
  1. [维度名] — [具体怎么评]
  2. [维度名] — [具体怎么评]
  3. ...

验证方式：[截图 / 测试 / 输出文件]
结束条件：全部 ≥ [目标分]/10 或 [N] 轮
```

用户可以修改任何一项。确认后进入迭代。

## Phase 3：准备基础设施

1. **备份**：`cp` 原文件到安全位置（如 `项目目录/iterations/backup/`）
2. **Mock/快速验证**：如果验证需要长时间运行（如 API 调用、编译），设置 mock 数据或缓存，确保每轮验证 < 10 秒
3. **基线截图/评分**：在修改前先跑一次 evaluator，记录基线分数
4. **创建迭代文件夹**：`项目目录/iterations/` 存放每轮截图和评分

## Phase 4：迭代循环（全自动，不需要用户介入）

### 4a. Generator（你自己做）
- 读上一轮 evaluator 反馈
- 按反馈修改代码（只改定义范围内的东西）
- 语法检查确保不报错（TypeScript: `npx tsc --noEmit`，Python: `python -c "import ast; ast.parse(...)"` 等）

### 4b. 验证
- 截图 / 跑测试 / 生成输出
- 保存到迭代文件夹：`iter_N_*.png` 或 `iter_N_output.txt`

**Playwright 截图模板**（前端类）：
```javascript
const pw = require('playwright');  // 或从全局路径 require
const browser = await pw.chromium.launch({ headless: true });
const page = await browser.newPage({ viewport: { width: 1440, height: 900 } });
await page.goto(URL);
await page.waitForTimeout(3000);
// 注意：SPA 的滚动容器可能不是 window，需要找 .overflow-y-auto 等元素
const scrollContainer = await page.$('.overflow-y-auto');
if (scrollContainer) {
  await scrollContainer.evaluate(el => el.scrollTop = 0);
}
await page.screenshot({ path: 'iter_N_top.png' });
await browser.close();
```

### 4c. Evaluator（spawn 独立 Agent）
- **必须用 Agent 工具 spawn 独立子 agent**，不能自己评自己
- 给 evaluator 的 prompt 包含：
  - 当前截图/输出文件路径
  - 上一轮截图/输出（对比用）
  - 评分标准原文
  - 指令："严格打分，不要客气。每项给出分数 + 剩余问题 + 一条具体修复建议"
- evaluator 输出：每项分数（1-10）+ 剩余问题 + 具体修复建议 + 汇总表

### 4d. 判断
- 全部达标 → 进入 Phase 5
- 未达标 → 回到 4a
- 达到最大轮数 → 进入 Phase 5 并告知用户当前分数

## Phase 5：收尾

### 5a. 生成对比产出

**前端/视觉类**：生成一个 HTML 文件，标签页切换每版迭代效果：

```html
<!-- 每个 tab 对应一轮迭代，点击切换显示对应截图 + 评分 -->
<div class="tabs">
  <div class="tab" data-iter="0">基线</div>
  <div class="tab" data-iter="1">Iter 1</div>
  ...
</div>
<div class="panel">
  <div class="eval">分数表格</div>
  <img src="iter_N_*.png" />
</div>
```

**其他类**：生成 markdown 对比报告。

### 5b. 总结

1. 列出所有修改的文件和改动内容
2. 记录迭代中发现的**超范围问题**（不在修改范围内但值得注意的），告诉用户

## 关键规则

1. **Evaluator 必须是独立 Agent** — 避免自评偏差，这是整个方法的核心
2. **每轮改动要小** — 一次改 2-3 个问题，不要大改，改太多分不清哪个有效
3. **不跑偏** — 严格只改定义范围内的东西，超范围的记录但不修
4. **Mock 数据用真实数据** — 不要用 placeholder 内容
5. **分数有波动是正常的** — ±1 分波动属于正常（10 分制），连续两轮同一项不过才算真的有问题
6. **备份在前** — 任何修改开始前必须备份，随时可以 `cp` 回去恢复
