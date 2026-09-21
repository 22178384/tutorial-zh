# 02 · 用 Python 调用 HTTP API

本教程用标准库完成一次 HTTP 接口调用，无需安装第三方库。

## 一、最简单的 GET
```python
import urllib.request

with urllib.request.urlopen("https://api.github.com/users/octocat") as r:
    print(r.status)
    print(r.read().decode("utf-8"))
```

## 二、带请求头与超时
```python
import urllib.request

req = urllib.request.Request(
    "https://api.github.com/user",
    headers={"Authorization": "Bearer ghp_xxx", "Accept": "application/vnd.github+json"},
)
with urllib.request.urlopen(req, timeout=10) as r:
    print(r.read().decode("utf-8"))
```

## 三、发送 JSON（POST）
```python
import json
import urllib.request

data = json.dumps({"name": "demo"}).encode("utf-8")
req = urllib.request.Request(
    "https://httpbin.org/post", data=data,
    headers={"Content-Type": "application/json"}, method="POST",
)
with urllib.request.urlopen(req) as r:
    print(r.read().decode("utf-8"))
```

## 四、进阶
需要会话、重试、超时策略时，可改用 `requests` 库；更多模板见
[@22178384/api-samples](https://github.com/22178384/api-samples)。
