# jf-open-pro-ai-* 技能签名使用规范

## ⚠️ 核心规则（必须严格遵守）

### 规则 1: timeMillis 和 signature 是成对生成的

```python
# ✅ 正确：同时生成一对
time_millis, signature = generate_signature(uuid, appkey, secret, movecard)

# ❌ 错误：分别生成
time_millis = generate_time_millis()  # 错误！
signature = generate_signature(...)    # 错误！
```

**原因：** signature 的计算依赖于特定的 timeMillis 值，两者必须匹配。

---

### 规则 2: 每次 API 调用前必须生成新的签名对

```python
# ✅ 正确：每次调用前都生成新的
for api_call in api_calls:
    time_millis, signature = generate_signature(uuid, appkey, secret, movecard)
    curl -X POST ... -H "timeMillis: $time_millis" -H "signature: $signature"

# ❌ 错误：重复使用旧的签名对
time_millis, signature = generate_signature(...)
curl -X POST ...  # 第一次调用
curl -X POST ...  # 第二次调用 - 错误！timeMillis 已经过期
```

**原因：** timeMillis 包含时间戳，每次调用都应该使用当前的时间戳。

---

### 规则 3: 生成后必须立即使用，不能有延迟

```python
# ✅ 正确：生成后立即使用
time_millis, signature = generate_signature(uuid, appkey, secret, movecard)
subprocess.run(curl_cmd)  # 立即执行

# ❌ 错误：生成后延迟使用
time_millis, signature = generate_signature(uuid, appkey, secret, movecard)
time.sleep(1)  # 错误！延迟了
subprocess.run(curl_cmd)  # timeMillis 可能已经"过期"
```

**原因：** 虽然 timeMillis 不会真正"过期"，但延迟可能导致：
1. 服务器端的时间验证失败
2. 多次调用时使用了相同的时间戳（counter 没递增）

---

### 规则 4: 在同一个进程中生成并使用

```python
# ✅ 正确：在同一个 Python 进程中
import subprocess
from jf_signature import generate_signature

time_millis, signature = generate_signature(uuid, appkey, secret, movecard)
curl_cmd = f'curl ... -H "timeMillis: {time_millis}" -H "signature: {signature}" ...'
subprocess.run(curl_cmd, shell=True)

# ❌ 错误：分多次调用 Python
time_millis=$(python3 -c "from jf_signature import generate_signature; t,s=generate_signature(...); print(t)")
signature=$(python3 -c "from jf_signature import generate_signature; t,s=generate_signature(...); print(s)")
# 错误！两次调用生成的 timeMillis 可能不同！
curl ... -H "timeMillis: $time_millis" -H "signature: $signature"
```

**原因：** 每次调用 Python 都是独立的进程，counter 会重置，导致生成的 timeMillis 不同。

---

## 📋 正确的使用模板

### 模板 1: Python 脚本内调用 API

```python
#!/usr/bin/env python3
from jf_signature import generate_signature
import requests

# 参数
uuid = '...'
appkey = '...'
secret = '...'
movecard = 7

# 生成签名（每次调用前都生成新的）
time_millis, signature = generate_signature(uuid, appkey, secret, movecard)

# 立即使用
headers = {
    'uuid': uuid,
    'appKey': appkey,
    'timeMillis': time_millis,
    'signature': signature,
    'Authorization': uuid
}
response = requests.post(url, headers=headers, json=payload)
```

### 模板 2: Python 生成签名并执行 curl

```python
#!/usr/bin/env python3
from jf_signature import generate_signature
import subprocess

# 参数
uuid = '...'
appkey = '...'
secret = '...'
movecard = 7

# 生成签名
time_millis, signature = generate_signature(uuid, appkey, secret, movecard)

# 立即执行 curl
curl_cmd = f'''curl -X POST "{url}" \\
  -H "Content-Type: application/json" \\
  -H "uuid: {uuid}" \\
  -H "appKey: {appkey}" \\
  -H "timeMillis: {time_millis}" \\
  -H "signature: {signature}" \\
  -H "Authorization: {auth}" \\
  -d '{json.dumps(payload)}' '''

subprocess.run(curl_cmd, shell=True)
```

### 模板 3: Shell 脚本调用 Python 生成并执行

```bash
#!/bin/bash

# 在同一个 Python 命令中生成签名并执行 curl
python3 << PYEOF
from jf_signature import generate_signature
import subprocess

uuid = '...'
appkey = '...'
secret = '...'
movecard = 7

# 生成签名
time_millis, signature = generate_signature(uuid, appkey, secret, movecard)

# 立即执行 curl
curl_cmd = f'curl ... -H "timeMillis: {time_millis}" -H "signature: {signature}" ...'
subprocess.run(curl_cmd, shell=True)
PYEOF
```

---

## ❌ 常见错误示例

### 错误 1: 分两次调用 Python 生成

```bash
# ❌ 错误！
TIME_MILLIS=$(python3 -c "from jf_signature import generate_signature; t,s=generate_signature(...); print(t)")
SIGNATURE=$(python3 -c "from jf_signature import generate_signature; t,s=generate_signature(...); print(s)")
# 问题：两次调用生成的 timeMillis 可能不同！
curl ... -H "timeMillis: $TIME_MILLIS" -H "signature: $SIGNATURE"
```

### 错误 2: 重复使用旧的签名对

```bash
# ❌ 错误！
TIME_MILLIS="00000011779934419681"  # 旧的 timeMillis
SIGNATURE="5f421f35d6c8321e312b46b96fc9c610"  # 旧的 signature
curl ... -H "timeMillis: $TIME_MILLIS" -H "signature: $SIGNATURE"  # 第一次调用
curl ... -H "timeMillis: $TIME_MILLIS" -H "signature: $SIGNATURE"  # 第二次调用 - 错误！
```

### 错误 3: 生成后延迟使用

```python
# ❌ 错误！
time_millis, signature = generate_signature(uuid, appkey, secret, movecard)
# ... 做一些其他操作 ...
# ... 打印日志 ...
# ... 等待用户输入 ...
curl ...  # timeMillis 已经"过期"
```

---

## ✅ 验证签名是否正确

### 方法 1: 检查 API 返回码

| 返回码 | 含义 |
|--------|------|
| `2000` | ✅ 签名正确 |
| `12504` | ❌ 签名验证失败（timeMillis 和 signature 不匹配） |
| `12513` | ✅ 签名正确，但设备未开通服务 |
| `12511` | ✅ 签名正确，但设备未找到 |

### 方法 2: 使用自测脚本

```bash
python3 /tmp/test_all_skills.py
```

---

## 📁 相关文件

- **签名工具：** `/root/.openclaw/workspace/skills/aiscene/jf-open-pro-ai-joblover/scripts/jf_signature.py`
- **自测脚本：** `/tmp/test_all_skills.py`
- **规范文档：** `/root/.openclaw/workspace/skills/aiscene/SIGNATURE_USAGE_RULES.md`

---

## 🎯 总结

1. **timeMillis 和 signature 是成对生成的** - 必须一起使用
2. **每次 API 调用前生成新的签名对** - 不要重复使用
3. **生成后立即使用** - 不要有延迟
4. **在同一个进程中生成并使用** - 不要分多次调用 Python

**严格遵守以上规则，确保签名验证通过！** ✅
