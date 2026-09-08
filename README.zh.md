# skills-4k

[English](./README.md) | 中文

为 AI 编程助手精心策划的技能包集合。

## 可用技能

### skill-forge

用可插拔运行器锻造、验证并调优 agent 技能——不锁定任何工具。搭建技能包脚手架、以 with/without 基准迭代运行测试提示词(宿主内子代理或 claude、codex、opencode、pi 等外部运行器)、按 expectations 评分、聚合基准,并优化 description 触发(native / explicit / simulated 三种判定策略)。分叉并精炼自 [anthropics/skills](https://github.com/anthropics/skills) 的 `skill-creator`。

**安装:**

```bash
npx skills add https://github.com/xypur/skills-4k --skill skill-forge
```

## 使用方法

用 Skills CLI 安装本仓库中的任意技能:

```bash
npx skills add https://github.com/xypur/skills-4k --skill <skill-name>
```

将支持技能系统的 AI 助手指向 `skills/` 目录,即可为其提供上下文感知的指导。

**作者:** LvHeng
