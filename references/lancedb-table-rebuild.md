# LanceDB 表重建 — 修复升级后 "Table not found" 故障

**Date:** 2026-06-01
**Trigger:** lancedb 库升级后，旧 profile 的 `memories.lance` 报 `ValueError: Table 'memories' was not found`。
**Affects:** 用旧版 lancedb（<0.10，估计是 0.3.x）写出的 `.lance` 单文件，升级到 `0.30.x` 后无法 `open_table` 打开。

## 故障特征

```
ValueError: Table 'memories' was not found
```

**实际原因**：不是数据丢了，而是表的 manifest 元数据目录（`_versions/`、`_transactions/`、`_deletions/`、`_indices/`）缺失或格式不对，新版 lancedb 找不到 manifest 所以拒绝打开表。

**诊断步骤**：
```bash
# 1. 看 memories.lance/ 目录里有哪些子目录
ls ~/.hermes/profiles/<name>/lance_memory/memories.lance/

# 正常 (default profile, 升级后):
#   _deletions  _indices  _transactions  _versions  data

# 故障 (受影响的 profile):
#   data   ← 只有 data

# 2. 看 data/ 目录里有没有 .lance 文件
ls ~/.hermes/profiles/<name>/lance_memory/memories.lance/data/ | head -5
# 000000000001000001100100a773bf44949948be9edaa4ee72.lance
# 000000011101001010111110968c1d48e58cf2605612ae50fd.lance
# ...

# 3. 每个 .lance 文件 = 1 条记忆记录
# 4. 文件前 4 字节 = 0x24 (LANCEDB magic) → 文件本身格式正常
```

## 修复流程

### Step 1: 备份

```bash
TS=$(date +%Y%m%d_%H%M%S)
cp -a ~/.hermes/profiles/<prof>/lance_memory \
      ~/.hermes/backups/lance_memory_<prof>_pre-fix_$TS
```

### Step 2: 用 lance.file.LanceFileReader 读出所有数据

**关键 API**：`lance.file.LanceFileReader(path).read_all().to_table()` 读单个 .lance 文件，不依赖 `_versions/` 目录。

```python
import lance.file as lf
import pyarrow as pa
from pathlib import Path

data_dir = Path.home() / ".hermes/profiles/<prof>/lance_memory/memories.lance/data"

all_tables = []
for f in sorted(data_dir.iterdir()):
    if not f.name.endswith(".lance"):
        continue
    try:
        tbl = lf.LanceFileReader(str(f)).read_all().to_table()
        if tbl.num_rows > 0:
            all_tables.append(tbl)
    except Exception as e:
        print(f"ERR {f.name}: {e}")

combined = pa.concat_tables(all_tables)
# combined 是 pyarrow Table，schema 与 default 一致
```

### Step 3: 重建表

```python
import lancedb
import pyarrow as pa
import shutil

profile_dir = Path.home() / ".hermes/profiles/<prof>"
lance_dir = profile_dir / "lance_memory"
memories_dir = lance_dir / "memories.lance"

# 1. 删除旧表目录
shutil.rmtree(memories_dir)

# 2. 重建表
db = lancedb.connect(str(lance_dir))
schema = pa.schema([
    pa.field("id", pa.string()),
    pa.field("content", pa.string()),
    pa.field("role", pa.string()),
    pa.field("session_id", pa.string()),
    pa.field("vector", pa.list_(pa.float32(), 1024)),
    pa.field("created_at", pa.float64()),
    pa.field("metadata", pa.string()),
])
rows = combined.to_pylist()
tbl = db.create_table("memories", data=rows, schema=schema, mode="overwrite")

# 3. 补 FTS 索引 (让 hybrid 搜索可用)
tbl.create_fts_index("content", replace=True)
```

### Step 4: 验证

```python
db = lancedb.connect(str(lance_dir))
tbl = db.open_table("memories")
assert tbl.count_rows() == expected_count
assert tbl.list_indices()  # 有 FTS 索引
```

## 已验证的修复案例 (2026-06-01)

| Profile | 修复前 rows | 修复后 rows | 时间 |
|---|---|---|---|
| <profileA> | 0 (Table not found) | 325 | 0.42s |
| <profileB> | 0 (Table not found) | 237 | 0.10s |

## Pitfalls

### 1. 不要用 `lance.dataset(path)` 读单文件

```python
# ❌ 错误：lance.dataset() 期望的是 dataset 目录，不是单文件
ds = lance.dataset(str(file))  
# → ValueError: LanceError(IO): Generic LocalFileSystem error: Unable to walk dir

# ✅ 正确：用 lance.file.LanceFileReader 读单文件
reader = lance.file.LanceFileReader(str(file))
tbl = reader.read_all().to_table()
```

### 2. `read_all()` 返回的不是 pyarrow Table

`LanceFileReader.read_all()` 返回 `ReaderResults`，要先 `.to_table()` 转换：

```python
# ❌ reader_results.num_rows 报错
# ✅ reader_results.to_table().num_rows
```

### 3. `read_all()` 不要传 `num_rows` 参数

API 设计：`read_all()` 是读全部，`read_range(start=, num_rows=)` 是读范围。

### 4. 安装 pylance

`lance.file` 在 `pylance` 包里，不是 lancedb：

```bash
uv pip install --python ~/.hermes/venv/bin/python pylance
```

### 5. 向量搜索验证

```python
import requests, numpy as np
emb = np.array(
    requests.post("http://localhost:11434/api/embed",
                  json={"model": "bge-m3:latest", "input": ["test"]}).json()["embeddings"][0],
    dtype=np.float32)
results = tbl.search(emb).limit(3).to_list()
```

## 自动化脚本

参考 `scripts/lancedb_rebuild_table.py`（见同目录）—— 一键扫描所有 profile，识别"Table not found"故障并自动重建。
