# 运行器与后端参考

skill-forge 运行器抽象的规范性参考。SKILL.md 工作流与运行器无关;本文件把每个原语映射到具体工具。状态注记反映作者机器上 2026-09-02 的探测结果——依赖之前请重新探测。

## 运行器契约

```python
class Runner:
    name: str

    def available(self) -> bool:
        """CLI 在 PATH 上(或宿主能力存在)且已通过认证。"""

    def complete(self, prompt: str, model: str | None = None, timeout: int = 120) -> str:
        """一次性文本生成。支撑:description 改进、LLM 评分、simulated 触发判定。
        不要求技能感知能力。"""

    def run_agent(self, task: str, cwd: str, timeout: int = 600,
                  skill_dirs: list[str] | None = None) -> RunResult:
        """一次 agent 会话,可选挂载技能。
        支撑:触发探测(native 策略)、执行器运行。
        RunResult = {output_text, skill_used: bool | None, transcript_path}"""
```

当后端无法观测技能使用时,`skill_used` 为 `None`——调用方必须把它当作"未知"处理,绝不能当作"未触发"。

## 后端对照表

| 后端 | `complete()` | `run_agent()` | 技能存放位置 | `skill_used` 检测 | 状态 2026-09-02 |
|---|---|---|---|---|---|
| `inhost` | 宿主模型本身(运行本技能的 agent) | 宿主原生子代理工具(pi subagents、Claude Code Task tool……) | 已由宿主安装 | 不适用(with_skill 运行是显式的) | **始终可用**——默认路径 |
| `claude` | `claude -p "<prompt>" --output-format json` | `claude -p "<task>"` | `~/.claude/skills/` | 解析 transcript 中的技能调用 | 已安装+已认证,但 `claude -p` **会挂起**(90 秒后以退出码 124 结束)——使用前先探测 |
| `codex` | `codex exec "<prompt>"` | `codex exec --cd <cwd> "<task>"` | 无原生技能——通过 prompt 注入(仅 explicit 策略) | 不可观测 → `None` | 可用 |
| `opencode` | `opencode run "<prompt>"` | `opencode run "<task>"` | AGENTS.md 生态约定 | 取决于版本 | 可用(也是当前 `PI_PROVIDER` 后端) |
| `pi` | pi headless/SDK 一次性调用 | pi SDK 脚本或 `pi` 非交互模式 | 原生技能系统(与本宿主相同) | 原生 `available_skills` + transcript | 作为宿主可用;headless 适配器 = P2 |

`--runner auto` 的探测顺序:`inhost`(始终可用)→ `codex` → `opencode` → `claude` → `pi`。探测失败(10 秒超时的 `complete("reply ok")`)的后端直接跳过。

## 触发探测策略

| 策略 | 机制 | 测量对象 | 保真度 | 需要 |
|---|---|---|---|---|
| `native` | 技能装进工具自己的技能目录;真实会话;解析 transcript 中的技能使用 | 真实触发行为 | 最高 | `skill_used` 可观测的后端 |
| `explicit` | 任务明示"先读技能 X" | 技能在场时的执行质量 | 中 | 任意 `run_agent` |
| `simulated` | 判定器只看到 name+description 列表 + 一条查询;返回它会咨询哪个(些)技能 | description 能否赢得咨询决策 | 对 description 调优足够 | 任意 `complete()` |

保真度告警:`simulated` 看不到技能被加载**之后**发生什么——执行质量的问题属于执行器运行(步骤 6),不属于触发评测。实证依据:一次 simulated 判定(每技能 20 条查询,全新上下文判定器)正确识别了一个技能的 20/20 触发行为,并在另一个技能中暴露了 3 个真实的漏触发,每个漏触发都源于只写在正文中(触发时不可见)的能力。

### Simulated 判定器提示词模板

```text
You are simulating the skill-triggering decision of an AI coding assistant.
The assistant has the following available_skills list (name + description ONLY —
bodies are not visible at decision time):

- <name>: <description>
...

For EACH user query below, decide which skill(s) the assistant would consult
based strictly on the descriptions. If no skill clearly matches, answer ["none"].
Return ONLY a JSON array: [{"id": 1, "consult": ["skill-name"]}, ...]

Queries:
1. <query>
...
```

规则:判定器以全新上下文运行(绝不继承撰写技能的对话);查询顺序打乱;判定器永远看不到 `should_trigger` 标签;边界模糊的"这会触发吗?"查询最有价值——运行前从集合中剔除显然无关的负例。

## 触发评测集契约

技能根目录下的 `trigger-evals.json`:

```json
[
  {"id": 1, "query": "真实用户查询", "should_trigger": true},
  {"id": 2, "query": "近似误触发查询", "should_trigger": false}
]
```

20 条查询:8–10 条 should-trigger(措辞多样,部分从不点名技能)+ 8–10 条近似误触发 should-not(共享关键词、邻近领域、模糊表述)。

## 优化循环的移植契约(P2)

上游 `run_eval.py` / `run_loop.py` / `improve_description.py`(anthropics/skills)硬编码了 `claude -p`。移植计划——schema 与目录契约保持不变:

1. 新增 `scripts/runners/{base,claude,codex,opencode,pi,inhost}.py`,实现上述契约。
2. 给三个脚本贯穿 `--runner auto|inhost|claude|codex|opencode|pi` 旗标;`auto` 使用上述探测顺序。
3. `run_loop.py` 增加 `--strategy native|explicit|simulated`(默认 `simulated`);`native` 需要 `skill_used` 可观测的后端。
4. `--holdout` 60/40 划分、`--max-iterations`、按保留集选优的语义与上游保持一致。
5. 移植完成前,按 SKILL.md 步骤 6/8 手动运行循环——所有产物落在相同的 schema 中,结果保持可比。

## 污染规则(适用于所有策略)

执行器的 `without_skill` 运行不能有机会发现技能。当技能就在 agent 会搜索的目录树中(如 `skills/<name>/SKILL.md`)时,从剪除副本或中性 cwd 运行执行器。已确认的失败模式:baseline agent 在仓库内找到并读了技能文件,在输出中引用它,产出了完全符合规范的结果——把测量到的差值悄悄清零。
