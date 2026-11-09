# 验收门：check.py + thresholds.json

## 结构

```
tools/
├── check.py           # 唯一入口：python tools/check.py <stage>
├── thresholds.json    # 所有阈值，人可改，agent 禁改
└── ...                # 各任务脚本
```

`check.py` 只做三件事：读阈值、跑断言、打印 `[ok]/[fail]/[info]` 并以退出码返回结论。**不要**在这里做实际处理。

## 最小骨架

```python
import json, os, sys

ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
INTER = os.path.join(ROOT, "_中间产物")

def thresholds():
    with open(os.path.join(ROOT, "tools", "thresholds.json"), encoding="utf-8") as f:
        return json.load(f)

def check_manifest() -> bool:
    th = thresholds()["manifest"]
    rows = [r for r in open(os.path.join(INTER, "manifest.csv"), encoding="utf-8-sig")]
    ok = True
    print(f"[info] manifest 行数 {len(rows)}")
    if len(rows) >= th["min_rows"]:
        print(f"[ok] 行数 ≥ {th['min_rows']}")
    else:
        print(f"[fail] 行数 < {th['min_rows']}")
        ok = False
    return ok

def check_env() -> bool:
    # 依赖是否装齐、目录是否存在、编码是否可用
    ...

STAGES = {
    "env": check_env,
    "manifest": check_manifest,
}

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("用法: python tools/check.py <stage>，可选: " + " ".join(STAGES))
        sys.exit(2)
    sys.exit(0 if STAGES[sys.argv[1]]() else 1)
```

## thresholds.json

```json
{
  "manifest": { "min_rows": 2000, "_note": "10-29 实测 2306 行，门槛取 95%" },
  "extract_docx": { "min_success_rate": 0.95, "max_empty_rate": 0.05 },
  "corpus": { "min_docs": 1500, "min_chars": 15000000,
              "_note": "原设 1800 不可达（sha1 去重 + 依赖剪枝后只有 1575），10-29 人批准改 1500" }
}
```

规矩：
- 每个数字都有 `_note` 说明来历；
- 只有人能改，改的时候追加日期与理由；
- `check.py` 里**不许**出现裸数字。

## 断言要测产出

差：`print("提取完成")`。
好：
- 行数与阶段 1 的清单数一致；
- 成功率 ≥ 阈值；
- 每条字段非空、类型正确；
- 去重率、空文本率在范围内；
- 兜底正则（PII、超长、乱码）0 命中。

## 反作弊（人做的检查）

| 检查 | 命令 | 期望 |
|---|---|---|
| 阈值没被改 | `git log -p -- tools/thresholds.json` | 只有人批准的提交 |
| 断言没被削弱 | `git log -p -- tools/check.py` | 只新增（diff 里 `-` 行为空） |
| 没有跳步 | `git log --oneline` | 一任务一提交，任务号连续 |
| 没有只处理前 N 条 | 抽样跑一遍脚本 | 输出行数与全量一致 |

允许 agent 新增 stage，不允许它修改已有断言——这条要在夜间提示词里写死。

## 阈值不可达怎么办

实测 1,575 篇，方案阶段估 2,800，验收写 1,800：按纪律任务必然 blocked 并连坐后续。正确做法不是让 agent 自己降，而是：
1. 它 blocked 并报数字（1,575）；
2. 人确认"不能为凑数拉进噪声数据"；
3. 人改 `thresholds.json` 为 1,500（=1,575×95%），写理由，提交；
4. 下一条提示词里告诉它新阈值从哪读。

这条经验说明：**阈值必须来自实测，方案阶段的估数是假设**。
