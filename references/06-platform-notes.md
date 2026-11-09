# 平台与工程注意（以 Windows 实测为准）

## 编码

- Python 在中文 Windows 上 `sys.stdout.encoding` 常是 `gbk`，`print` 中文会抛 `UnicodeEncodeError` 或乱码。开工第一条：终端设 `PYTHONIOENCODING=utf-8`。
- 读写文件一律显式 `encoding="utf-8"`；用 Excel 打开过的 CSV 可能带 BOM，读时用 `utf-8-sig`。
- 输出给外部工具的 `.txt` 若对方要 GBK，同时产 UTF-8 与 GBK 两份，逐行比对一致（只差编码）。

## Office 文档

- `python-docx` 的 `document.paragraphs` **不含表格单元格**。只读段落会漏掉大量正文（实测一份方案文档只抽到 73 字符，全是表格横线）。要同时遍历 `document.tables` 的每个 cell。
- 老 `.doc` 用 Word COM 转 `.docx` 再抽：Word 不接受正斜杠路径，`D:/x.docx` 会被转成 `%20` 之类的错误；先 `os.path.normpath` 换成反斜杠。
- COM 会残留 `WINWORD.EXE`，收工前查 `tasklist`；它的输出是 GBK，需 `decode("gbk")`，否则判断失效。
- 大批量转换慢且吃内存，分批 + 失败清单重试。

## 长任务

- 防睡眠：`powercfg /change standby-timeout-ac 0`、`hibernate-timeout-ac 0`（需管理员；图形界面改更稳）。
- 关掉 Word/Excel 等会锁文件的程序，尤其 COM 任务。
- 超时：前台命令有上限，长任务放后台并留日志；注意有些环境把"进程被杀"报成 exit 1 而不是信号。

## 云盘目录

- OneDrive/Dropbox 的"按需下载"会让第一次读取极慢（实测 220 篇 docx 超过 5 分钟），I/O 是瓶颈而不是 CPU：分批 + 增量写 JSONL + 断点续跑。
- 云盘会锁 `.git` 里的碎文件，导致冲突副本：`.git` 用 `--separate-git-dir` 放本地盘（见 03）。
- 大目录扫描（41,995 文件 / 59 GB）先做 manifest 再处理，不要边扫边跑。

## 依赖

开工先做 `env` stage：逐项 import 并打印版本，缺的当场 `pip install`，装不上就 blocked，不要带着半个环境跑一夜。

## 外部工具的上限

如果用第三方工具做最终交付（输入法词库、导入接口），它的限制必须实测并写进文档：文件格式、编码、单文件大小、去重规则（是否区分大小写、是否裁空格）、单次条数上限。实测方法：造一个小样本文件（含重复、大小写、空格、超长、特殊字符），导入后看它返回的计数和列表，把结论写成"规则表 + 实测记录表"。
