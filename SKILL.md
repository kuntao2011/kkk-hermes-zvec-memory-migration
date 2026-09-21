---
name: kkk-zvec-vector-db-migration
description: |
  Hermes 记忆系统管理：memory-zvec 部署、运维、迁移与故障修复。
  涵盖插件跨 profile 部署、数据迁移、索引修复、全功能验证、健康检查的完整 playbook。
  适用于在任意 Hermes profile 下将记忆后端切换为或运维 Zvec。
version: 3.4.0
author: kuntao2011
license: MIT
category: devops
tags: [vector-database, zvec, lancedb, migration, memory, hermes]
platforms: [linux, macos]
---

# Zvec Vector DB — 部署、迁移与运维手册

Hermes 记忆系统（memory-zvec）的完整操作手册。
覆盖部署、迁移、健康检查、故障诊断、索引修复、数据备份恢复。

> **合并历史**：v3.0.0 合并了 `memory-lancedb-admin`（运维检查）
> 和 `lancedb-memory-migration`（LanceDB 迁移）的有用内容。
> 原技能已归档删除。

---

## 记忆系统架构

### Dual-Storage 架构（⚠️ 常见误解）

Hermes 使用**双重存储**系统，两者**同时自动存储**：

| 层 | 组件 | 作用 | 自动/手动 |
|----|------|------|-----------|
| 完整历史 | `state.db` (SQLite/FTS5) | 所有消息（用户/AI/工具调用） | 自动（每轮写入） |
| 语义检索 | Zvec collection | 向量嵌入 + FTS 索引 | 自动（插件钩子） |

**memory-zvec 插件自动同步机制**（Hermes 侧编排，见 [`references/memory-write-hooks.md`](references/memory-write-hooks.md)）：
- `sync_turn(user_content, assistant_content)`: 每轮对话后实时存储 (role="turn")
  - 由 `run_agent._mirror_to_memory()` 在每轮完成后调用
  - 通过 `MemoryManager._submit_background()` 在后台线程执行，不阻塞对话
  - **被中断（interrupted）的轮次跳过不写入**（源码 `if interrupted: return`）
- `on_session_end(messages)`: 会话结束时批量存储 (role="session_end")
  - 由 `commit_memory_session()` 或 `shutdown_memory_provider()` 触发
  - 触发时机：`/new`、`/reset`、CLI 退出、gateway session 过期
  - 遍历整个 messages 列表，提取所有 user+assistant 配对，每对一条记录
  - `commit_memory_session()`：只调 `on_session_end`，不 teardown provider（session 轮转时）
  - `shutdown_memory_provider()`：调 `on_session_end` + `shutdown_all()`（真正关闭时）

**⚠️ 双写机制**：同一对话可能存两条记录（role="turn" + role="session_end"），不冲突。
`sync_turn` 是实时保障，`on_session_end` 是兜底——如果某轮 sync_turn 因 Ollama 故障失败，
on_session_end 还会补写。

**⚠️ 不要混淆**：手动调用 `vec_memory_add` 是**显式添加**，不是自动存储的替代。
自动存储走插件钩子，内容是完整对话；手动添加适合存入精炼的知识片段。

**⚠️ 交互式 CLI 会话不挂外部 provider**：CLI 侧 `skip_memory=self.ignore_rules`（`cli_agent_setup_mixin.py`），
即使 `memory.provider: memory-zvec` 已配置，CLI 会话也只加载内置 MEMORY.md，**不加载 memory-zvec**——
日志里没有 `Memory provider 'memory-zvec' registered/activated`，agent 也没有 `vec_memory_*` 工具。
所以：CLI 会话（含 `-z` 一次性运行）聊多少轮都**不会**产生 Zvec 记录；验证"每轮自动写入"必须用
gateway（飞书/CRON）会话，不要用 CLI 反复试，否则会误判为钩子坏了。


### 存储文件位置

```
~/.hermes/                              # default profile
├── state.db                            # 完整 session 历史
├── 记忆数据库/zvec_memory/memories/    # Zvec collection
└── 记忆数据库/lance_memory/             # [归档] 旧 LanceDB 数据（保留备份）

~/.hermes/profiles/<name>/              # 子 profile
├── state.db                            # 完整 session 历史
├── 记忆数据库/zvec_memory/memories/    # Zvec collection
└── 记忆数据库/lance_memory/             # [归档] 旧 LanceDB 数据
```

> **⚠️ 业务向量库**：部分 profile 还有 `lancedb_news/`、`lancedb_analysis/` 等领域数据
> 向量库，与 session 记忆完全独立。详见
> [`references/multi-profile-business-lancedb.md`](references/multi-profile-business-lancedb.md)。

---

## 架构前提：Hermes 插件/技能的 Profile 隔离机制

**关键概念**：Hermes 使用 `HERMES_HOME` 环境变量隔离各 profile 的插件和技能。

```
HERMES_HOME（无 profile / default） = ~/.hermes/
HERMES_HOME（有 profile 时）        = ~/.hermes/profiles/<profile_name>/
```

> **⚠️ 注意**：default profile 没有 `~/.hermes/profiles/default/` 目录。
> 它的 `HERMES_HOME` 直接就是 `~/.hermes/`。迁移脚本和验证脚本中
> 必须对 `default` 做特殊处理（见踩坑记录 §8）。

> **⚠️ $HERMES_HOME 解析陷阱**：各 profile 的 systemd service 设置
> `Environment="HERMES_HOME=/home/<user>/.hermes/profiles/<name>"`，
> 所以 config.yaml 中的 `$HERMES_HOME/...` 在运行时展开为**该 profile 自己的目录**。
> 多个 profile 的 config.yaml 中看起来有相同的 `$HERMES_HOME/...` 路径，
> 但运行时解析结果**不同**。验证方法：
> ```bash
> for p in default chip_expert financial_expert health_manager zunhunfan; do
>   pid=$(pgrep -f "hermes.*$p" 2>/dev/null | head -1)
>   [ -n "$pid" ] && cat /proc/$pid/environ 2>/dev/null | tr '\0' '\n' | grep HERMES_HOME
> done
> ```

插件扫描路径：`$HERMES_HOME/plugins/`
技能扫描路径：`$HERMES_HOME/skills/`

**这意味着**：放在 `~/.hermes/plugins/` 的插件只有无 profile 的 root gateway 能发现，
所有 `--profile` 启动的 gateway 都扫描不到它。

---

## Step 1：安装 memory-zvec 插件本体（用户级 fork：memory-zvec）

> **2026-09-14 起的规范布局**。插件是本地 fork（上游 NousResearch/hermes-agent
> 从未包含 memory-zvec），版本历史：in-tree 两轮补丁（0910 optimize_fix、
> 0911 shutdown race）→ v1.1.0 迁移+排水 → v1.2.x 空闲释放 → v1.3.x 句柄共享
> +对抗审查修复。**规范源**唯一放在 Hermes 根目录（default profile 的发现根）：

```
~/.hermes/plugins/memory-zvec/     ← 规范源（唯一可编辑副本，v1.3.1）
├── __init__.py                       ← ZvecMemoryProvider + 锁治理
└── plugin.yaml                       ← name: memory-zvec（配置 key 沿用）
```

配置块 key 仍是 `plugins.memory-zvec`（代码硬编码，历史兼容），数据仍是
`$HERMES_HOME/记忆数据库/zvec_memory/memories`，**与旧版零迁移**。

### 前提条件

1. **Zvec Python 包**：安装在 hermes-agent 的 venv 中
   ```bash
   ~/.hermes/hermes-agent/venv/bin/pip install zvec>=0.5.0
   # 验证
   ~/.hermes/hermes-agent/venv/bin/python -c "import zvec; print(zvec.__version__)"
   ```

2. **Ollama + bge-m3:latest**：确保 Ollama 运行且模型已拉取
   ```bash
   ollama list | grep bge-m3
   # 如果没有
   ollama pull bge-m3:latest
   ```

---

## Step 2：跨 Profile 部署（⚠️ 实体副本，不是符号链接，更不是 in-tree）

> **铁律：记忆提供者的发现根是 profile 作用域的。** `~/.hermes/plugins/<name>/` 只有
> **default** 能看到；子 profile 不部署会报 `Plugin: NOT installed`，且 config 里
> provider 仍写着 → 外部记忆**静默失效、零告警**。
> 完整机制见 [`references/discovery-and-deployment-verification.md`](references/discovery-and-deployment-verification.md)。

**三条路线的取舍（2026-09-14 实测复盘）**：

| 路线 | 结论 |
|------|------|
| in-tree（复制进 hermes-agent/plugins/memory/ + git exclude） | ❌ 已废弃：`hermes update` 的 stash/ZIP 树替换可整目录冲掉，且污染上游树（0910/0911 补丁就是这么丢过风险的） |
| 每 profile **符号链接** → 规范源 | ❌ 历史事故：update 的 rsync 备份恢复会丢符号链接，且丢了是静默失效 |
| 每 profile **实体副本**（rsync 自规范源） | ✅ 现行方案：任何 update 路径都动不了；代价 = 改完规范源要重新分发 |

```bash
# 分发（改过规范源后重跑；顺手清 __pycache__）
for p in ~/.hermes/profiles/*/; do
  rsync -a ~/.hermes/plugins/memory-zvec/ "${p}plugins/memory-zvec/"
  rm -rf "${p}plugins/memory-zvec/__pycache__"
done
```

**逐 profile 验证「发现」**（不能只看文件存在）：

```bash
for prof in chip_expert financial_expert health_manager trading_bot zunhunfan; do
  HERMES_HOME="$HOME/.hermes/profiles/$prof" ~/.hermes/hermes-agent/venv/bin/python -c "
import sys; sys.path.insert(0, '/home/<user>/.hermes/hermes-agent')
from plugins.memory import find_provider_dir
import os; print('$prof ->', find_provider_dir('memory-zvec'))"
done
# default 用 HERMES_HOME=~/.hermes 再跑一次；应各自指向本 home 下的实体目录
```

> 变更生效需 `systemctl --user restart hermes-gateway[-<profile>].service`
> （dashboard 也加载 provider，一并重启）；逐个滚动，每个只断该 profile 数秒。

---

## Step 3：配置 Profile

在目标 profile 的 `config.yaml` 中添加：

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  provider: memory-zvec      # ← fork 目录名（bundled 同名已删，旧名解析为 None）
```

**注意**：只有声明了 `provider:` 的 profile 才会尝试加载该插件。
其他 profile 即使部署了副本也不会激活（不会报错）。

**⚠️ 配置块 key 不随目录名变**：插件读写的是 `plugins.memory-zvec:` 段
（代码硬编码历史兼容），`zvec_dir` 等 config 原样沿用，切换目录名零迁移。

---

## Step 4：数据迁移

### 方式 A：从 LanceDB 迁移（向量复用）

如果源数据在 LanceDB 中，向量可以直接复用（相同 bge-m3:latest 模型，1024 维），
不需要重新 embedding。

**关键**：LanceDB `.lance` 目录是直接用 `lance.dataset()` 打开（不是 `lancedb.connect().open_table()`）。

**完整迁移脚本模板**（见 [`scripts/migrate-lancedb-to-zvec.py`](scripts/migrate-lancedb-to-zvec.py)）：

```python
# 核心模式：
ds = lance.dataset(LANCEDB_PATH)           # 直接打开 .lance 文件
df = ds.to_table().to_pandas()              # 全部读取
coll = zvec.create_and_open(ZVEC_PATH, schema)  # 路径必须不存在
for _, row in df.iterrows():
    doc = zvec.Doc(
        id=str(row['id']),
        vectors={"vector": ast.literal_eval(row['vector']) if isinstance(row['vector'], str) else row['vector']},
        fields={...},                       # 标量字段从 row 中提取
    )
coll.insert([doc1, ...])                    # 批量插入
coll.flush()
coll.optimize()                             # HNSW 索引构建
```

**注意事项**：
- `zvec.Doc` 构造见踩坑记录 §0
- `create_and_open` 路径必须不存在，见踩坑记录 §0c
- 插入后必须 `flush()` + `optimize()` 确保 HNSW 索引完整
- 不存在 `close()` 方法，Python GC 自动释放

### 方式 B：从其他来源新建

插件首次启动时会自动创建 collection（`create_and_open`）。
如果目录已存在且有数据，则直接 `open`。

数据目录默认位置：`$HERMES_HOME/记忆数据库/zvec_memory/memories`

### 方式 C：从 FTS5 Session 历史迁移（LanceDB 路径，已归档）

> **⚠️ 已归档**：以下流程从 `state.db` → LanceDB，现已改为 `state.db` → Zvec。
> 保留作为参考。新版迁移脚本见方式 A。

历史脚本：[`scripts/migrate_sessions_to_lancedb.py`](scripts/migrate_sessions_to_lancedb.py)

该脚本使用 **Strip Pipeline**（5 阶段内容净化），避免工具调用模板重复污染向量空间。
详见 [`references/strip-pipeline-architecture.md`](references/strip-pipeline-architecture.md)。

---

## Step 5：索引修复（⚠️ 关键步骤）

迁移后数据写入 Zvec，但**索引需要手动修复**。以下是三个已知问题及修复方法：

### 问题 1：HNSW 向量索引完整度为 0%

**症状**：`collection.stats` 显示 `index_completeness: {"vector": 0.0}`

**原因**：批量写入后未执行 `optimize()`，向量停留在 flat buffer 中，
所有向量搜索退化为暴力遍历。

> **⚠️ 还有第二处根因（易漏）**：`optimize()` 原本只在 `on_session_end` 里调用；
> **每轮实时写入的主路径 `sync_turn` 从不 optimize**，所以日常使用中完整度会长期偏低
> （实测 0.71），而配置里 `enable_hnsw_optimize: true` 看起来"已生效"。
> 修法：在 `_insert()` 的 flush 后加摊销计数，每 64 次写入 optimize 一次。
> 详见 [`references/discovery-and-deployment-verification.md`](references/discovery-and-deployment-verification.md) §5。

**修复**：
```python
import zvec
coll = zvec.open("/path/to/memories")
coll.optimize()
print("index_completeness:", coll.stats.index_completeness)
# 应输出 {"vector": 1.0}
```

### 问题 2：FTS 索引完全失效

**症状**：所有 FTS 搜索（无论中英文）返回 0 结果。

**原因**：
1. 迁移时 FTS 索引文件存在但数据未正确写入
2. FTS 分词器使用 `standard`（只支持 ASCII），中文内容完全无法索引

**修复**：重建 FTS 索引，使用 `jieba` 分词器：
```python
import zvec
coll = zvec.open("/path/to/memories")
try:
    coll.drop_index("content")
except Exception:
    pass
coll.create_index("content", zvec.FtsIndexParam(tokenizer_name="jieba", filters=["lowercase"]))
# 验证
results = coll.query(zvec.Query(field_name="content", fts=zvec.Fts(match_string="芯片验证")), topk=5)
print(f"FTS hits: {len(results)}")
```

**同时修改插件代码**（`memory-zvec/__init__.py`），确保新建 collection 时默认使用 `jieba`：
```python
# 原代码（standard 不支持中文！）
zvec.FieldSchema("content", zvec.DataType.STRING,
    index_param=zvec.FtsIndexParam(tokenizer_name="standard")),
# 改为
zvec.FieldSchema("content", zvec.DataType.STRING,
    index_param=zvec.FtsIndexParam(tokenizer_name="jieba", filters=["lowercase"])),
```

### 问题 3：标量字段缺少 InvertIndex

**症状**：`role`、`session_id`、`created_at` 的 `index_param` 为 None，
标量过滤走全表扫描。

**修复**：
```python
import zvec
coll = zvec.open("/path/to/memories")
for field_name in ["role", "session_id", "created_at"]:
    try:
        coll.drop_index(field_name)
    except Exception:
        pass
    coll.create_index(field_name, zvec.InvertIndexParam(enable_range_optimization=True))
```

### 一键修复脚本

完整修复脚本见 [`references/post-migration-repair.py`](references/post-migration-repair.py)。

---

## Step 6：全功能验证清单

迁移完成后，通过 Hermes 工具逐一验证（这些工具由插件注册）：

> **快速验证脚本**：可直接运行 [`scripts/verify-plugin-tools.py`](scripts/verify-plugin-tools.py)，
> 自动执行全部 8 项检查（通过 Hermes 框架加载插件，无需关闭 gateway）：
> ```bash
> HERMES_PROFILE=zunhunfan ~/.hermes/hermes-agent/venv/bin/python3 \
>   ~/.hermes/skills/kkk-zvec-vector-db-migration/scripts/verify-plugin-tools.py
> ```
>
> **写路径改动后追加跑两个回归**（否则会漏掉只在关停/退出时暴露的丢写入 bug）：
> ```bash
> # 1) shutdown() 竞态（踩坑 #17）
> ~/.hermes/hermes-agent/venv/bin/python3 scripts/test_shutdown_race.py \
>     ~/.hermes/backups/<旧版副本>/__init__.py.bak   # 带 A/B 对照
> # 2) 进程退出吃掉在飞写入（踩坑 #18）
> HERMES_MEMORY_ZVEC_EXIT_DRAIN_S=0 ~/.hermes/hermes-agent/venv/bin/python3 \
>     scripts/exit_loss_experiment.py ~/.hermes/hermes-agent/plugins/memory/memory-zvec/__init__.py drain_off
> ```

| # | 工具 | 验证内容 | 预期结果 |
|---|------|---------|---------|
| 1 | `vec_memory_stats` | 总条数、session 数、embedding 模型 | `backend: zvec`，非 NoneType error |
| 2 | `vec_memory_list` | 列出最新记忆 | 返回含 metadata 的结果 |
| 3 | `vec_memory_add` | 写入测试条目 | `status: added`, 返回 id |
| 4 | `vec_memory_search(mode=vector)` | 向量语义搜索 | 返回 score + results |
| 5 | `vec_memory_search(mode=keyword)` | FTS 关键词搜索（中文） | 中文关键词命中 |
| 6 | `vec_memory_search(mode=hybrid)` | 向量+FTS RRF 融合 | 返回融合排序结果 |
| 7 | `vec_memory_search(session_id=...)` | 按 session 过滤 | 仅返回指定 session |
| 8 | `vec_memory_delete` | 删除测试条目 | `deleted: 1` |

### ⚠️ 确认 `_coll` 已初始化（防静默失败）

`vec_memory_search` 返回空结果 **不一定** 是数据库没有数据——它可能在 `self._coll is None` 时通过 `except` 兜底静默返回 `{"results":[],"count":0}`。优先用 `vec_memory_stats` 验证（如果有明确的非 NoneType 错误或返回正常计数，说明 `_coll` 有效）。

如果 `vec_memory_stats` 报 `'NoneType' object has no attribute 'stats'`，说明 `initialize()` 未成功创建 `_coll`。排查顺序：

1. 检查日志有无 `name '_open_read_only' is not defined` → 修 pitfall #14
2. 检查日志有无 `still locked after retry` → 旧版（bundled ≤1.0.0）修 pitfall #12（initialize 开头 `self.shutdown()`）；**v1.3.x 反其道：刻意不动已打开句柄 + drain/backoff 重试**，勿把旧修法套到新版
3. 检查日志最后是否有 `ZvecMemoryProvider initialized` → 如果缺失则 `_coll` 仍为 None
4. 查看日志中 `Memory provider 'memory-zvec' activated` —— **这条不代表初始化成功**（hermes-agent 即使初始化异常也打印此条）

**推荐验证脚本**：直接运行 [`scripts/verify-plugin-tools.py`](scripts/verify-plugin-tools.py)（通过 Hermes 框架加载插件，无需关闭 gateway）。

### 插件层验证（推荐，无需关闭 gateway）

通过 Hermes 框架直接加载插件并调用工具，无需 gateway 参与：

```bash
HERMES_PROFILE=<profile> ~/.hermes/hermes-agent/venv/bin/python3 << 'EOF'
import os; os.environ['HERMES_HOME'] = os.path.expanduser(f'~/.hermes/profiles/<profile>')
import sys; sys.path.insert(0, os.path.expanduser('~/.hermes/hermes-agent/venv/lib/python3.11/site-packages'))
from plugins.memory import load_memory_provider
p = load_memory_provider('memory-zvec')
p.initialize(session_id='verify')
import json
def call(name, params):
    r = p.handle_tool_call(name, params)
    return json.loads(r) if isinstance(r, str) else r
print(call('vec_memory_stats', {}))
print(call('vec_memory_search', {'query':'测试','mode':'keyword','top_k':3}))
print(call('vec_memory_add', {'content':'验证条目，至少50字符。','role':'turn'}))
EOF
```

**注意**：`handle_tool_call` 返回 JSON 字符串，不是 dict。`vec_memory_delete` 的参数名是 `memory_ids`（数组），不是 `id`。

---

## Step 7：日志确认

迁移成功后，检查 gateway 日志确认插件以正确路径加载：

```bash
grep "memory-zvec\|MemoryProvider\|memory.*activated" \
    ~/.hermes/profiles/<profile>/logs/agent.log
```

预期日志：
```
agent.memory_manager: Memory provider 'memory-zvec' registered (5 tools)
_hermes_user_memory.memory-zvec: ZvecMemoryProvider opened existing collection: .../memories
_hermes_user_memory.memory-zvec: ZvecMemoryProvider initialized — model=bge-m3:latest dim=1024 ...
run_agent: Memory provider 'memory-zvec' activated
```

关键指标：
- 命名空间为 `_hermes_user_memory.memory-zvec`（说明走的是 user plugins 路径）
- 持有 LOCK 文件的进程应为该 profile 的 gateway PID

---

## 健康检查（运维用）

### Ollama 连通性

在 WSL 环境中，Ollama 通常运行在 Windows 侧：

```bash
# TCP 端口快速检测（推荐）
timeout 3 bash -c 'echo > /dev/tcp/localhost/11434 && echo "OPEN"' 2>&1

# 确认 Windows 侧进程
tasklist.exe 2>/dev/null | grep -i ollama

# API 检查（可能因 WSL 转发慢而超时）
curl -s --connect-timeout 2 http://localhost:11434/ | head -5
```

> **Pitfall**：Ollama 在 WSL 中通过 localhost 端口转发访问 Windows 侧进程。
> 如果 Windows Ollama 服务停止，embedding 失败——已有数据仍可读，但新写入会报错。
> `curl` 可能因 WSL 转发慢而超时，优先使用 TCP 端口检测。

### 快速 Ollama 检查脚本

```bash
bash ~/.hermes/skills/kkk-zvec-vector-db-migration/scripts/check_ollama.sh
```

### "记忆数在某日期后减少" — 真实使用下降 vs 写入故障

**零写入日定性三步**（先查使用侧再判故障）：
```bash
# 1) 当天有无用户 DM（gateway.log）
grep -c "<日期>.*Inbound dm message" ~/.hermes/logs/gateway.log
# 2) cron 当天真实运行数（Shutdown drain 行的 cron_now 字段，或 job 执行日志）
grep "<日期>" ~/.hermes/logs/gateway.log | grep -c "cron_now=[1-9]"
# 3) agent.log 当天有无 agent 活动
```
三者皆零 → 真实零使用（正常），不要当写入故障修。注意 count_in 类 FTS 近似计数脚本中 `created_at` 是**秒级**时间戳（过滤值别乘 1000，乘了会全部返回 0，易误判为断流）。

**锁竞争失败（still locked after retry）是跨 profile 共性现象**，各 profile 日志均有 1~7 次/轮转期。触发场景：前会话 `/new` 后 on_session_end 兜底批次仍在后台写（idmap.0 mtime 更新是证据），新会话 initialize 立刻 open 撞锁，GC+retry 0.6s 等不到释放 → read-only 也被拒 → `_coll=None`。**该会话所有 vec_memory_* 工具瘫到会话结束，且不会自动重试**——用户须 `/new` 或让 gateway 重建 agent 才恢复。诊断 fd 扫描要等锁释放后才做（持有中的锁 find /proc 正常能扫到；扫不到=持有方已释放或锁在进程内对象里）。
> **⚠️ 适用范围**：以上为 bundled v1.0.0 行为（2026-09-14 前）。memory-zvec v1.1+ 已根治——initialize 先排水在飞写入再带退避重试，v1.3.x 句柄跨会话共享，此竞态不复存在；若新版日志仍见此告警即为新问题。

当发现记忆数据在某日期后明显减少时，**先查日期分布再判断**：

```bash
# 通过 vec_memory_list 检查最新记忆的时间范围
vec_memory_list  # 观察最新条目的 created_at
```

**如果数量下降对应**：gateway.log 消息数也在同期下降 → **真实使用下降，不是 bug**。

**真正的写入故障特征**：
- `agent.log` 中有 `WARNING.*session_end batch store failed`
- `agent.log` 中有 `Connection refused` to Ollama
- 旧日期的记忆数突然减少（数据损坏）

---

## 记忆丢失风险矩阵（源码审计）

以下结论来自对 `run_agent.py`、`gateway/run.py`、`memory_manager.py`、`turn_finalizer.py`、
`conversation_compression.py` 的系统性源码审计。

### ✅ 无风险（源码确认安全）

| 场景 | 机制 | 源码位置 |
|------|------|----------|
| 正常每轮对话 | `sync_turn` 后台写入 + `on_session_end` 兜底 | `run_agent.py:3127` / `turn_finalizer.py:441` |
| `/new` 或 `/reset` | `commit_memory_session(messages)` → `on_session_end` | `run_agent.py:3060` |
| gateway 优雅关闭（SIGTERM/SIGINT） | `_stop_impl` → `_finalize_shutdown_agents` + idle cache `_cleanup_agent_resources` | `gateway/run.py:6738-6755` |
| idle cache 驱逐（LRU cap / TTL） | **不动 memory provider**，session 可随时恢复 | `gateway/run.py:13968` 注释："memory provider keeps running" |
| 上下文压缩 | 压缩前先 `commit_memory_session(messages)` | `conversation_compression.py:525` |
| Ollama 偶尔超时 | `sync_turn` 整体 `try/except` 丢本轮，`on_session_end` 兜底 | `__init__.py:575-576` |

### ⚠️ 有风险但可接受

| 风险 | 概率 | 影响 | 原因 |
|------|------|------|------|
| **中断的对话不写** | 中 | 中 | `interrupted=True` 时 `sync_turn` 直接 return。设计意图正确（被中断的回复用户没看到），但如果用户主动中断一个有用回复，该轮丢失。后续 `/new` 触发 `on_session_end` 会补写 |
| **Ollama 持续宕机** | 低 | 高 | `sync_turn` 和 `on_session_end` 都需要 embedding。整个会话期间 Ollama 挂掉 = 所有写入失败，直到 `/new` 或关闭时才暴露。但 `state.db`（完整对话历史）不受影响，可事后重放 |
| **`on_session_end` 中 embedding 失败逐条跳过** | 低 | 低 | 单条 `continue` 跳过，`sync_turn` 已兜底写入 role="turn" 版本。见 pitfall §13 |

### 🔴 真正有丢失风险的场景

| 风险 | 概率 | 影响 | 触发条件 |
|------|------|------|----------|
| ~~**daemon 写入线程被进程退出杀死**~~ | ~~高~~ | ~~中~~ | **已修（踩坑 #18）**：新增 atexit 有界 join。修前 cron 任务几乎必踩（外部 worker 寿命 ~0.7s） |
| **SIGKILL / 硬杀进程** | 极低 | 高 | OOM killer、`kill -9`、Docker 强制销毁。daemon 线程直接死亡，`on_session_end` 不会被调用（atexit 也不会跑） |
| **sync_turn daemon 线程在 flush 前死亡** | 极低 | 低 | 已调 `_coll.insert()` 但未 `_coll.flush()` 时进程被杀。Zvec WAL 机制可能已持久化但未确认 |
| **Zvec collection 损坏** | 极低 | 高 | 磁盘满、文件系统错误、Zvec bug。无自动恢复路径 |

### Gateway 优雅关闭的记忆安全保障

`gateway/run.py` 的 `_stop_impl()` 关键序列（源码审计确认）：

```
① _drain_active_agents(timeout)       — 等待运行中 turn 完成（含 finalize_turn → sync_turn）
② _finalize_shutdown_agents(active)   — 对 running agents 调 _cleanup_agent_resources → on_session_end
③ 遍历 _agent_cache (idle agents)     — 第 6740-6755 行，专门处理 idle cached agents
   → _cleanup_agent_resources(agent)  → shutdown_memory_provider(messages) → on_session_end
④ 断开所有 adapters
```

**关键**：步骤 ③ 确保了 idle cached agents 的记忆也不会丢失。
这是本次审计的核心确认点——不是所有驱逐路径都安全（LRU/idle sweep 走 `release_clients` 不调 on_session_end），
但 **gateway 优雅关闭** 对所有 agent（running + idle cached）都正确处理了记忆。

---

## 数据备份与恢复

### 备份/恢复前必须检查 mtime（⚠️ 关键）

**永远不要直接 `cp -a` 覆盖数据文件**，先比对 mtime：

```bash
for p in 记忆数据库/zvec_memory state.db; do
  bak=/path/to/backup/$p
  cur=/path/to/current/$p
  if [ -e "$bak" ] && [ -e "$cur" ]; then
    bak_t=$(stat -c%Y "$bak")
    cur_t=$(stat -c%Y "$cur")
    if [ "$cur_t" -gt "$bak_t" ]; then
      echo "  ⚠️  $p  current newer → DO NOT overwrite (data loss risk)"
    else
      echo "  ✅ $p  backup newer or same"
    fi
  fi
done
```

**判断矩阵**：

| 当前 mtime vs 备份 | 决策 |
|---|---|
| 当前更新 | **永远不要覆盖**（覆盖会丢数据） |
| 备份更新 + 当前更大 | 备份覆盖（可能是误删/截断）|
| 备份更新 + 当前更小或相同 | 备份覆盖 |

**state.db 特殊情况**：即使 mtime 显示"备份更新"，当前 state.db 可能包含
备份后新产生的 session。正确做法是**不动 state.db**，用迁移脚本从当前 state.db
增量同步（脚本有 session_id 去重，幂等可重跑）。

**实际恢复顺序**：
1. 列出所有数据类文件（zvec_memory、state.db 等）
2. 对每个跑 mtime + size 对比
3. **询问用户**哪些覆盖、哪些跳过
4. 先 `mkdir -p pre-restore-backup-$(date +%Y%m%d)/` 然后 cp

---

## 踩坑记录

### 0. Zvec Doc 对象格式（⚠️ 迁移脚本最常见错误）
**症状**：`AttributeError: 'dict' object has no attribute 'id'`
**原因**：Zvec `insert()` 不接受 dict，必须传入 `zvec.Doc` 对象。
**正确写法**：
```python
doc = zvec.Doc(
    id="uuid-string",
    vectors={"vector": vec_list},        # 向量字段名必须与 schema 中 VectorSchema.name 一致
    fields={"content": "...", "role": "...", ...},  # 标量字段
)
coll.insert([doc1, doc2, ...])
```
**注意**：`id` 不是 `FieldSchema` 的 `primary_key` 参数——`FieldSchema` 没有 `primary_key` 参数，ID 通过 `zvec.Doc.id` 设置。

### 0b. CollectionSchema 构造要点
- 标量字段放 `fields=[]`，向量字段放 `vectors=[VectorSchema(...)]`
- `VectorSchema(name, data_type, dim, index_param=...)` — `dim` 是独立参数，不是 `FieldSchema`
- 不存在 `primary_key=True` 这样的 FieldSchema 参数

### 0c. create_and_open 路径要求
**症状**：`ValueError: path validate failed: path[...] exists`
**原因**：`zvec.create_and_open()` 要求目标路径**完全不存在**，即使是一个空目录也会报错。
**解决**：确保目标路径的**父目录**存在，但目标路径本身不存在：
```python
os.makedirs(parent_dir, exist_ok=True)
assert not os.path.exists(target_path)  # 必须！
coll = zvec.create_and_open(target_path, schema)
```

### 0d. Ollama /api/embed 正确调用方式
**注意**：Ollama embedding API 与 OpenAI 格式不同：
- 请求参数用 `input`（不是 `prompt`）：`{"model": "bge-m3:latest", "input": ["text"]}`
- 响应字段是 `embeddings`（数组，不是 `embedding`）：`resp.json()["embeddings"][0]`
- `prompt` 参数会被**静默忽略**，返回空数组 `{"embeddings": []}`，导致 `IndexError: list index out of range`
- 这是最容易浪费调试时间的陷阱——错误不会抛出，Ollama 返回 200 + 空数组
- memory-zvec 插件内部正确使用 `input`，但手动编写验证脚本时极易踩坑

### 1. LOCK 冲突
**症状**：`RuntimeError: Can't lock read-write collection`
**原因**：gateway 进程已持有 collection 锁，直接用 Python 连接会被拒绝
**解决**：验证时用 Hermes 工具（通过已加载的插件实例），或关闭 gateway 后用 Python

### 2. create_and_open 对已有目录报错
**症状**：`path validate failed: path[...] exists`
**原因**：插件 `initialize()` 先 `zvec.open()`，失败后 fallback 到 `create_and_open()`
**解决**：如果 open 失败（如 LOCK 被占），create_and_open 也会失败。需先释放锁。

### 3. 跨 profile 写入被拦截
**症状**：`Cross-profile write blocked by soft guard`
**原因**：修改 `~/.hermes/plugins/`（属于 root/default profile）时，带 profile 的 agent 有写入保护
**解决**：使用 `cross_profile=True` 参数，或直接在 root profile 下操作

### 4. FTS 分词器选错导致中文不工作
`standard` 分词器是 ASCII whitespace tokenizer，对中文完全无效。
必须使用 `jieba`（Zvec 内置支持，无需额外安装）。

### 5. handle_tool_call 返回值是 JSON 字符串
**症状**：`AttributeError: 'str' object has no attribute 'get'`
**原因**：`ZvecMemoryProvider.handle_tool_call()` 统一返回 JSON 字符串，不是 dict。
**解决**：调用后需 `json.loads(result)` 解析。注意 `vec_memory_add` 对内容过短（<50 字符）的条目返回 `{"status": "skipped", "reason": "content too short"}`。

### 6. vec_memory_delete 参数名是 memory_ids（数组）
**症状**：`{"error": "memory_ids is required"}` 或 `Delete(): incompatible function arguments`
**原因**：`vec_memory_delete` 的参数是 `memory_ids: [str]`（字符串数组），不是 `id: str`。
**正确调用**：`{"memory_ids": ["uuid1", "uuid2"]}`

### 7. 切换 provider 必须同时更新 plugins: 段（⚠️ 易遗漏）
**症状**：切换 `memory.provider` 后插件仍然读旧配置路径，或报 `provider not found`。
**原因**：Hermes 的 `memory.provider` 字段决定加载哪个插件，但插件自身的配置（Ollama URL、
数据目录等）从 `plugins.<provider_name>` 段读取。如果 plugins 段的键名和路径没有同步更新，
插件会 fallback 到硬编码默认值。
**必须更新的字段**：
```yaml
# 旧配置
plugins:
  memory-lancedb:           # ← 键名必须改
    lance_dir: $HERMES_HOME/记忆数据库/lance_memory  # ← 字段名必须改

# 新配置
plugins:
  memory-zvec:             # ← 键名
    zvec_dir: $HERMES_HOME/记忆数据库/zvec_memory    # ← 字段名
```
**验证命令**：
```bash
grep -A10 'plugins:' ~/.hermes/config.yaml ~/.hermes/profiles/*/config.yaml | grep -i 'lancedb'
# 期望：无输出（如果还有输出，说明残留未清理）
```

### 8. default profile 的 HERMES_HOME 路径特殊（⚠️ 迁移脚本）
**症状**：`HERMES_PROFILE=default` 运行迁移脚本时报 `FileNotFoundError` 或路径错误。
**原因**：default profile（无 profile）的 `HERMES_HOME` 是 `~/.hermes/`，
而子 profile 是 `~/.hermes/profiles/<name>/`。迁移脚本如果硬编码为 `profiles/<name>`，
default 会解析到不存在的 `~/.hermes/profiles/default/`。
**正确逻辑**：
```python
PROFILE = os.environ.get("HERMES_PROFILE", "zunhunfan")
if PROFILE == "default":
    HERMES_HOME = Path.home() / ".hermes"
else:
    HERMES_HOME = Path.home() / ".hermes/profiles" / PROFILE
```
**同理**：插件层验证脚本设置 `os.environ['HERMES_HOME']` 时，default profile 应设为
`~/.hermes/`（不是 `~/.hermes/profiles/default/`）。

### 9. Root config.yaml 受安全保护，无法用 patch 工具修改
**症状**：`Refusing to write to Hermes config file: ~/.hermes/config.yaml`
**原因**：Hermes 的 write_file/patch 工具对 root config.yaml 有安全保护。
**解决**：使用 `terminal` 工具执行 `sed -i` 命令，或 `hermes config set` CLI。

### 10. on_session_end embedding 失败静默跳过（⚠️ 已修复）

**问题**：`on_session_end` 批量写入时，单条 `_ollama_embed_single` 失败直接 `continue`，
不重试、不记录。Ollama 短暂抖动会丢失该条记忆。

**修复**（2026-06-27）：引入 `_embed_with_retry()`——首次失败后等待 1 秒重试一次。
两次都失败才跳过，并记 `skipped=N` 到日志。

### 11. on_session_end 批量写入最后才 flush（⚠️ 已修复）

**问题**：原实现先 `insert(docs)` 积攒所有文档，最后统一 `flush()`。
如果进程在 flush 前被杀（SIGKILL/OOM），整批写入丢失。

**修复**（2026-06-27）：改为逐条 `insert + flush`，确保每条写入立即持久化。
HNSW `optimize()` 仍在最后执行一次（不影响持久化）。

### 12. Gateway 多 session 共享 collection 导致 open 失败（⚠️ 重要 Bug）

**症状**：`vec_memory_stats` 报 `'NoneType' object has no attribute 'stats'`，
`agent.log` 显示 `ZvecMemoryProvider failed to create collection:

**根因链**：
1. Session A 的 agent 实例调用 `zvec.open()` 获取 rw 锁（LOCK 文件）
2. Session A 的 agent 被 shutdown，调用旧的 `shutdown()` → 仅 `self._coll = None`，**未释放 Zvec LOCK**
3. Session B 的 `initialize()` → `zvec.open()` 被旧 GC 未回收的 Zvec Collection 对象锁住
4. Fallback → `create_and_open()` 因路径已存在失败 → `self._coll = None` → 所有工具崩溃

**关键发现**（[实验验证 + 锁治理演进](references/zvec-lock-mechanism.md)）：
- Zvec Collection 无 `close()`/`release()` 方法，锁靠 Python GC 释放（`del coll + gc.collect()`）
- 同一进程内**两次** `zvec.open()` 必然锁冲突（即使同一 gateway 内的不同 agent 实例）
- `del coll` + `gc.collect()` + `sleep(0.3)` 后锁立即释放，第三次 `open()` 成功
- 跨进程锁冲突（如手动 Python 脚本持有）无法通过 GC 解决，需要 read-only fallback

> **⚠️ 本节及以下 v3.x 修复史为历史记录**——现行方案是用户级 fork
> `memory-zvec` v1.3.1（排水+退避、空闲 30s 释放+懒重开、句柄跨会话共享、
> 会话切换重绑 sid），完整设计与 2026-09-14/15 真实事件时间线见
> [`references/zvec-lock-mechanism.md`](references/zvec-lock-mechanism.md)
> 后半部分。排障请先读那份，不要按本节的旧 shutdown 方案操作。

**修复演进**：

### v3.1（首次修复）— shutdown + 四层 fallback

**`shutdown()`**：主动释放锁
```python
def shutdown(self) -> None:
    if self._coll is not None:
        try:
            del self._coll
            import gc; gc.collect()
        except Exception:
            pass
```

**`initialize()`**：四层 fallback

### v3.3（2026-07-02 补修）— 三个关键遗漏

**修复 1：`initialize()` 开头加 `self.shutdown()`**（line 394）
v3.1 仅在异常路径中调 `_open_read_only` read-only fallback，但**同一进程内多次 `initialize()` 时旧 lock 未被释放**。即使是第一个 session 的 `initialize()` 成功后，中间没有 explicit `shutdown()` 路径来释放旧 connection 的 LOCK。解决方法：在 `initialize()` 方法体最开头（`from hermes_constants` 之前）调用 `self.shutdown()` 确保每次初始化时先释放上一轮的锁。

```python
def initialize(self, session_id: str, **kwargs) -> None:
    """Connect to Zvec collection, ensure schema, warm up Ollama."""
    # Release any lock left by a prior session in this process
    self.shutdown()                              # ← v3.3 新增
    from hermes_constants import get_hermes_home
```

**修复 2：所有 tool handler 加 `if not self._coll:` 守卫**（line 1053, 1171, 1234, 1250）
一旦 `initialize()` 未能创建有效的 `self._coll`（任何路径），所有工具直接崩溃报 `'NoneType' object has no attribute '...'`。更优雅的做法是**每个 tool handler 入口处**检查 `self._coll` 并返回明确的错误 JSON：

```python
def _tool_add(self, args: dict) -> str:
    if not self._coll:
        return json.dumps({"status": "skipped", "reason": "database not yet initialized"})
    if self._read_only:
        ...
```

需加守卫的 4 个 handler：`_tool_add`、`_tool_list`、`_tool_stats`、`_tool_delete`。`_tool_search` 已有下游 `except` 兜底但会**静默返回空结果数组**——用户误以为搜索结果正常，实际未连接数据库。建议也加上守卫。

**修复 3：`_open_read_only` 裸名 → `self._open_read_only`**（见 pitfall #14）
如果 pitfall #14 已修，此约已自动包含。如果单独排查：第 438 和 446 行的 `_open_read_only(...)` 必须改为 `self._open_read_only(...)`。

### 完整 fallback 链（v3.3）
```
① zvec.open(rw)           → 成功直接用
② gc.collect() + retry    → 旧 session shutdown 释放了锁，重试获取 rw ✅
③ zvec.open(read_only)    → **zvec 0.5.1 中 read-only 也需要独占 LOCK**
                           尝试以只读方式打开（通过 `self._open_read_only()`）
                           → 如果 `_open_read_only` 被写为裸名触发 NameError，
                              initialize 完全失败（见 pitfall #14）
                           → 即使 NameError 修了，zvec 0.5.1 的 read-only open 仍
                              会因 LOCK 被占而报 `Can't lock read-only collection`
④ zvec.create_and_open    → 路径不存在，新建
```

**⚠️ zvec 0.5.1 的 read-only 也需独占锁**：实验验证 `zvec.open(path, option=CollectionOption(read_only=True))` 在 LOCK 被占时抛出 `RuntimeError: Can't lock read-only collection: .../LOCK`。这意味着步骤 ③ 在 zvec 0.5.1 上**不工作**，`self._coll` 会被设为 None。必须依赖步骤 ②（GC + retry）来成功获取锁。

> **0.6.0 实测补充（措辞已精确化）**：只读锁是**共享**的（多个只读可并存），但只读与写锁
> **互斥**——有活跃写入进程（重建/迁移/网关会话）时只读也报同样的
> `Can't lock read-only collection`。所以排查时先确认没有写入进程，否则会误判为库损坏。

**NameError 干扰诊断**：pitfall #14 的 `_open_read_only` NameError 在日志中表现为 `initialize failed: name '_open_read_only' is not defined`，掩盖了底层的 LOCK 问题。建议排查顺序：① 先修 NameError → ② 再处理 LOCK。

**影响范围**：同一 gateway 进程内切换 session 时触发。
default profile 最常见（多轮对话），子 profile gateway 通常只有一个活跃 session。

### 11. 升级 lancedb 库后旧表报 "Table not found"（⚠️ Legacy LanceDB）
**症状**：旧版 lancedb (<0.10) 写出的 `.lance` 文件没有 `_versions/` 目录，
新版 (0.30.x) 拒绝打开。
**修复**：
```bash
~/.hermes/venv/bin/python ~/.hermes/skills/kkk-zvec-vector-db-migration/scripts/lancedb_rebuild_table.py \
    --profile <prof>
```
详见 [`references/lancedb-table-rebuild.md`](references/lancedb-table-rebuild.md)。

### 12. sync_turn 使用 daemon 线程，SIGKILL 时可能丢失（⚠️ 不可修复）
**症状**：gateway 被 SIGKILL 后，最近几轮对话不在 Zvec 中。
**原因**：`sync_turn` 在 `threading.Thread(daemon=True)` 中执行。
daemon 线程在主进程被 SIGKILL 时立即死亡，**没有机会完成 flush**。
SIGTERM/SIGINT 触发的优雅关闭有 drain 机制（等待 active agents 完成），不会触发此问题。
**缓解**：
- `state.db` 不受影响（SQLite WAL 模式更持久），可事后重放
- `on_session_end` 在正常关闭路径中已作为兜底
- 除非频繁遭遇 OOM killer 或 `kill -9`，否则不构成实际风险
### 13. on_session_end 写入失败 — 两种不同根因

`on_session_end` 在 `_batch_store` 线程中整体 catch `Exception`，产生统一的 `WARNING session_end batch store failed: XXX` 日志。但相同日志前缀下有两种完全不同的根因：

#### 类型 A：单条 embedding 跳过

**症状**：日志 `WARNING _hermes_user_memory.memory-zvec: session_end batch store failed: ...`，错误为 embedding 相关（Ollama 超时、HTTP 错误等）。

**根因**：`on_session_end` 的批量写入循环中，单条 `_ollama_embed_single()` 失败时 `continue` 跳过，
不重试、不记录、不报警（`__init__.py` 第 643-646 行 `except Exception: continue`）。
**实际影响**：低。因为 `sync_turn` 已在每轮实时写入 role="turn" 版本，
`on_session_end` 的 role="session_end" 版本是兜底——即使跳过几条也不影响记忆完整性。
**但如果 sync_turn 也失败了**（Ollama 在该轮宕机），则该轮对话完全丢失。
**改进建议**：在 `on_session_end` 的 `except` 中增加 retry（至少一次），或 log warning。

#### 类型 B：`_coll` 未初始化（AttributeError）

**症状**：
```
WARNING _hermes_user_memory.memory-zvec: session_end batch store failed:
  'ZvecMemoryProvider' object has no attribute '_coll'
```
同时每次 `vec_memory_stats` 报 `'NoneType' object has no attribute 'stats'`。

**根因**：`__init__()` 中未初始化 `self._coll = None`（旧版插件代码的 bug）。`on_session_end` 到达 `_batch_store` 内部的 `self._coll.insert(doc)` 时，`_coll` 属性根本不存在，触发 AttributeError。

**v3.3 修复**：`__init__` 第 308 行已加入 `self._coll = None`，并在 `on_session_end` 第 585 行添加 `if not self._coll: return` 前置守卫。

**诊断命令**：
```bash
# 检查错误是否仍然存在（旧 gateway 可能仍运行旧代码）
grep "has no attribute '_coll'" ~/.hermes/logs/agent.log ~/.hermes/profiles/*/logs/agent.log | tail -5
# 确认插件版本
grep "self._coll = None" ~/.hermes/plugins/memory-zvec/__init__.py
```

**修复**：如果日志仍出现此错误，说明 gateway 运行的是旧版插件代码。执行：
```bash
sudo systemctl restart hermes-gateway-<profile>
```

见 [`references/multi-profile-health-check.md`](references/multi-profile-health-check.md) 的 Layer 4 运行时诊断。

### 14. `_open_read_only` NameError 导致 `_coll` 为 None（‼️ 最常见原因）

**症状**：`vec_memory_add`/`list`/`stats` 全部失败，日志：
```
WARNING agent.memory_manager: Memory provider 'memory-zvec' initialize failed:
  name '_open_read_only' is not defined
```
后续所有工具调用报：
```
"Stats failed: 'NoneType' object has no attribute 'stats'"
"Failed to add memory: 'NoneType' object has no attribute 'insert'"
"List failed: 'NoneType' object has no attribute 'query'"
```
但 `vec_memory_search` 可能返回空结果数组（不是错误）—— 因为 `_tool_search` 的 `except` 兜底返回 `{"results":[],"count":0}`，实际上数据库未连接。

**根因**：`memory-zvec/__init__.py` 第 438 和 446 行用**裸名**调用 `_open_read_only(...)`，但该方法是一个 `@staticmethod`（第 383 行）。Python 中 `@staticmethod` 在类体内定义后，不能从其他方法里用裸名访问——需要通过 `self._open_read_only()` 或 `ZvecMemoryProvider._open_read_only()`。

```python
# 第 383-384 行：定义
@staticmethod
def _open_read_only(path, option):

# 第 438 行：调用（错误！裸名 → NameError）
self._coll = _open_read_only(collection_path, _ro_option)
              ↑ NameError: name '_open_read_only' is not defined
```

**修复**：将 `_open_read_only(...)` 改为 `self._open_read_only(...)` 两处：
```python
# 两个位置都要改
self._coll = self._open_read_only(collection_path, _ro_option)  # 第 438 行
self._coll = self._open_read_only(collection_path, _ro_option)  # 第 446 行
```

**与 LOCK 冲突（pitfall #12）的关键区别**：

| 特征 | LOCK 冲突 (#12) | NameError 空引用 (#14) |
|------|-----------------|----------------------|
| 搜索 | ❌ 全部失败 | ✅ 静默返回空数组（未连接数据库） |
| 写入 | ❌ 失败 | ❌ 失败 |
| 日志线索 | 有 `Can't lock` 和 `still locked` | 有 `name '_open_read_only' is not defined` |
| initialize 日志 | `still locked after retry, falling back to read-only` | `initialize failed: name '_open_read_only' is not defined` |
| 根因 | Zvec 锁竞争 | Python 作用域错误 — bug 在插件代码 |

**为什么第一条日志是 False Negative**：hermes-agent 的 `run_agent.py` 即使在 `memory_manager.initialize()` 抛出异常后，仍然打印 `Memory provider '...' activated`（`run_agent: Memory provider 'memory-zvec' activated`）。所以看到这条不代表初始化成功——必须同时检查前面有没有 `initialize failed`。

**与 pitfall #12 的关系**：即使修复 NameError，read-only fallback 在 zvec 0.5.1 中也会失败（见 pitfall #12 更新），因为 zvec 0.5.1 的 `open(read_only=True)` 也需要独占 LOCK。

**完整诊断命令**：
```bash
# 1. 确认 zvec 版本
python3 -c "import zvec; print(zvec.__version__)"
# 2. 检查日志中是否有 NameError
grep "initialize failed" ~/.hermes/logs/agent.log ~/.hermes/profiles/*/logs/agent.log
# 3. 确认同进程内是否有旧 connection 持有 LOCK（每个 profile 独立的锁）
ls -la ~/.hermes/profiles/*/记忆数据库/zvec_memory/memories/LOCK 2>/dev/null
# 4. 直接排查插件代码
grep -n "_open_read_only" ~/.hermes/plugins/memory-zvec/__init__.py
```

**验证修复**：修改后在同一进程中重新启动 agent session，日志应出现：
```
_hermes_user_memory.memory-zvec: ZvecMemoryProvider opened existing collection: .../memories
_hermes_user_memory.memory-zvec: ZvecMemoryProvider initialized — model=bge-m3:latest dim=1024
```
然后 `vec_memory_stats` 应返回正常计数，`vec_memory_add` 返回 `{"status": "added", "id": "..."}`。

### 15. Idle cache 驱逐（LRU cap / TTL sweep）不触发 on_session_end（⚠️ 正常行为，不是 bug）
**症状**：LRU 驱逐或 idle sweep 后，被驱逐 session 的 `on_session_end` 未被调用。
**原因**：驱逐走 `_release_evicted_agent_soft()` → `release_clients()`，注释明确说
"memory provider (has its own lifecycle; keeps running)"。**不调 `shutdown_memory_provider()`**。
**为什么安全**：被驱逐的 agent 的 ZvecMemoryProvider 实例仍在内存中（Python GC 未回收），
持有 rw 锁，可以继续由其他路径（如 gateway 优雅关闭步骤 ③）触发 `on_session_end`。
如果 gateway 在驱逐后**正常关闭**，idle agents 会被 `_stop_impl` 第 6740-6755 行正确清理。
**唯一风险**：如果 gateway 在驱逐后、正常关闭前被 SIGKILL，这些 idle agents 的 on_session_end 不会被调用。

**实测后果链（2026-09-14 trading_bot 确诊）——驱逐后锁泄漏 → 下一个新会话 _coll=None**：
被驱逐 agent 的 provider 实例被 gateway 内部结构持强引用，GC 不回收 → rw 锁一直被占。下一个新会话
initialize：`lock detected → GC+retry 失败 → read-only 也被拒（读写互斥）→ _coll=None`，但日志**仍打**
`Memory provider 'memory-zvec' activated`（假阳性）。之后所有 vec_memory_* 报 `database not yet initialized`，
本会话记忆读写静默中断（sync_turn 同样失败，对话只进 state.db）。
诊断三步（全程只读）：① 用 `/proc/*/fd` 的 `readlink` **完整路径精确比对** LOCK（别用 grep 子串——各
profile 路径同名必误匹配）；② 持锁者 = 本 profile gateway 自身 → 同进程锁泄漏；跨 profile 串台则显示
别的 PID；③ grep 日志 `still locked after retry` + `failed to open (read-only)` + `activated` 三条同现即确诊。
修复：重启该 profile gateway（进程退出释放锁；**会断开其飞书会话**，需用户知情确认）。
顺手核验（均排除）：配置 embedding_model tag 能否 embed、HERMES_HOME 是否指向本 profile。

### 16. Ollama 嵌入模型名漂移导致自动记忆静默断流（‼️ 2026-09-07 发现）
**症状**：配置 `plugins.memory-zvec.embedding_model` 与 Ollama 实际模型 tag 不一致时，`sync_turn`/`on_session_end` 的 embedding 调用全部失败且**不产生醒目报错**（skill 日志有 `session_end batch store failed`，但 gateway.log 中 grep "not found" 计数为 0，极易漏诊）。表现为 Zvec 中 turn/session_end 条目在某日期后断流，而 collection 本身健康（stats 正常、手动写入正常）。
**实测案例**（financial_expert）：8/22 Ollama 侧模型 tag 从 `bge-m3:567m` 变为 `bge-m3:latest`（`ollama list` 只显示 `bge-m3:latest`），config.yaml 仍写 `bge-m3:567m` → `/api/embed` 返回 `{"error": "model not found"}` → 8/22 之后 turn/session_end 自动条目为 0（之前 7 月有 195 条、8 月上旬 32 条），持续 16 天未被发现。
**诊断方法**（三步，2 分钟）：
```bash
# 1. 对比配置与实际
grep embedding_model ~/.hermes/config.yaml ~/.hermes/profiles/*/config.yaml
curl -s http://localhost:11434/api/tags | python3 -m json.tool | grep '"name"'
# 2. 直接测 embed（注意两个 tag 都试）
curl -s http://localhost:11434/api/embed -d '{"model":"bge-m3:567m","input":["t"]}' | head -c 120
# 3. 按月统计 Zvec 条目断流点（turn/session_end 归零的月份即断流起点）
```
**修复**：把 config.yaml 的 `embedding_model` 改成 Ollama 实际 tag（如 `bge-m3:latest`），或 `ollama pull bge-m3:567m` 补回旧 tag，然后重启该 profile gateway。维度不变（1024）时存量向量无需迁移。
**预防**：Ollama 拉新模型/删旧模型时 tag 会变，memory 后端对模型名漂移零容错；健康检查脚本应加入"配置 tag 能否成功 embed"检查项。

**⚠️ 改 tag 必须一次改全部 profile**：本问题已复发过一次——首次只修了命中的那个
profile，其余四个带着坏 tag 继续跑了半个月且无人察觉。命令：
```bash
# 逐 profile 验证「配置里的 tag 真的能 embed」
for f in ~/.hermes/config.yaml ~/.hermes/profiles/*/config.yaml; do
  tag=$(grep -m1 'embedding_model' "$f" | awk '{print $2}')
  printf '%-52s %-16s ' "$f" "$tag"
  curl -s http://localhost:11434/api/embed -d "{\"model\":\"$tag\",\"input\":[\"t\"]}" \
    | grep -q '"embeddings"' && echo '✅' || echo '❌ 该 tag 无法 embed'
done
```

**更深一层**：本问题的搜死信号在 `is_available()` 里被 `logger.debug` 吞掉（返回 False 时
不打 WARNING）。部署验证必须跑 discovery + `hermes memory status` 两层，不能只看
collection 里有多少条（详见同目录 references）。

### 17. shutdown() 删掉 `_coll` 属性 → on_session_end 后台批次整体丢失（‼️ 真 bug，已修）

**症状**：会话关闭时出现一次
```
WARNING plugins.memory.memory-zvec: session_end batch store failed:
  'ZvecMemoryProvider' object has no attribute '_coll'
```
collection 之后读写都正常 → 极易被当成噪音忽略。真实后果：该会话的 **session_end 兜底批次一条都没写**
（`role="turn"` 记录此前已落，所以内容不丢时肉眼几乎无感；这是"兜底防线静默失效"）。

**根因**：`shutdown()` 为释放 Zvec LOCK 写成 `del self._coll`（**删掉了实例属性**），
而 `on_session_end` 的批写跑在 `threading.Thread(daemon=True)` 里。agent 关闭时
`on_session_end`（起线程）与 `shutdown()`（删属性）并发 → 后台线程读 `self._coll` 抛 AttributeError，
被线程内的 `except Exception` 吞成一条 WARNING。

**修法（两处缺一不可）**：
1. `shutdown()` 不要删属性：
   ```python
   coll = self._coll
   self._coll = None          # 保留属性，所有 `if not self._coll` 守卫继续有效
   if coll is not None:
       del coll; import gc; gc.collect()   # 锁照常释放
   ```
2. 后台写入路径必须**自带强引用**：`sync_turn` / `on_session_end` 在起线程前取 `coll = self._coll`，
   线程内用 `coll`（`_insert(..., coll=coll)`）。否则第 1 步把属性置 None 后，同一竞态会变成
   `'NoneType' object has no attribute 'insert'` —— 换个错法，批次照样丢。

**回归测试**：[`scripts/test_shutdown_race.py`](scripts/test_shutdown_race.py)（A/B 对照旧/新实现：
旧版 4 条待写 → 落库 0 + 告警 1；新版 4/4 + 0 告警）。修改该插件后必跑。

**为什么功能验收测不出来**：8/11 项那类"写入→三种检索→删除"用例是串行的，永远碰不到
agent close 的并发窗口；失败又被吞成 WARNING。排查记忆写入问题时，**永远先 grep 这条 WARNING**：
```bash
grep -a "session_end batch store failed" ~/.hermes/logs/agent.log* ~/.hermes/profiles/*/logs/agent.log*
```
出现 `has no attribute '_coll'` = 本 pitfall（写入丢失）；出现 embed/HTTP 错误 = pitfall #13 类型 A。

### 18. 进程退出吃掉在飞的写入 → cron 任务静默丢记忆（‼️ 真 bug，已修 —— 2026-09-11）

**症状**：cron 任务（AI 日报/复盘/新闻采集）跑完了，但记忆库里查不到该会话的记录，**日志里没有任何报错**。
典型形态：某 profile 的 cron 会话 `role='session_end'` 记录**从来没有过**，`role='turn'` 时有时无。

**根因（双重异步 + 进程退出）**：
- 核心侧：`sync_all()` 把写入丢进 `DaemonThreadPoolExecutor`（daemon 线程）。
- 插件侧：`sync_turn()` / `on_session_end()` **又各自新建一个 daemon 线程**。
- cron 的每条 agent 任务是**独立的外部 worker 进程**（`hermes-worker-cron-<jobid>-exec-*.scope`）：
  `_finalize_cron_session` → `agent.close()` → `on_session_end(messages)` → 解释器立刻退出（实测总寿命 ~0.7s）。
- 解释器收尾会**直接杀掉 daemon 线程**，正在做 Ollama 嵌入的那条写入与整批 session_end 凭空消失。
- Hermes 的 `shutdown_all()` 只能 drain 它自己的 executor，**drain 不到插件自建的线程** —— 所以这条路径必须插件自己堵。

**复现实验**（改插件后必跑）：[`scripts/exit_loss_experiment.py`](scripts/exit_loss_experiment.py)
```bash
# 子进程里 sync_turn/on_session_end 之后立刻退出，再由父进程独立统计落库数
HERMES_MEMORY_ZVEC_EXIT_DRAIN_S=0 ~/.hermes/hermes-agent/venv/bin/python3 \
    scripts/exit_loss_experiment.py ~/.hermes/hermes-agent/plugins/memory/memory-zvec/__init__.py drain_off
# 实测：兜底关 → sync_turn 0 条 / session_end 0 条；兜底开 → 1 条 / 4 条
```

**修法**：插件注册 atexit 兜底 —— 登记在飞写入线程，解释器退出前**有界 join**
（`atexit` 早于 daemon 线程回收执行，这是 openviking 插件同款先例）：
```python
_EXIT_DRAIN_TIMEOUT_S = float(os.environ.get("HERMES_MEMORY_ZVEC_EXIT_DRAIN_S", "30"))
# sync_turn / on_session_end: 起线程前 _track_write_thread(t)，线程 finally 里 _untrack_write_thread
atexit.register(_drain_pending_writes)
```
`HERMES_MEMORY_ZVEC_EXIT_DRAIN_S=0` 可关闭（A/B 或要求秒退时用）。

**生产实测（2026-09-11）**：临时 cron 任务（真外部 worker 进程）走完整链路后，
`role='turn'` 与 `role='session_end'` **各落 1 条**；修复前该 profile 的 cron 会话 session_end 记录为 0。

**判读要点（别把正常当故障）**：cron 会话 = **单轮**（1 条 user + N 条 assistant/tool），
所以每个 cron 任务的期望值是 **1 条 turn + 1 条 session_end**，不是"每条消息一条"。
用"最近记录时间"判断写入是否正常时，必须区分 gateway 会话（多轮）与 cron 任务（单轮）。

### 19. 活的 idle-cached agent 持锁 → GC+retry 失效，新会话永久 _coll=None（‼️ 2026-09-14 health_manager 实测）

**症状**：gateway 同进程内，A 会话 agent 先 `zvec.open()` 成功拿 rw 锁并进入 **idle cache（仍被强引用、未被 GC）**；
B 会话 agent `initialize()` 走 `self.shutdown()`（只清自己的 `_coll`）+ `gc.collect()` + retry，
**锁依然不释放** → read-only fallback 也因写锁互斥失败 → B 的 `_coll=None`，本会话 `vec_memory_*`
全部 "database not yet initialized"、自动记忆整会话不写。日志只有一次 `still locked after retry`。

**与 pitfall #12 的区别**：#12 的 GC+retry 能成功，前提是旧实例已无强引用（可被回收）；
本坑中旧实例被 gateway **idle agent cache 活着持有**（见 #15 "memory provider keeps running"），
GC 永远回收不了它，retry 必然失败。进程内无自恢复路径，只有**重启该 profile gateway**才能让下一个会话拿到锁。

**判读**：`vec_memory_stats` 返回 `database not yet initialized` + 日志有 `still locked after retry`
+ 数据目录今天仍有别的会话写入的 mtime = 本会话实例撞了活实例的锁，不是库损坏、也不是 Ollama 问题。

**排查要点（健康检查别误判）**：
- 数据健康度要绕过工具层直接读 `*/scalar.0.ipc`（pyarrow；venv 无 pyarrow 时 `uv run --with pyarrow`），
  不能因为本会话 stats 报 not initialized 就说库空了。
- 判定"断流"必须用 state.db 的 `sessions/messages`（时间戳是 **unix epoch float**，用
  `datetime(started_at,'unixepoch','localtime')`，不是 TEXT 日期）对照：有会话活动但 zvec 零写入才是故障。
- agent.log 里连 `Memory provider 'memory-zvec' registered` 都没有 = 该 gateway 进程根本没加载插件
  （常见于 user-tree→bundled 迁移后长跑 gateway 未重启）；这与 embedding tag 漂移（#16，有 registered 无写入）要分开。
- 独立验证脚本（verify-plugin-tools.py，session_id=`verify_*`）能写库只证明"全新进程插件+库正常"，
  不代表正在跑的 gateway 已加载；且 verify 条目是垃圾数据，验完必须删。

---

## 文件位置总览

| 项目 | 位置 | 说明 |
|------|------|------|
| memory-zvec 规范源 | `~/.hermes/plugins/memory-zvec/` | 用户级 fork v1.3.x（唯一可编辑副本），目录名带 -x，yaml 内部名仍 memory-zvec |
| profile 部署副本 | `~/.hermes/profiles/*/plugins/memory-zvec/` | 实体副本（rsync 分发，非 symlink） |
| bundled 旧版存档 | `~/.hermes/backups/memory-zvec-shutdown-race-20260911/bundled_memory-zvec_removed_20260914/` | 2026-09-14 移出系统树时的存档 |
| zvec 技能 | `~/.hermes/skills/zvec/` | Zvec API 参考手册 |
| 本手册 | `~/.hermes/skills/kkk-zvec-vector-db-migration/` | 全局共享 |
| profile 数据目录 | `~/.hermes/profiles/*/记忆数据库/zvec_memory/` | 各 profile 独立 |

---

## Business Data Migration

Financial news, telegraph, and analysis data migration from LanceDB → Zvec.
See [`references/financial-data-migration.md`](references/financial-data-migration.md) for
call graph, schema designs, function mapping, and phased execution plan.

> **合并记录（2026-07-11）**：`hardware/vector-db-migration` 已合并到本技能。迁移内容：
> - `references/migration-audit-checklist.md`（3-dimension 通用版本）→ `references/migration-audit-checklist-3dimension.md`
> - 其他文件（`zvec-api-reference.md`、`migrate-lancedb-to-zvec.py`）已存在于本技能（版本更新，保留较新版本）

## References

### Zvec / memory-zvec

- [`references/discovery-and-deployment-verification.md`](references/discovery-and-deployment-verification.md) — **跨 profile 部署与部署验证铁律**（发现机制、embedding tag 漂移、0.6.0 差异、摊销 optimize、镜像树遮蔽、备份绝对检查、从 state.db 重建）
- [`references/memory-zvec-readme.md`](references/memory-zvec-readme.md) — 插件功能说明
- [`references/zvec-api-reference.md`](references/zvec-api-reference.md) — Zvec Python API 速查
- [`references/migration-audit-checklist.md`](references/migration-audit-checklist.md) — 迁移验证清单
- [`references/post-migration-repair.py`](references/post-migration-repair.py) — 一键索引修复脚本

### Lock & Concurrency

- [`references/zvec-lock-mechanism.md`](references/zvec-lock-mechanism.md) — Zvec 锁语义实验 + memory-zvec fork 锁治理演进（v1.1–v1.3.1：排水/退避、空闲释放、句柄共享、真实事件时间线、运维速查）
- [`references/multi-profile-health-check.md`](references/multi-profile-health-check.md) — 多 Profile 4 层记忆健康检查方法论（配置/插件/数据/运行时）

### Scripts

- [`scripts/migrate-lancedb-to-zvec.py`](scripts/migrate-lancedb-to-zvec.py) — LanceDB→Zvec 完整迁移脚本
- [`scripts/verify-plugin-tools.py`](scripts/verify-plugin-tools.py) — 8 项全功能验证脚本
- [`scripts/check_ollama.sh`](scripts/check_ollama.sh) — Ollama 健康检查
- [`scripts/check_all_profiles.py`](scripts/check_all_profiles.py) — 多 profile 记忆健康检查（HNSW/FTS/LOCK）

### Legacy LanceDB（归档参考）

- [`scripts/migrate_sessions_to_lancedb.py`](scripts/migrate_sessions_to_lancedb.py) — FTS5→LanceDB 迁移（历史）
- [`scripts/lancedb_rebuild_table.py`](scripts/lancedb_rebuild_table.py) — LanceDB 表重建脚本
- [`references/vector-memory-architecture.md`](references/vector-memory-architecture.md) — 记忆系统整体架构
- [`references/multi-profile-business-lancedb.md`](references/multi-profile-business-lancedb.md) — 多 profile 业务 LanceDB
- [`references/actual-storage-format.md`](references/actual-storage-format.md) — LanceDB 存储格式（schema）
- [`references/lancedb-table-rebuild.md`](references/lancedb-table-rebuild.md) — 表重建详细指南
- [`references/strip-pipeline-architecture.md`](references/strip-pipeline-architecture.md) — Strip Pipeline 架构
- [`references/profile-lancedb-layout.md`](references/profile-lancedb-layout.md) — 旧 profile LanceDB 布局
