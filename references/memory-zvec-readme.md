# memory-zvec — Zvec Vector Memory Plugin

**Drop-in replacement for `memory-lancedb`**. Same 6 tool schemas, same
prefetch/sync_turn/on_session_end hooks, same Ollama bge-m3:latest embedding —
but backed by Zvec v0.5.0+ with **native hybrid query** (MultiQuery + RRFReRanker).

## Benefits Over memory-lancedb

| Aspect | memory-lancedb (旧) | memory-zvec (新) |
|--------|---------------------|-------------------|
| Hybrid fusion | 应用层 RRF（30 行手算） | **Zvec 原生 MultiQuery + RRFReRanker**（3 行） |
| Scalar filter | `.where()` 在 ANN 管道内 | **索引级预过滤**（filter 字段都有 InvertIndexParam） |
| FTS 引擎 | Tantivy（外挂） | **RocksDB 原生**（`FtsIndexParam`） |
| Insert 性能 | — | **~13000 docs/s** (1024维) |
| 搜索模式 | 3 种 (hybrid/vector/keyword) | 3 种 (hybrid/vector/keyword) — **保持一致** |
| Tool schemas | 5 个 (add/search/list/delete/stats) | 5 个 — **完全一致** |
| 迁移 | — | 直接迁移，向量不重新 embedding |

## Install

**前提条件:**

1. Zvec 已安装: `pip install zvec`（或加入 venv）
2. Ollama 运行中: `ollama pull bge-m3:latest`
3. Hermes agent（任意 profile）

**安装插件:**

```bash
cp -r ~/.hermes/plugins/memory-zvec /path/to/your/hermes/plugins/memory-zvec
```

## Migration: Switch from memory-lancedb

### 1. Run the migration script

```bash
python3 ~/.hermes/profiles/chip_expert/skills/hardware/vector-db-migration/references/migrate-lancedb-to-zvec.py
```

这将把 LanceDB 的 105 条记忆完整迁移到 Zvec（向量已存在，不重新 embedding）。

### 2. Edit config.yaml

在 `$HERMES_HOME/config.yaml` 中添加：

```yaml
plugins:
  memory-zvec:
    base_url: http://localhost:11434
    embedding_model: bge-m3:latest
    vector_dim: 1024
    zvec_dir: $HERMES_HOME/记忆数据库/zvec_memory
    collection_name: memories
    batch_size: 32
    search_top_k: 5
    min_content_len: 50
    vector_weight: 0.7
    fts_weight: 0.3
    enable_hnsw_optimize: true
```

然后将 `memory.provider` 从 `memory-lancedb` 改为 `memory-zvec`。

### 3. Restart Hermes

```bash
# 重启 agent
hermes stop && hermes start
# 或重新加载配置（需重启）
```

### 4. Verify

启动后，说"查看记忆统计"或"搜索记忆"来确认插件生效。`vec_memory_stats` 的返回中 `backend: zvec` 确认切换成功。

### 5. Rollback

Zvec 和 LanceDB 数据目录共存。想回退只需：

1. `config.yaml` 中恢复 `memory.provider: memory-lancedb`
2. 重启 Hermes

## Configuration

| Key | Default | Description |
|-----|---------|-------------|
| `base_url` | `http://localhost:11434` | Ollama server URL |
| `embedding_model` | `bge-m3:latest` | Embedding model (Ollama) |
| `vector_dim` | `1024` | Vector dimension (bge-m3 = 1024) |
| `zvec_dir` | `$HERMES_HOME/记忆数据库/zvec_memory` | Zvec collection path |
| `collection_name` | `memories` | Collection name |
| `batch_size` | `32` | Max texts per embedding batch |
| `search_top_k` | `5` | Default top-k results |
| `min_content_len` | `50` | Skip content shorter than this |
| `vector_weight` | `0.7` | Hybrid branch weight (vector) |
| `fts_weight` | `0.3` | Hybrid branch weight (FTS) |
| `enable_hnsw_optimize` | `true` | Call optimize() after batch writes |

## Tool Schemas (Identical to memory-lancedb)

| Tool | Purpose |
|------|---------|
| `vec_memory_add(content, role, session_id, metadata)` | Store a fact |
| `vec_memory_search(query, mode, top_k, session_id, after_timestamp, before_timestamp)` | Semantic/hybrid/keyword search |
| `vec_memory_list(limit, session_id, after_timestamp, before_timestamp)` | Browse memories |
| `vec_memory_delete(memory_ids)` | Delete by IDs |
| `vec_memory_stats()` | Show store stats |

## Known Differences (vs memory-lancedb)

1. **filter 语法**: Zvec 用单 `=`（SQL 风格），跟 LanceDB 一样——无差异
2. **HNSW 参数名**: `m=16`（小写），不是 `M=16`
3. **stats 属性**: `doc_count` 而不是 `num_rows`
4. **RRF 类名**: `RrfReRanker`（小写 `r`），不是 `RRFReRanker`
5. **FTS flush on destroy**: Zvec v0.5.0 在删除 collection 时 RocksDB 有已知的 flush 误日志（不影响生产）

## Related Skills

- **`hardware/zvec`** — Zvec 通用参考手册（schema 设计、hybrid search patterns、troubleshooting）
- **`hardware/vector-db-migration`** — 迁移 playbook（含 `migrate-lancedb-to-zvec.py` 脚本）
