# 04 用 Python 写脚本

这篇不讲语法，讲"怎么写出一个别人（包括三个月后的你）能用的脚本"。语法随便找本
书看。

## 1. 一个能用的脚本长什么样

很多人写的脚本是这样：

```python
import os
import sys

f = open(sys.argv[1])
data = f.read()
lines = data.split('\n')
for i in range(len(lines)):
    print(lines[i])
```

能跑，但问题一堆：文件名写死参数位置、文件没关、没法加参数。我改一版：

```python
#!/usr/bin/env python3
"""统计文件行数。"""

import argparse
import logging
import sys
from pathlib import Path

log = logging.getLogger("linecount")


def count_lines(path: Path) -> int:
    """返回文件行数。空文件算 0 行。"""
    with path.open(encoding="utf-8", errors="replace") as f:
        return sum(1 for _ in f)


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="统计文件行数")
    parser.add_argument("paths", nargs="+", type=Path, help="要统计的文件")
    parser.add_argument("-v", "--verbose", action="store_true")
    args = parser.parse_args(argv)

    logging.basicConfig(
        level=logging.DEBUG if args.verbose else logging.INFO,
        format="%(levelname)s %(message)s",
    )

    total = 0
    failed = False
    for p in args.paths:
        try:
            n = count_lines(p)
        except OSError as exc:
            # 一个文件读不了不该让整个脚本挂掉
            log.error("读不了 %s: %s", p, exc)
            failed = True
            continue
        print(f"{n:>8}  {p}")
        total += n

    print(f"{total:>8}  合计")
    return 1 if failed else 0


if __name__ == "__main__":
    sys.exit(main())
```

几个点值得说：

- **`if __name__ == "__main__"`**：这样这个文件既能当脚本跑，也能被 import。
- **`sys.exit(main())`**：返回退出码。成功 0，失败非 0。shell 里 `&&` 靠这个判断。
- **`argparse`**：自动生成 `--help`，参数有类型检查。
- **`logging` 而不是 `print` 打错误**：print 的信息没法关掉，logging 能分级。
- **用 `pathlib.Path` 不用 `os.path`**：路径拼接更清楚，`path / "sub" / "file"`。

## 2. 读写文件

```python
from pathlib import Path

# 读整个文件
text = Path("data.txt").read_text(encoding="utf-8")

# 按行读，大文件用这个（不会一次性读进内存）
with Path("big.log").open() as f:
    for line in f:
        line = line.rstrip("\n")

# 写
Path("out.txt").write_text("hello\n", encoding="utf-8")

# 追加
with Path("out.txt").open("a", encoding="utf-8") as f:
    f.write("more\n")
```

**永远显式写 `encoding="utf-8"`。** 不写的话，在 Windows 上 Python 会用
GBK，读 UTF-8 文件直接 `UnicodeDecodeError`。这个坑我在 Windows 上踩过，
报错信息是：

```
UnicodeDecodeError: 'gbk' codec can't decode byte 0xe4 in position 0
```

看到 `'gbk' codec` 基本就是这个原因。

## 3. 处理 JSON 和 CSV

```python
import json
import csv
from pathlib import Path

# JSON
data = json.loads(Path("config.json").read_text(encoding="utf-8"))
Path("out.json").write_text(
    json.dumps(data, ensure_ascii=False, indent=2),
    encoding="utf-8",
)

# CSV
with Path("data.csv").open(encoding="utf-8", newline="") as f:
    for row in csv.DictReader(f):
        print(row["name"], row["age"])
```

**`ensure_ascii=False`** 很重要，不然中文会被转成 `\u4f60\u597d` 这种，
人能看但很难受。读 JSON 时注意，`json.loads` 失败会抛 `json.JSONDecodeError`，
配置文件被人手改坏了很常见，包一层比较好。

**CSV 的 `newline=""`** 是官方文档要求的，不加的话在某些系统上会多出空行。

## 4. 调外部命令

```python
import subprocess

result = subprocess.run(
    ["git", "rev-parse", "--short", "HEAD"],
    capture_output=True,
    text=True,        # 输出是 str 不是 bytes
    check=False,      # 非 0 退出码不要抛异常，我自己判断
    timeout=10,       # 超时保护
)
if result.returncode != 0:
    raise RuntimeError(f"git 失败: {result.stderr}")
commit = result.stdout.strip()
```

**坑一**：不要用 `shell=True`，除非你真的需要 shell 特性。`shell=True` 加用户
输入就是命令注入漏洞。用列表形式传参数，Python 会正确处理转义。

**坑二**：`subprocess.run` 默认不设 timeout，外部命令卡住你的脚本也卡住。
尤其是调网络命令，一定加 timeout。

## 5. 日志怎么用

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)-7s %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
log = logging.getLogger(__name__)

log.debug("只有 -v 时才看得到")
log.info("正常信息")
log.warning("有点不对劲")
log.error("出错了")
```

**别用 f-string 拼日志**：

```python
log.info(f"处理 {count} 个文件")   # 不管日志级别，字符串都会先拼出来
log.info("处理 %d 个文件", count)  # 只有真的要输出时才格式化
```

单条无所谓，循环里差很多。

## 6. 让脚本能直接执行

```bash
chmod +x myscript.py
./myscript.py --help
```

前提是文件第一行有 shebang：`#!/usr/bin/env python3`。

## 7. 最后一点建议

脚本超过 200 行就该考虑拆成模块了。我见过一个 2000 行的"脚本"，谁都不敢动。
判断标准很简单：**如果改一个功能需要通读全文，就该拆了。**

下一篇：[部署入门](05-部署入门.md)。
