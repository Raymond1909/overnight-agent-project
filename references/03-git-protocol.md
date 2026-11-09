# Git 协议

## 分支与提交

- 主分支 `main`，只有人合并。
- 每晚一条 `night/YYYY-MM-DD`，从 main 切出；早上验收通过再合并，失败就丢弃。
- 一次提交只对一个任务：`T04: 提取 docx 正文（含表格）`。
- 提交前 `git status` 确认只包含本任务文件；禁 `git add -A`、`git commit -a`。

## 回滚

| 场景 | 命令 |
|---|---|
| 任务失败，丢掉未提交改动 | `git checkout -- .` |
| 任务已提交但验收不过 | `git revert --no-edit <sha>`（保留历史，别 `reset --hard`） |
| 整晚报废 | 切回 main：`git checkout main`，`night/日期` 分支留着备查 |

`reset --hard` 会连别人的改动一起吞掉，只在你自己独占分支、且明确知道后果时用。

## 云同步目录放工作树、.git 放本地盘

OneDrive / Dropbox / 坚果云会锁文件、产生 `xxx (1).git` 冲突副本。做法：

```bash
mkdir -p /d/code/myproj
git init -b main --separate-git-dir=/d/code/myproj/myproj.git
```

工作树留在同步目录（文档可见、可备份），`.git` 实体在本地盘。注意：
- `--separate-git-dir` 的父目录必须先存在，否则 `fatal: Invalid path`；
- 初始化后工作树里的 `.git` 是一个文本文件：`gitdir: /d/code/myproj/myproj.git`；
- 有些 git 版本默认分支仍是 `master`，用 `-b main` 显式指定；
- 中文文件名显示成 `\346\225\231` 时：`git config core.quotepath false`；
- Windows 上避免行尾警告：`git config core.autocrlf false`，并对 `.txt`/`.csv` 设 `-text`。

## 备份（本地盘不在同步范围）

```bash
git bundle create "_备份/myproj-$(date +%F).bundle" --all
git bundle verify "_备份/myproj-2026-10-30.bundle"
```

bundle 是单文件，天然适合网盘/邮件/移动硬盘；恢复用 `git clone <bundle> myproj`。

## pre-commit 钩子

钩子路径要用 `git rev-parse --absolute-git-dir` 定位（`--separate-git-dir` 下 `.git` 只是指针文件）。

```sh
#!/bin/sh
python tools/check.py precommit || exit 1
```

`precommit` stage 至少查：产物里没有 PII、没有大文件、没有 `_中间产物/` 被加进索引。

## .gitignore 常见项

```gitignore
_中间产物/
_备份/*.bundle
__pycache__/
*.pyc
.venv/
*.jsonl
*.log
```

## 不要 push 的仓库

工作树里含真实数据、人名、绝对路径时，**不要加 remote**。要开源就另起干净仓库（详见发布清单：只搬脚本、文档、成品，先扫脱敏）。
