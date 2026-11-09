# overnight-agent-project

给 AI 编码 agent 用的一夜工作流：把长任务拆成 30–60 分钟的小任务，每个任务有条能跑过的验收命令，进度写进文件，第二天早上 15 分钟验收。

适配 Claude Code、Codex、Cursor、Gemini CLI、zcode 等能读写文件、能跑命令的 agent。

## 它解决什么

agent 跑长任务翻车，通常不是模型不行，是下面四件事没做：

| 症状 | 根因 | 本技能的解法 |
|---|---|---|
| 第二天不知道做到哪 | 进度只在对话里 | `TASKS.md` + `PROGRESS.md` 落盘 |
| "完成了"但产物是空的 | 没有可执行的验收 | 每个任务一条 `check.py <stage>` |
| 半夜把测试改松了 | 验收标准可被改 | `thresholds.json` 冻结 + `git log -p -- tools/` 必空 |
| 卡住不说、假装在推进 | 没有 blocked 纪律 | 失败 3 次即 blocked，写证据，跳过 |

## 目录

```
├── SKILL.md                      # 主文件：何时用、四不变式、工作流、失败模式
└── references/
    ├── 01-templates.md           # TASKS / PROGRESS / 夜间提示词 / 交接 / 早间清单
    ├── 02-verification-gates.md  # check.py 骨架、thresholds.json、断言设计、反作弊
    ├── 03-git-protocol.md        # 分支、一任务一提交、回滚、bundle、云盘 .git
    ├── 04-handover-and-blocked.md  # blocked 三条件、交接字段、时间戳纪律
    ├── 05-llm-judging.md         # LLM 逐条判定、漂移测量、人定标准
    └── 06-platform-notes.md      # Windows 编码、Office 抽取、云盘 I/O、防睡眠
```

## 安装

技能就是一组 Markdown，没有依赖。把它放到所用 agent 能读到的地方即可。先克隆仓库：

```bash
git clone https://github.com/Raymond1909/overnight-agent-project.git
```

**Claude Code**

```bash
cp -r overnight-agent-project ~/.claude/skills/overnight-agent-project
```

**Codex**

```bash
cp -r overnight-agent-project ~/.codex/skills/overnight-agent-project
```

或在该项目的 `AGENTS.md` 里加一行引用，指向你实际放置的位置：`- 长任务流程见 <放置路径>/SKILL.md`。

**Cursor**

放到 `.cursor/rules/overnight-agent-project/`，或把 `SKILL.md` 转成 `.mdc`（加 frontmatter `description:` 与 `globs:`）。

**Gemini CLI / 其他支持 skill 目录的工具**

```bash
cp -r overnight-agent-project ~/.gemini/skills/overnight-agent-project
```

**没有 skill 机制的工具**

把 `SKILL.md` 全文贴进系统提示或项目约定文件（`AGENTS.md`、`.cursorrules`、`CLAUDE.md`），references 留在仓库里，需要时让它自己读。

## 怎么用

1. **第一天（人做，30 分钟）**
   - 写 `TASKS.md`：每个任务 30–60 分钟，附带一条能失败的命令；
   - 写 `tools/check.py` 与 `tools/thresholds.json`，阈值来自实测而不是估计；
   - `git init`，建 `night/<日期>` 分支，先提交骨架。

2. **每晚（把提示词发给 agent）**
   用 `references/01-templates.md` 里的夜间提示词模板，填现状、目标任务、红线、停止条件。核心三条：
   - 一次只做一个任务；
   - 验收不过，修 3 次即 blocked；
   - 只 `add` 本任务文件，一任务一提交。

3. **每天早间（人做，15 分钟）**
   按 `references/01-templates.md` 的 6 步验收：看 `git log`、`git status`、自己重跑 `check.py`、确认 `tools/` 的 diff、对时间戳、抽查一个 blocked。

## 最小示例

```
project/
├── TASKS.md
├── PROGRESS.md
├── tools/
│   ├── check.py
│   └── thresholds.json
└── _中间产物/            # 不入 git
```

`TASKS.md`：

```markdown
### T01 骨架与验收
- 做什么：建目录、requirements.txt、check.py 的 env stage
- 验收：`python tools/check.py env` 退出码 0
- blocked：依赖装不上（贴 pip 错误）
- 状态：[todo]
```

跑一夜之后，早上你要能看到：`T01: 骨架与验收` 这样的提交、`PROGRESS.md` 里一行带数字的记录、以及一个你自己重跑也通过的 `check.py env`。

## 适配与改造

- **任务粒度**：按你的验收命令耗时定；一条命令跑 10 分钟以上就再拆。
- **多 agent 并行**：每个 agent 一条分支、一个独立任务池，别让两个 agent 改同一个 `TASKS.md`。
- **判定的活**：规则筛选先跑到 70–80%，剩下的用 LLM 逐条判，见 `references/05-llm-judging.md`。
- **不适用**：一次性脚本、探索性调研、需要人实时决策的任务。

## 来源

这套流程来自一个真实项目的多夜实测（从 4.2 万个文件里抽语料、挖术语、交付输入法词库），包括三次翻车与修正：

- 验收阈值按方案估的 1,800 写，实测只有 1,575，任务必然 blocked → 阈值改由实测决定；
- 规则迭代 9 轮卡在 76%，换短语级挖掘仍 68%，最后靠 LLM 逐条判到可用；
- 盲判复检报 20% 翻转率，人工复核发现 19 条是正常数据被误删 → 翻转率不等于质量。

这些教训都写进了 references，能用就直接用，别重新踩一遍。

## 许可

MIT
