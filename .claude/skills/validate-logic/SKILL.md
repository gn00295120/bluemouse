---
name: validate-logic
description: >
  L13-L17 類型和邏輯驗證 - 檢查類型一致性、邏輯完整性、錯誤處理、安全性、性能。
  BlueMouse 17-Layer Validation Group 4（最深層檢查）。
  Triggers: "logic", "security", "performance", "error handling", "安全檢查"
allowed-tools:
  - Read
  - Bash
  - Grep
  - Glob
user-invocable: true
context: fork
---

# Validate Logic Skill (L13-L17)

BlueMouse 17-Layer Validation System - Group 4: 類型和邏輯驗證（最深層檢查）

## Two Ways to Use

### 1. AI-Guided Validation
Follow the checklist below to analyze code.

### 2. Script Execution
```bash
python3 .claude/skills/validate-logic/validator.py myfile.py
python3 .claude/skills/validate-logic/validator.py --verbose myfile.py
```

---

# L13-L17 Validation Checklist

## L13: 類型一致性檢查

**What**: All functions in the code have ≥70% type hint coverage

**How**:
```python
funcs = [n for n in ast.walk(tree) if isinstance(n, ast.FunctionDef)]
total = len(funcs)
with_hints = sum(
    1 for f in funcs
    if f.returns or any(arg.annotation for arg in f.args.args)
)
coverage = int(with_hints / total * 100)
passed = coverage >= 70
```

**Pass**: `"函數類型提示覆蓋率: {coverage}%"` (≥70%)
**Fail**: `"函數類型提示覆蓋率: {coverage}%"` (<70%)

---

## L14: 邏輯完整性檢查 (Informational)

**What**: Code has control flow structures

**How**:
```python
if_count = sum(1 for n in ast.walk(tree) if isinstance(n, ast.If))
for_count = sum(1 for n in ast.walk(tree) if isinstance(n, ast.For))
while_count = sum(1 for n in ast.walk(tree) if isinstance(n, ast.While))

has_branches = (if_count + for_count + while_count) > 0
```

**Output**:
- Has control flow: `"邏輯結構完整"`
- No control flow: `"邏輯結構簡單"`

**Pass**: Always (informational only)

---

## L15: 錯誤處理檢查 ⚠️ ANTI-PATTERN DETECTION

**What**: No empty try-except blocks or pass-only handlers

**How**:
```python
try_nodes = [node for node in ast.walk(tree) if isinstance(node, ast.Try)]

bad_handlers = 0
for node in try_nodes:
    for handler in node.handlers:
        # Empty handler
        if not handler.body:
            bad_handlers += 1
        # Only pass statement
        elif len(handler.body) == 1 and isinstance(handler.body[0], ast.Pass):
            bad_handlers += 1
```

**Pass**: Has try-except AND `bad_handlers == 0` → `"檢測到 N 個有效錯誤處理塊"`
**Fail**:
- No try-except: `"建議添加 try-except 錯誤處理塊"` (L15 視為失敗)
- Bad handlers: `"發現 N 個空的或只有 pass 的錯誤處理塊 (Anti-pattern)"`

**Examples**:
```python
# ❌ FAIL: Empty handler
try:
    risky()
except:
    pass

# ❌ FAIL: Only pass
try:
    risky()
except Exception as e:
    pass

# ✅ PASS: Proper handling
try:
    risky()
except Exception as e:
    logger.error(f"Error: {e}")
    raise
```

---

## L16: 安全性檢查 🔒 SECURITY SCAN

**What**: No dangerous functions or hardcoded secrets

### Dangerous Functions
| Function | Risk | Alternative |
|----------|------|-------------|
| `eval()` | Arbitrary code execution | `ast.literal_eval()` |
| `exec()` | Arbitrary code execution | Avoid |
| `compile()` | Code injection | Avoid |
| `__import__()` | Dynamic import risk | Use regular import |
| `pickle` | Deserialization attack | `json` |

**Detection**:
```python
dangerous_funcs = ['eval', 'exec', 'compile', '__import__']

for node in ast.walk(tree):
    if isinstance(node, ast.Call):
        if isinstance(node.func, ast.Name):
            if node.func.id in dangerous_funcs:
                issues.append(f"使用了危險函數: {node.func.id}")
```

### Hardcoded Secrets
| Pattern | Example |
|---------|---------|
| `api_key = "..."` | `api_key = "sk-123456789"` |
| `password = "..."` | `password = "secret123"` |
| `secret = "..."` | `secret = "mysecret"` |
| `token = "..."` | `token = "eyJ..."` |
| AWS keys | `aws_access_key_id = "AKIA..."` |

**Detection**:
```python
secret_patterns = [
    r'api_key\s*=\s*[\'"][^\s\'\"]{10,}[\'"]',
    r'password\s*=\s*[\'"][^\s\'\"]{8,}[\'"]',
    r'secret\s*=\s*[\'"][^\s\'\"]{10,}[\'"]',
    r'token\s*=\s*[\'"][^\s\'\"]{10,}[\'"]',
    r'aws_access_key_id\s*=\s*[\'"]AKIA',
]
```

**Pass**: `"未發現明顯安全問題"`
**Fail**: `"發現 N 個潛在安全性問題"` + list issues

**Examples**:
```python
# ❌ FAIL: Dangerous function
result = eval(user_input)

# ❌ FAIL: Hardcoded secret
api_key = "sk-1234567890abcdef"

# ✅ PASS: Safe alternatives
import os
api_key = os.environ.get('API_KEY')
result = ast.literal_eval(safe_input)
```

---

## L17: 性能檢查 ⚡ COMPLEXITY ANALYSIS

**What**: No deeply nested loops (≥3 levels)

**How**:
```python
def get_loop_depth(node, current_depth=0):
    max_depth = current_depth
    for child in ast.iter_child_nodes(node):
        if isinstance(child, (ast.For, ast.While)):
            child_depth = get_loop_depth(child, current_depth + 1)
        else:
            child_depth = get_loop_depth(child, current_depth)
        max_depth = max(max_depth, child_depth)
    return max_depth

# Find max nesting depth
for node in ast.walk(tree):
    if isinstance(node, (ast.For, ast.While)):
        depth = get_loop_depth(node, 1)
        max_depth = max(max_depth, depth)

passed = max_depth < 3
```

**Pass**: `"最高循環嵌套深度: {depth} (符合效能規範)"` (depth < 3)
**Fail**: `"檢測到過深的循環嵌套 (Depth: {depth})，建議優化算法"` (depth ≥ 3)

**Examples**:
```python
# ✅ PASS: 2-level nesting (O(n²))
for i in range(n):
    for j in range(n):
        process(i, j)

# ❌ FAIL: 3-level nesting (O(n³))
for i in range(n):      # Level 1
    for j in range(n):  # Level 2
        for k in range(n):  # Level 3 - TOO DEEP!
            process(i, j, k)
```

**Optimization Suggestions**:
- Use dictionary lookups instead of nested loops
- Restructure algorithm
- Use vectorized operations (numpy)

---

## Output Format

```
==================================================
L13-L17: 類型和邏輯驗證
==================================================

Status: ✅ PASSED / ❌ FAILED
Score: X/100 (N/5 layers)

✅/❌ L13: 類型一致性檢查 - 函數類型提示覆蓋率: X%
✅ L14: 邏輯完整性檢查 - 邏輯結構完整/簡單
✅/❌ L15: 錯誤處理檢查 - [message]
✅/❌ L16: 安全性檢查 - [message]
✅/❌ L17: 性能檢查 - [message]

[Verbose mode shows detailed issues]
```

---

## Related Skills

| Skill | Layers |
|-------|--------|
| `/validate-17-layers` | L1-L17 (完整) |
| `/validate-syntax` | L1-L4 |
| `/validate-signature` | L5-L8 |
| `/validate-dependencies` | L9-L12 |
| `/validate-logic` | **L13-L17** |

---

*Part of BlueMouse 17-Layer Validation System*
