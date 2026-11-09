# 模板

## 1. TASKS.md 任务条目

```markdown
### T04 提取 docx 正文（含表格）
- 依赖：T02 产出 manifest.csv
- 做什么：对 manifest 里 status=included 且 ext=docx 的条目，抽段落 + 表格单元格，写 `_中间产物/corpus/docx.jsonl`（字段 path/text/chars/err）
- 验收：`python tools/check.py extract-docx`
  - 行数 = manifest 里 docx 的 included 数
  - 成功率 ≥ 95%
  - 每条 chars > 0（空文档另行统计，不超过 5%）
  - 无 18 位数字（PII 兜底）
- blocked：python-docx 装不上；或成功率 < 80%（说明文件损坏率高，需要换抽取方式）
- 状态：[todo]
```

任务状态只有四种：`[todo] [doing] [done] [blocked]`。改成 done 的唯一依据是验收命令退出码 0。

## 2. PROGRESS.md 条目

```markdown
## 2026-10-30 夜（agent）

- 02:34 T06 done 3min，338 篇全覆盖，文本层 230 / 扫描件 108（32%），chars 中位数 4,122
- 02:43 T07 done 12min，pptx 67/67、xlsx 380/380，xlsx 只取字符串单元格
- 03:20 T09 blocked 47min，候选池在阈值内只有 6,283 条，判完仍差 762，需要人决定是否放宽候选闸
```

规则：
- 一行一个任务，格式 `HH:MM Txx 状态 耗时 关键数字`；
- **保留数字**（行数、成功率、耗时），不要写"完成了"；
- blocked 要写"卡在什么数字上、需要人做什么决定"；
- 时间戳先跑 `date` 再写。

## 3. 夜间提示词

```text
你是执行 agent。项目目录：<绝对路径>
先跑 pwd、git rev-parse --show-toplevel、git branch --show-current、git status，确认位置、分支、工作区。

【现状】
- HEAD <sha>（上一任务），分支 night/<日期>，工作区 <干净/有N处未提交>
- 已完成：T01–T05（见 PROGRESS.md）
- 今晚目标：T06 收尾 + T07–T12

【做法】
一次只做一个任务。开始前把 TASKS.md 里该任务读完整；做完跑它的验收命令：
- 退出码 0 → 任务状态改 [done]，`git add` 只加本任务文件，提交 `Txx: 简短说明`，PROGRESS 记一行（带数字）；
- 非 0 → 修，最多 3 次；3 次仍不过 → 状态改 [blocked]，把验收输出贴进 PROGRESS，跳过，继续下一个。

【红线】
- 不改 tools/thresholds.json；不改 check.py 已有断言（只能新增 stage）
- 不 add -A、不 commit -a、不 --no-verify、不 push、不 reset --hard 到更早提交
- 不删除或跳过验收来"通过"
- 收工前：git status 必须干净，打 bundle

【停止条件】
- 时间到 <HH:MM> 或所有今晚任务 done/blocked；
- 收工报告只写：HEAD、各任务状态、blocked 的原因与证据、明早需要人决定的 1–3 件事。
```

## 4. 夜与夜之间的交接

```markdown
## 交接（第二晚）

- 分支：night/2026-10-28；HEAD：<sha>
- 未提交改动：tools/check.py（新增某 stage 一半）、tools/extract_pdf.py（未跑）
- 今晚先收尾：把 extract_pdf 跑完并提交 T06，再进 T07
- 阈值变更：corpus.min_docs 1,800 → 1,500（10-29 白天，人批准，记在 thresholds._note）
- 上一晚 blocked：无 / T05（Word COM 残留 WINWORD，已处理）
```

## 5. 早间验收清单（15 分钟）

1. `git log --oneline -10`：是否一任务一提交、提交信息是否带任务号；
2. `git status`：是否干净；不干净说明它没走完流程；
3. 自己重跑 `python tools/check.py <stage>`：**不看它的总结**；
4. `git log -p -- tools/`：是否为空（空=阈值与断言没被动过）；
5. `PROGRESS.md` 时间戳 vs 提交时间：是否对得上；
6. 抽查一个 blocked：证据是否真的、还是借口。
