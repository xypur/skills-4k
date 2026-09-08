# skill-forge 设计与实现记录

> 2026-09-02 · 本文档记录从 skill-creator 审计到 skill-forge 落地的完整过程，
> 作为后续迭代（P2 运行器移植等）的决策依据。
>
> 迁移更新：skill-forge 及本文档已从 private-skills 迁至 `dev/coding/skills-4k`；
> 按该仓库的每技能独立协议政策补入 `skills/skill-forge/LICENSE`（Apache-2.0）。

## 1. 背景与动机

起点是对 `dev/coding/skills` 仓库做的一次 skill-creator 规范合规审计（2026-09-02）。
审计结论：写作与结构层面合规良好，但**评测环节缺失**（8 个技能 6 个无 evals）。
随后在执行补齐计划时暴露了两个事实：

1. **基准实验真实可跑**：用宿主（pi）原生子代理完成了 12 个运行
   （js-coding-style + imperative-commits × 3 eval × with/without），
   聚合出 benchmark 并生成了审阅页面——全程没有用到任何外部 CLI。
2. **上游 skill-creator 的核心循环在本机跑不通**：`run_loop.py` /
   `run_eval.py` / `improve_description.py` 硬编码 `claude -p` 子进程，
   而本机 `claude -p` 挂起（90s 超时、exit 124，`auth status` 却显示已登录）。

同时，触发优化走了一条**降级路线并意外成功**：用 fresh 上下文的判定器
仅凭 name+description 逐查询判定"会咨询哪个技能"，20 条查询/技能，
准确发现 `vue-component-authoring` 的 3 个漏报——根因是三个能力
（副作用清理、attrs 透传、SSR）只写在正文、触发时不可见，据此修正了
description。这条被实证有效的路径，成为新设计的核心。

## 2. 依赖解剖：skill-creator 到底卡在哪

| 依赖点 | 用途 | 写死程度 | 结论 |
|---|---|---|---|
| `claude -p` 子进程 | description 改写（一次性 LLM 调用）、触发探测（真实会话） | 硬编码在 `run_eval.py` / `run_loop.py` / `improve_description.py` | 唯一真正的耦合点 |
| 子代理派生 | with/without 执行器运行 | 写在 SKILL.md 叙述里，机制因宿主而异 | pi / Claude Code 有原生能力；codex/opencode 走外部 CLI |
| Python 脚本（aggregate/viewer/schemas） | 聚合、可视化、schema | 纯 Python，无内部包依赖 | 天然可移植，原样继承 |
| 评测产物布局（workspace / grading.json / benchmark.json） | — | 与工具无关 | 保留 |

结论：真正被写死的只有**"一次性问模型"**与**"带技能跑一个 agent 会话"**
两个原语；其余约 80% 可直接复用。

## 3. 核心设计

### 3.1 Runner 抽象（3 个原语）

```python
class Runner:
    def available(self) -> bool            # CLI 在 PATH + 已认证
    def complete(prompt, model?) -> str    # 一次性文本：改写 description、LLM 评分、模拟判定
    def run_agent(task, cwd, skill_dirs?) -> RunResult
                                           # 带可选技能的会话：触发探测 + 执行器运行
```

后端适配器：`inhost`（宿主自身能力，**默认路径，恒可用**）、`claude`、
`codex`、`opencode`、`pi`。探测顺序 `inhost → codex → opencode → claude → pi`。

### 3.2 触发探测三级策略（保真度递减）

| 策略 | 机制 | 保真度 | 需要 |
|---|---|---|---|
| `native` | 技能装进工具原生技能目录，解析 transcript | 最高 | claude、pi |
| `explicit` | 任务明示"先读技能 X"，测执行质量 | 中 | 任意 `run_agent` |
| `simulated` | 判定器只看 name+description 列表，逐查询判定 | 对 description 调优足够 | 任意 `complete()` |

`simulated` 的保真度上限：看不到技能加载后的执行行为——执行质量问题
归属执行器运行（Step 6），两层各司其职。

### 3.3 污染规则（本次实验的教训）

12 运行基准发现：baseline 的 without_skill 运行**在仓库里自行发现并读取了
被测技能**（transcript 明确引用"遵循 js-coding-style 技能"），导致 4/6 条
eval 的 delta 被静默归零。规则成文：**baseline 必须从剪除技能的中性 cwd 运行**。

### 3.4 基准目录布局修正

实测发现 `aggregate_benchmark.py` 要求 `eval-*/{with,without}_skill/run-1/`
的 `run-N` 层；`grading.json` 的字段名（`expectations[{text,passed,evidence}]`）
为 viewer 硬依赖。均已写入 skill-forge 的 SKILL.md 与 schemas.md。

## 4. 命名与落点决策

- **不沿用 `skill-creator`**：上游副本被 `skills-lock.json` 管理（同名会被
  覆盖/校验失败）；且高重叠 description 共存会造成触发冲突。
- **命名 `skill-forge`**：覆盖"锻造 + 验证"双重含义。
- **落点 `skills-4k`**（`~/dev/coding/skills-4k`）：与公开技能仓库分离的收藏仓库；
  该仓库采用**每技能独立协议**（刻意无根协议），衍生技能原样保留上游协议，
  故 `skill-forge` 目录携带 Apache-2.0 `LICENSE`；按 canon + 中文镜像双落盘。
- **与 skill-dev 的分工边界**（解决触发面冲突）：skill-dev = 创作/审查/入库
  仓库 pack；skill-forge = 脚手架、基准、触发调优。两者 description 已各自
  去重。

## 5. 实现清单

```
skills-4k/skills/skill-forge/
├── SKILL.md                        # 新写，159 行（上游 485 行蒸馏 + 新设计）
├── LICENSE                         # Apache-2.0（继承上游，per-skill licensing）
├── GENERATION.md                   # fork 来源、理由、继承清单
├── CHANGES.md                      # 初始记录
├── evals.json                      # 3 条自测提示词（自举）
├── references/
│   ├── schemas.md                  # 原样继承（评测 schema 权威定义）
│   └── backends.md                 # ★ 新写：运行器契约、后端表、策略分级、P2 计划
└── scripts/
    ├── aggregate_benchmark.py      # 原样继承（已验证独立运行）
    ├── generate_review.py          # 原样继承（viewer.html 相对路径定位，已验证）
    ├── viewer.html                 # 原样继承
    └── NOTICE.md                   # Apache-2.0 归属（上游许可要求）
```

中文镜像：`skills-zh/skill-forge/SKILL.zh.md`（与英文逐行对齐：159/159 行、15/15 标题）。

新写入内容要点：

- 第 5 条核心原则 **"默认宿主无关"**——工作流零厂商调用，外部能力全部
  经运行器层或宿主内路径；
- 触发策略表、`trigger-evals.json` 契约（20 条 = 8–10 正例 + 8–10 近似负例）、
  模拟判定器提示词模板（本会话实战模板的通用化）；
- 上游 485 行中的 Claude.ai / Cowork 环境分支全部裁除；
- 评测目录布局按实测要求修正（`run-N` 层）。

## 6. 验证记录

| 项目 | 方法 | 结果 |
|---|---|---|
| 搬运脚本可独立运行 | `aggregate_benchmark.py --help`、`generate_review.py --help`、`ast.parse` | passed |
| evals.json 自举 | `json.load` | passed（3 evals） |
| 中英镜像对齐 | `wc -l` + `grep -c '^#'` | passed（159/159 行、15/15 标题） |
| 正文长度约束 | `wc -l` < 500 | passed（159 行） |
| 上游可用性假设 | 本机 `claude -p` 探测 | failed（挂起，exit 124）——已如实记入 backends.md |

## 7. 遗留事项（P2 及后续）

1. **运行器移植（P2）**：实现 `scripts/runners/{base,claude,codex,opencode,pi,inhost}.py`，
   为 `run_eval` / `run_loop` / `improve_description` 加 `--runner` 标志；
   契约已在 `references/backends.md` 固化，schema 不变。移植前按 SKILL.md
   步骤 6/8 手动跑，产物 schema 相同、结果可比。
2. **旧 skill-creator 去留**：建议 skill-forge 实际投入使用后，将
   `.agents/skills/skill-creator` 移除或降级至 `.agents/unskills/`，
   消除 description 触发冲突。
3. **skill-dev 安装副本过期**：`dev/coding/skills/.agents/unskills/skill-dev`
   落后 pack 侧一个版本，需定同步策略。
4. **仓库脚手架（已完成）**：skills-4k 已初始化（master 分支）并按当前项目模式补齐
   `AGENTS.md` / `AGENTS.zh.md` / `README.md` / `README.zh.md` / `.gitignore`；
   待初始提交。
