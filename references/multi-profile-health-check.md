# 多 Profile Zvec 记忆系统健康检查

## 4 层诊断方法论

系统性检查所有 Hermes profile 的记忆系统健康，按以下四层逐一排查：

```
Layer 1: 配置层  → memory.provider + plugins.memory-zvec 配置
Layer 2: 插件层  → 符号链接完整性 + 插件文件时间戳
Layer 3: 数据层  → zvec 数据目录、大小、最后写入时间
Layer 4: 运行时层 → 日志中的初始化/激活序列
```

### Layer 1: 配置层

检查每个 profile 的 `memory.provider` 是否指向 `memory-zvec`：

```bash
echo "=== default ===" && grep -A6 '^memory:' ~/.hermes/config.yaml 2>/dev/null
for p in <你的profile列表>; do
  echo "=== $p ==="
  grep -A6 '^memory:' ~/.hermes/profiles/$p/config.yaml 2>/dev/null || echo "NO CONFIG"
done
```

预期每项输出包含 `provider: memory-zvec`。

同时检查 `plugins:` 段下存在 `memory-zvec` 配置块（zvec_dir / embedding_model / vector_dim 等）：

```bash
for p in <你的profile列表>; do
  echo "=== $p plugins ==="
  grep -A10 '^plugins:' ~/.hermes/profiles/$p/config.yaml 2>/dev/null
done
```

### Layer 2: 插件层

检查插件本体是否存在，以及各子 profile 的符号链接是否完好：

```bash
echo "=== memory-zvec plugin ===" && ls -la ~/.hermes/plugins/memory-zvec/ | head -5
echo ""
for p in <你的profile列表>; do
  link="$HOME/.hermes/profiles/$p/plugins/memory-zvec"
  if [ -L "$link" ] && [ -d "$(readlink -f "$link")" ]; then
    echo "✅ $p: symlink OK → $(readlink "$link")"
  else
    echo "❌ $p: BROKEN or missing"
  fi
done
```

**关键检查**：插件 `__init__.py` 的修改时间戳，确认是否所有 profile 使用同一版本：

```bash
stat -c '%y %n' ~/.hermes/plugins/memory-zvec/__init__.py
```

如果插件刚更新过，但某 profile 的 gateway 未重启，则运行时仍用旧版代码。

### Layer 3: 数据层

检查各 profile 的数据目录是否存在、大小、最后活跃时间：

```bash
echo "=== default ===" && du -sh ~/.hermes/记忆数据库/zvec_memory/ 2>/dev/null && \
  ls -lt ~/.hermes/记忆数据库/zvec_memory/memories/manifest.* 2>/dev/null | head -1

for p in <你的profile列表>; do
  dir="$HOME/.hermes/profiles/$p/记忆数据库/zvec_memory/"
  if [ -d "$dir" ]; then
    echo "=== $p ==="
    du -sh "$dir" 2>/dev/null
    manifest=$(ls -t "$dir"/memories/manifest.* 2>/dev/null | head -1)
    if [ -n "$manifest" ]; then
      echo "  last manifest: $(stat -c '%y' "$manifest")"
    fi
  else
    echo "❌ $p: NO DATA DIR"
  fi
done
```

manifest 的最后修改时间是判断该 profile 是否有最近记忆写入的最可靠指标（比 agent.log 更直接）。

### Layer 4: 运行时层 — 日志激活序列

读取各 profile 的 agent.log，检查 memory-zvec 的初始化序列是否完整：

```bash
for p in <你的profile列表>; do
  if [ "$p" = "<profile>" ]; then
    logfile="$HOME/.hermes/profiles/$p/logs/agent.log"
  else
    logfile="$HOME/.hermes/profiles/$p/logs/agent.log"
  fi
  echo "=== $p ==="
  if [ -f "$logfile" ]; then
    # 检查完整初始化序列
    grep "Memory provider.*memory-zvec registered\|ZvecMemoryProvider.*opened existing\|ZvecMemoryProvider initialized\|Memory provider.*activated\|initialize failed" "$logfile" 2>/dev/null | tail -5
  else
    echo "  No agent.log"
  fi
  echo ""
done
```

#### 正常激活序列（3 个关键日志）

```
① Memory provider 'memory-zvec' registered (5 tools)     ← 插件被发现并注册 5 个工具
② ZvecMemoryProvider opened existing collection: ...      ← 成功打开 zvec collection
③ ZvecMemoryProvider initialized — model=... dim=...     ← 初始化完成（含 Ollama 预热）
④ run_agent: Memory provider 'memory-zvec' activated      ← hermes-agent 确认激活
```

**⚠️ 常见误判**：hermes-agent 即使 `initialize()` 抛出异常，也会打印第 ④ 条日志。所以看到 `activated` **不代表初始化成功**——必须同时看到第 ②/③ 条。

#### 异常模式对照表

| 日志特征 | 根因 | 影响 |
|---------|------|------|
| 仅有 ① + ④，缺 ②③ | `initialize()` 异常退出（NameError / LOCK 等） | `_coll=None`，所有工具报 NoneType |
| `initialize failed: name '_open_read_only' is not defined` | 插件代码中裸名调用 static method（pitfall #14） | `_coll` 始终为 None |
| `still locked after retry, falling back to read-only` | Zvec LOCK 被旧进程或旧 session 持有（pitfall #12） | 见 pitfall #12 的 fallback 链 |
| `session_end batch store failed: 'ZvecMemoryProvider' object has no attribute '_coll'` | `__init__` 未初始化 `self._coll = None` | each session end 报 warning，无实际写入 |
| `session_end batch store failed: ...`（其他错误） | `_batch_store` 线程中异常 | 整批写入失败，`sync_turn` 已兜底 turn 版本 |

### 综合判断矩阵

| Profile | 配置 | 插件 | 数据 | 运行时 | 结论 |
|---------|------|------|------|--------|------|
| default | ✅ | ✅ | ✅ 76MB/694条 | ✅ 正常 | ✅ |
| <profile1> | ✅ | ✅ (旧代码) | ✅ 20MB | ⚠️ NameError → 只读 | 需重启 gateway |
| <profile2> | ✅ | ✅ (旧代码) | ✅ 7.8MB | ⚠️ NameError → 只读 | 需重启 gateway |
| <profile3> | ✅ | ✅ | ✅ 19MB | ✅ 正常（新版代码） | ✅ |
| <profile4> | ✅ | ✅ | ✅ 6.0MB | ❓ 近期无对话 → 未激活 | 收到下条消息自动加载 |

### 修复后验证：重启 gateway

插件更新后，旧 gateway 进程仍在内存中运行旧版代码。需重启：

```bash
sudo systemctl restart hermes-gateway-<profile>
```

重启后检查日志是否出现完整激活序列（三条关键日志齐全）：

```bash
grep -E "ZvecMemoryProvider opened|ZvecMemoryProvider initialized|Memory provider.*activated" \
  ~/.hermes/profiles/<profile>/logs/agent.log | tail -5
```

然后用 `vec_memory_stats` 确认返回正常计数而非 NoneType 错误。

### 参考

- 对应踩坑记录：pitfall #13（session_end 写入失败）、#14（_open_read_only NameError）、#12（LOCK 冲突）
- 自动化检查脚本：[`scripts/check_all_profiles.py`](../scripts/check_all_profiles.py)（数据层检查，不含运行时）
