# jf-open-pro-ai-* 技能 API 调用规范

## ⚠️ 重要原则

所有 `jf-open-pro-ai-*` 开头的技能必须严格遵循以下原则：

### 1️⃣ 必须遵循 curl 的实际结果

- ✅ **以实际 API 响应为准**
- ❌ **禁止使用 Python 脚本缓存回答**
- ❌ **禁止编造解释**

### 2️⃣ 始终用 curl 直接验证 API

- ✅ 重要查询必须输出完整 curl 命令
- ✅ 重要查询必须输出原始响应
- ❌ 不要信任脚本缓存

### 3️⃣ 不编造解释

- ✅ 不一致时承认不确定
- ✅ 如 API 返回与预期不符，如实报告
- ❌ 不要猜测或编造原因

### 4️⃣ 第一时间验证

- ✅ 重要查询直接展示 curl 和响应
- ✅ 使用 `--verbose` 参数输出详细信息
- ❌ 不要延迟验证

---

## 📋 脚本规范

### 必需参数

所有查询脚本必须支持 `--verbose` 或 `-v` 参数：

```python
parser.add_argument("--verbose", "-v", action="store_true", 
                    help="输出详细 curl 命令和原始响应（重要查询必须使用）")
```

### 输出格式

**verbose 模式下必须输出：**

1. **完整 curl 命令**（可用于独立验证）
2. **原始 API 响应**（JSON 格式）
3. **解析后的状态**（简洁明了）

**示例输出：**

```
============================================================
🔧 CURL 命令（用于验证）
============================================================
curl -X POST "https://api-cn.jftechws.com/..." \
  -H "Content-Type: application/json" \
  -H "uuid: xxx" \
  ...

============================================================
📊 API 响应结果
============================================================
{
  "code": 2000,
  "msg": "success",
  "data": {
    "aiAnalysisSwitch": true
  }
}

✅ 岗位巡检服务：已开启
```

### 错误处理

**错误响应必须如实报告：**

```python
if result.get("code") != 2000:
    code = result.get('code')
    msg = result.get('msg', '')
    # 如实报告错误，不编造解释
    return f"❌ API 错误码：{code}\n{msg}"
```

**禁止的行为：**
- ❌ "可能是缓存问题"
- ❌ "两个 API 返回不同"（未经验证）
- ❌ "应该是开启的"（猜测）

---

## 🔍 验证流程

### 标准查询流程

```bash
# 1. 使用 verbose 模式查询
python3 script.py --action status --verbose \
  --sn "xxx" --user "xxx" ...

# 2. 检查输出的 curl 命令

# 3. 检查原始响应

# 4. 确认解析结果与响应一致
```

### 不一致时的处理

1. **承认不确定**
   ```
   ⚠️ 查询结果与预期不符，需要进一步验证
   ```

2. **提供原始数据**
   ```
   API 返回：code=12504, msg=xxx
   请检查设备套餐状态
   ```

3. **不编造解释**
   ```
   ❌ 错误：可能是缓存问题
   ✅ 正确：API 返回 12504，表示套餐未开通
   ```

---

## 📁 适用技能

以下技能必须遵循此规范：

| 技能 | 目录 |
|------|------|
| 岗位巡检 | `jf-open-pro-ai-joblover` |
| 儿童看护 | `jf-open-pro-ai-child-care` |
| 老人看护 | `jf-open-pro-ai-elderly-care` |
| 室内安防 | `jf-open-pro-ai-indoor-security` |
| 宠物看护 | `jf-open-pro-ai-pet-care` |
| 室外安防 | `jf-open-pro-ai-outdoor` |

---

## 🚫 违规示例

### ❌ 错误做法

```python
# 使用缓存
if cached_result:
    return cached_result

# 编造解释
if code == 12504:
    return "可能是缓存问题，实际应该是开启的"

# 不验证就回答
print("✅ 服务已开启")  # 没有实际调用 API
```

### ✅ 正确做法

```python
# 直接调用 API
result = call_api_directly()

# 如实报告
if code == 12504:
    return "❌ API 错误码：12504\n设备未开通套餐"

# 输出 curl 和响应
if verbose:
    print_curl_command()
    print_raw_response()
```

---

## 📝 更新记录

| 日期 | 更新内容 |
|------|----------|
| 2026-05-27 | 初始版本，规范 API 调用行为 |

---

**所有 jf-open-pro-ai-* 技能必须遵循此规范！**
