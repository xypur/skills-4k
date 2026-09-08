---
name: skill-forge
description: "用可插拔运行器锻造、验证并调优 agent 技能——不锁定任何工具。适用于:从零搭建技能包、起草或重构 SKILL.md、以 with/without 基准迭代运行测试提示词(通过宿主内子代理或 claude、codex、opencode、pi 等外部运行器)、按 expectations 评分并聚合基准、或优化技能 description 的触发准确率(native、explicit、simulated 三种判定策略)。不负责仓库特定的打包约定——宿主仓库存在作者技能(如 skill-dev)时以其为准。"
metadata:
  author: LvHeng
  version: "2026.9.2"
  source: Forked & distilled from anthropics/skills skill-creator (Apache-2.0); external dependencies abstracted behind a runner layer
---

# Skill Forge

锻造 agent 技能并做实证验证——**不锁定任何运行器**。所有外部能力(一次性 LLM 调用、agent 会话、触发探测)都经过可插拔的运行器层,默认使用当前宿主已具备的能力。

## 何时使用此技能

- 从零或从一段已捕获的工作流搭建新的技能包
- 起草、重构或精简 SKILL.md
- 以 with/without 基准迭代运行测试提示词
- 按 expectations 评分、聚合基准、审阅输出
- 优化技能 `description` 的触发准确率
- 把评测/优化循环移植到不同的 agent 工具(claude、codex、opencode、pi……)

## 核心原则

1. **渐进披露**——上下文是稀缺资源。L1 frontmatter(`name` + `description`,约 100 词,始终加载)负责赢得触发;L2 正文(<500 行,触发时加载);L3 `references/` + `scripts/`(按需)。详细材料绝不膨胀正文。
2. **解释 why**——带理由的指令能泛化到未预料的情况;光秃秃的 `ALWAYS`/`NEVER` 不能。硬性禁止仅留给真正永远错误的事。
3. **泛化而非过拟合**——你只在少数测试用例上迭代,却要面向无数未来提示词交付。应用任何修复前先问"这个修复能泛化吗"。
4. **确定性部分脚本化**——脚手架、校验、聚合属于 `scripts/`,不靠反复的 agent 人工操作。
5. **默认宿主无关**——绝不在工作流里硬编码厂商 CLI。优先宿主内路径;通过运行器层优雅降级([references/backends.md](references/backends.md))。

## 工作流

### 步骤 1:捕获意图

写任何东西之前先回答;若用户说"把这个做成 skill",先从对话中提取:

1. 这个技能要让 agent 能做什么?
2. 何时触发?(用户措辞、场景、近似误触发)
3. 触发后的预期输出/行为是什么?
4. 产出是否可客观验证?是则规划测试提示词(步骤 5);若主观(写作风格、设计),依赖定性评审。

### 步骤 2:搭建包结构

```text
<skill-name>/            # kebab-case,与 frontmatter name 一致
├── SKILL.md             # 必需
├── references/          # 详细文档,按需加载
├── scripts/             # 确定性/重复性逻辑,可不加载直接执行
├── assets/              # 输出用到的文件(模板、字体、图标)
└── evals.json           # 测试提示词 + expectations
```

多领域技能按变体组织 `references/`,只读相关文件。

### 步骤 3:写 Frontmatter 与 Description

description 是触发机制:必须同时携带**技能做什么**与**何时使用**——3 个以上具体触发场景、用户会真实输入的领域名词、略微 pushy(模型倾向于漏触发)。description 里不放使用说明;那属于正文。

**弱:** `description: "Vue component library conventions."`

**强:** `description: "Vue 3 component library authoring conventions. Use when writing or reviewing library components, designing props/emits/slots APIs, deciding emits vs callback props in JSX, or organizing tests and hooks in a component library."`

### 步骤 4:写正文

- 一句话概述 → 触发场景 → 工作流(过程性内容用 `### Step N:` 编号) → 附加章节(速查用表格) → 仅在确有内容时保留 Prohibitions / When Unsure。
- 关键指导前置;假设后面的章节可能永远不会被读。
- 引用文件时附**何时读**的指针;超过 300 行的文件需要目录。
- 映射关系用表格,目录树和命令用代码块。

### 步骤 5:创建测试提示词

写 2–3 条真实提示词——真实用户会说的话,具体且实质到"查阅技能确实有帮助"。连同 `expected_output` 与 `expectations`(可验证陈述)记录在技能根目录的 `evals.json`。完整 schema:[references/schemas.md](references/schemas.md)。

```json
{
  "skill_name": "<skill-name>",
  "evals": [
    {
      "id": 1,
      "prompt": "以用户口吻描述的任务",
      "expected_output": "正确结果长什么样",
      "expectations": ["可验证陈述 1", "可验证陈述 2"]
    }
  ]
}
```

### 步骤 6:运行评测循环

工作区布局(聚合脚本要求 `run-N` 层):

```text
<skill-name>-workspace/
└── iteration-1/
    └── eval-<描述性名称>/
        ├── eval_metadata.json          # {eval_id, eval_name, prompt, assertions}
        ├── with_skill/run-1/           # grading.json + outputs/(+ timing.json)
        └── without_skill/run-1/
```

**执行器运行**——每种配置各一个运行,同一轮全部发出:

- *宿主内路径(默认,零外部依赖):* 使用宿主原生子代理能力(pi subagents、Claude Code Task tool……)。with_skill 运行先读技能路径再应用;without_skill 运行只收到任务本身。
- *外部运行器路径:* 经后端适配器路由——各工具命令与探测要点见 [references/backends.md](references/backends.md)。
- **污染规则:** without_skill 运行不能有机会发现技能。当技能就在 agent 会搜索的仓库里时,从剪除副本(或中性 cwd)执行——agent 确实会读它们找到的技能文件,并悄悄让你的 baseline 失效。

**评分**——能用程序检查的尽量用脚本(脚本胜过目测);需要判断力的用 LLM 评分器(宿主模型或 `complete()` 运行器)。写 `grading.json`,字段为 `expectations[{text, passed, evidence}]`——字段名必须精确,viewer 依赖它们。

**聚合与审阅:**

```bash
python3 scripts/aggregate_benchmark.py <workspace>/iteration-1 --skill-name <name>
python3 scripts/generate_review.py <workspace>/iteration-1 \
  --skill-name <name> --benchmark <workspace>/iteration-1/benchmark.json \
  --static <workspace>/review.html
```

在聚合数据之上做分析师批注:两臂都通过的断言(无区分度)、高方差 eval(不稳定)、时间/token 权衡——记入 `benchmark.json` 的 `notes`。

### 步骤 7:依据反馈迭代

- 修原因而非症状;只修补某个测试用例的花哨补丁是过拟合。
- 删掉没起作用的部分;读 transcript 而不只是最终输出。
- 如果每个执行器运行都在独立重造同一个辅助工具,把它提升进技能的 `scripts/`。
- 以 `iteration-2/` 重跑(baseline = 上一迭代或无技能;保持一致),带 `--previous-workspace` 重新生成 viewer;反馈为空或进展停滞时停止。

### 步骤 8:优化 Description

构建触发评测集:20 条真实查询——8–10 条 should-trigger(措辞多样,部分不点名技能)+ 8–10 条近似误触发 should-not(共享关键词、邻近领域、模糊表述;绝不出显然无关的)。存为 `trigger-evals.json`。

按保真度与可用性选择探测策略(细节:[references/backends.md](references/backends.md)):

| 策略 | 做法 | 保真度 | 需要 |
|---|---|---|---|
| `native` | 技能装进工具自己的技能目录,跑真实会话,从 transcript 检测技能使用 | 最高 | claude、pi |
| `explicit` | 任务明示"先读技能 X";测的是执行质量而非触发 | 中 | 任意 `run_agent` |
| `simulated` | 只给判定器 name+description 列表 + 查询;问它会咨询哪个技能 | 对 description 调优足够 | 任意 `complete()` |

默认 `simulated`(处处可用,已被证明能抓住只写在正文、触发时不可见的能力)。60/40 划分训练/保留集,评估,让强模型依据失败案例提出改进 description,再评估,迭代 ≤5 轮,按**保留集**分数选优并应用。当宿主模型不宜评判自己的 description 时,经运行器层换判定器。

### 步骤 9:打包与集成

适配宿主约定:来源元数据(`GENERATION.md`)、变更日志(`CHANGES.md`)、双语镜像、技能目录/README 索引。更新宿主列出技能的任何索引——包括安装命令。需要 `.skill` 产物时可用 `scripts/package_skill.py`(见 NOTICE)。

## 禁止事项

- 不得在工作流或脚本中硬编码厂商 CLI——每次外部调用都经过运行器适配器或宿主内路径。
- 不得让 without_skill 运行有机会发现技能(baseline 污染会让基准作废)。
- 不得把使用说明或长表格塞进 `description`。
- 不得靠内联参考材料让 SKILL.md 超过 ~500 行。
- 不得交付内容与用户所述意图相悖(令人意外)的技能。
- 没有真实命令输出或评审证据支撑,不得声称"已验证"。

## 不确定时

- 选哪种触发策略 → description 工作用 `simulated`;执行质量问题用 `inhost`/`run_agent`;仅当后端的 transcript 解析真正实现时才用 `native`。
- 某章节是否需要 → 先省略;真实案例需要时再加回。
- 测试该多严格 → 默认 2–3 条带记录 expectations 的提示词;技能高频使用或用户要求时才做更深基准。
- 哪个后端可用 → 按 [references/backends.md](references/backends.md) 的探测顺序;一个都没有时,以上一切仍可在宿主内完成。
