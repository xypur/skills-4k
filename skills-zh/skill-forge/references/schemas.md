# JSON Schema

本文档定义 skill-creator 所使用的 JSON schema。

---

## evals.json

定义技能的评测集。位于技能目录下的 `evals/evals.json`。

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "用户的示例提示词",
      "expected_output": "预期结果的描述",
      "files": ["evals/files/sample1.pdf"],
      "expectations": [
        "输出包含 X",
        "技能使用了脚本 Y"
      ]
    }
  ]
}
```

**字段:**
- `skill_name`: 与技能 frontmatter 一致的名称
- `evals[].id`: 唯一整数标识符
- `evals[].prompt`: 要执行的任务
- `evals[].expected_output`: 对成功的可读描述
- `evals[].files`: 可选的输入文件路径列表(相对于技能根目录)
- `evals[].expectations`: 可验证陈述列表

---

## history.json

在 Improve 模式中追踪版本演进。位于工作区根目录。

```json
{
  "started_at": "2026-01-15T10:30:00Z",
  "skill_name": "pdf",
  "current_best": "v2",
  "iterations": [
    {
      "version": "v0",
      "parent": null,
      "expectation_pass_rate": 0.65,
      "grading_result": "baseline",
      "is_current_best": false
    },
    {
      "version": "v1",
      "parent": "v0",
      "expectation_pass_rate": 0.75,
      "grading_result": "won",
      "is_current_best": false
    },
    {
      "version": "v2",
      "parent": "v1",
      "expectation_pass_rate": 0.85,
      "grading_result": "won",
      "is_current_best": true
    }
  ]
}
```

**字段:**
- `started_at`: 改进开始时的 ISO 时间戳
- `skill_name`: 被改进技能的名称
- `current_best`: 最佳版本的标识符
- `iterations[].version`: 版本标识符(v0、v1……)
- `iterations[].parent`: 派生自的父版本
- `iterations[].expectation_pass_rate`: 评分得出的通过率
- `iterations[].grading_result`: "baseline"、"won"、"lost" 或 "tie"
- `iterations[].is_current_best`: 是否为当前最佳版本

---

## grading.json

评分 agent 的输出。位于 `<run-dir>/grading.json`。

```json
{
  "expectations": [
    {
      "text": "输出包含姓名 'John Smith'",
      "passed": true,
      "evidence": "在 transcript 步骤 3 中找到:'Extracted names: John Smith, Sarah Johnson'"
    },
    {
      "text": "电子表格单元格 B10 中有 SUM 公式",
      "passed": false,
      "evidence": "未创建电子表格。输出是一个文本文件。"
    }
  ],
  "summary": {
    "passed": 2,
    "failed": 1,
    "total": 3,
    "pass_rate": 0.67
  },
  "execution_metrics": {
    "tool_calls": {
      "Read": 5,
      "Write": 2,
      "Bash": 8
    },
    "total_tool_calls": 15,
    "total_steps": 6,
    "errors_encountered": 0,
    "output_chars": 12450,
    "transcript_chars": 3200
  },
  "timing": {
    "executor_duration_seconds": 165.0,
    "grader_duration_seconds": 26.0,
    "total_duration_seconds": 191.0
  },
  "claims": [
    {
      "claim": "表单有 12 个可填写字段",
      "type": "factual",
      "verified": true,
      "evidence": "在 field_info.json 中数得 12 个字段"
    }
  ],
  "user_notes_summary": {
    "uncertainties": ["使用了 2023 年数据,可能已过期"],
    "needs_review": [],
    "workarounds": ["对不可填写字段回退为文本叠加"]
  },
  "eval_feedback": {
    "suggestions": [
      {
        "assertion": "输出包含姓名 'John Smith'",
        "reason": "一份提到该姓名的幻觉文档同样能通过"
      }
    ],
    "overall": "断言只检查存在性,不检查正确性。"
  }
}
```

**字段:**
- `expectations[]`: 附证据的评分结果
- `summary`: 通过/失败的聚合计数
- `execution_metrics`: 工具使用与输出规模(来自执行器的 metrics.json)
- `timing`: 墙钟计时(来自 timing.json)
- `claims`: 从输出中提取并验证的断言
- `user_notes_summary`: 执行器标记的问题
- `eval_feedback`: (可选)对评测集的改进建议,仅当评分器发现有值得提出的问题时出现

---

## metrics.json

执行器 agent 的输出。位于 `<run-dir>/outputs/metrics.json`。

```json
{
  "tool_calls": {
    "Read": 5,
    "Write": 2,
    "Bash": 8,
    "Edit": 1,
    "Glob": 2,
    "Grep": 0
  },
  "total_tool_calls": 18,
  "total_steps": 6,
  "files_created": ["filled_form.pdf", "field_values.json"],
  "errors_encountered": 0,
  "output_chars": 12450,
  "transcript_chars": 3200
}
```

**字段:**
- `tool_calls`: 各工具的调用次数
- `total_tool_calls`: 所有工具调用总数
- `total_steps`: 主要执行步骤数
- `files_created`: 已创建的输出文件列表
- `errors_encountered`: 执行期间的错误次数
- `output_chars`: 输出文件总字符数
- `transcript_chars`: transcript 字符数

---

## timing.json

一次运行的墙钟计时。位于 `<run-dir>/timing.json`。

**捕获方法:** 子代理任务完成时,任务通知会包含 `total_tokens` 与 `duration_ms`。立即保存——它们不会持久化到任何其他地方,事后无法恢复。

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3,
  "executor_start": "2026-01-15T10:30:00Z",
  "executor_end": "2026-01-15T10:32:45Z",
  "executor_duration_seconds": 165.0,
  "grader_start": "2026-01-15T10:32:46Z",
  "grader_end": "2026-01-15T10:33:12Z",
  "grader_duration_seconds": 26.0
}
```

---

## benchmark.json

Benchmark 模式的输出。位于 `benchmarks/<timestamp>/benchmark.json`。

```json
{
  "metadata": {
    "skill_name": "pdf",
    "skill_path": "/path/to/pdf",
    "executor_model": "claude-sonnet-4-20250514",
    "analyzer_model": "most-capable-model",
    "timestamp": "2026-01-15T10:30:00Z",
    "evals_run": [1, 2, 3],
    "runs_per_configuration": 3
  },

  "runs": [
    {
      "eval_id": 1,
      "eval_name": "Ocean",
      "configuration": "with_skill",
      "run_number": 1,
      "result": {
        "pass_rate": 0.85,
        "passed": 6,
        "failed": 1,
        "total": 7,
        "time_seconds": 42.5,
        "tokens": 3800,
        "tool_calls": 18,
        "errors": 0
      },
      "expectations": [
        {"text": "...", "passed": true, "evidence": "..."}
      ],
      "notes": [
        "使用了 2023 年数据,可能已过期",
        "对不可填写字段回退为文本叠加"
      ]
    }
  ],

  "run_summary": {
    "with_skill": {
      "pass_rate": {"mean": 0.85, "stddev": 0.05, "min": 0.80, "max": 0.90},
      "time_seconds": {"mean": 45.0, "stddev": 12.0, "min": 32.0, "max": 58.0},
      "tokens": {"mean": 3800, "stddev": 400, "min": 3200, "max": 4100}
    },
    "without_skill": {
      "pass_rate": {"mean": 0.35, "stddev": 0.08, "min": 0.28, "max": 0.45},
      "time_seconds": {"mean": 32.0, "stddev": 8.0, "min": 24.0, "max": 42.0},
      "tokens": {"mean": 2100, "stddev": 300, "min": 1800, "max": 2500}
    },
    "delta": {
      "pass_rate": "+0.50",
      "time_seconds": "+13.0",
      "tokens": "+1700"
    }
  },

  "notes": [
    "断言 'Output is a PDF file' 在两种配置下都 100% 通过——可能无法区分技能价值",
    "Eval 3 方差很高(50% ± 40%)——可能不稳定或依赖模型",
    "without_skill 运行始终在表格提取类断言上失败",
    "技能平均增加 13 秒执行时间,但将通过率提高了 50%"
  ]
}
```

**字段:**
- `metadata`: 本次基准运行的信息
  - `skill_name`: 技能名称
  - `timestamp`: 基准运行时间
  - `evals_run`: eval 名称或 ID 列表
  - `runs_per_configuration`: 每种配置的运行次数(如 3)
- `runs[]`: 单次运行结果
  - `eval_id`: eval 的数字标识符
  - `eval_name`: 可读的 eval 名称(viewer 中作为节标题)
  - `configuration`: 必须是 `"with_skill"` 或 `"without_skill"`(viewer 用这个精确字符串做分组与着色)
  - `run_number`: 整数运行序号(1、2、3……)
  - `result`: 嵌套对象,含 `pass_rate`、`passed`、`total`、`time_seconds`、`tokens`、`errors`
- `run_summary`: 每种配置的统计聚合
  - `with_skill` / `without_skill`: 各含 `pass_rate`、`time_seconds`、`tokens` 对象,带 `mean` 与 `stddev` 字段
  - `delta`: 差值字符串,如 `"+0.50"`、`"+13.0"`、`"+1700"`
- `notes`: 分析师的自由格式观察

**重要:** viewer 按精确字段名读取。把 `configuration` 写成 `config`,或把 `pass_rate` 放在运行顶层而非 `result` 下,viewer 都会显示空值/零值。手动生成 benchmark.json 时务必对照本 schema。

---

## comparison.json

盲比较器的输出。位于 `<grading-dir>/comparison-N.json`。

```json
{
  "winner": "A",
  "reasoning": "输出 A 提供了完整方案,格式规范且字段齐全。输出 B 缺少日期字段且格式不一致。",
  "rubric": {
    "A": {
      "content": {
        "correctness": 5,
        "completeness": 5,
        "accuracy": 4
      },
      "structure": {
        "organization": 4,
        "formatting": 5,
        "usability": 4
      },
      "content_score": 4.7,
      "structure_score": 4.3,
      "overall_score": 9.0
    },
    "B": {
      "content": {
        "correctness": 3,
        "completeness": 2,
        "accuracy": 3
      },
      "structure": {
        "organization": 3,
        "formatting": 2,
        "usability": 3
      },
      "content_score": 2.7,
      "structure_score": 2.7,
      "overall_score": 5.4
    }
  },
  "output_quality": {
    "A": {
      "score": 9,
      "strengths": ["方案完整", "格式规范", "字段齐全"],
      "weaknesses": ["页眉风格略有不一致"]
    },
    "B": {
      "score": 5,
      "strengths": ["输出可读", "基本结构正确"],
      "weaknesses": ["缺少日期字段", "格式不一致", "数据提取不完整"]
    }
  },
  "expectation_results": {
    "A": {
      "passed": 4,
      "total": 5,
      "pass_rate": 0.80,
      "details": [
        {"text": "输出包含姓名", "passed": true}
      ]
    },
    "B": {
      "passed": 3,
      "total": 5,
      "pass_rate": 0.60,
      "details": [
        {"text": "输出包含姓名", "passed": true}
      ]
    }
  }
}
```

---

## analysis.json

事后分析器的输出。位于 `<grading-dir>/analysis.json`。

```json
{
  "comparison_summary": {
    "winner": "A",
    "winner_skill": "path/to/winner/skill",
    "loser_skill": "path/to/loser/skill",
    "comparator_reasoning": "比较器选择获胜方的理由摘要"
  },
  "winner_strengths": [
    "对多页文档处理有清晰的分步指导",
    "附带了校验脚本,捕获了格式错误"
  ],
  "loser_weaknesses": [
    "'适当处理文档'这类含糊指令导致行为不一致",
    "没有校验脚本,agent 只能即兴发挥"
  ],
  "instruction_following": {
    "winner": {
      "score": 9,
      "issues": ["轻微:跳过了可选的日志步骤"]
    },
    "loser": {
      "score": 6,
      "issues": [
        "未使用技能的格式模板",
        "自创方法而未遵循步骤 3"
      ]
    }
  },
  "improvement_suggestions": [
    {
      "priority": "high",
      "category": "instructions",
      "suggestion": "把'适当处理文档'替换为明确的步骤",
      "expected_impact": "消除导致行为不一致的歧义"
    }
  ],
  "transcript_insights": {
    "winner_execution_pattern": "读技能 -> 遵循 5 步流程 -> 使用校验脚本",
    "loser_execution_pattern": "读技能 -> 方法不明 -> 尝试了 3 种不同方法"
  }
}
```
