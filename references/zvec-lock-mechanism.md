# Zvec Lock Mechanism — 实验验证 + 锁治理演进（2026-09-15 更新）

> 本文前半是 zvec 0.5.x/0.6.0 锁语义的实验结论（仍然有效）；后半记录
> memory-zvec fork（v1.1.0→v1.3.1）的锁治理设计与真实事件——旧的
> "shutdown 释放 + gc retry" 方案已废弃，不要再按旧文档操作。

## 锁类型与基础语义（实验结论）

文件级 LOCK，位于 `<collection_path>/LOCK`。

| 操作 | 结果 |
|------|------|
| `zvec.open()` | 获取 rw 锁。占用中 → `RuntimeError: Can't lock read-write collection` |
| `zvec.open(option=read_only)` | **zvec ≥ 0.5.1 完全互斥**：仍需获取锁，占用中 → `Can't lock read-only collection`（read-only **不是**绕锁手段，实测 0.6.0 亦然） |
| `zvec.create_and_open()` | 同 open，获取 rw 锁。路径必须不存在 |
| Collection 无 `close()`/`release()` | 唯一释放 = Python 对象引用归零 + `gc.collect()`，或进程退出（fd 关闭锁落） |

**GC 释放实验**：`del coll1; gc.collect()` 即释放，无需 sleep。

**同进程二次 open 同一路径**：也会撞锁（锁按路径全局判定，不区分进程内外）。
**切勿在自己进程已持有句柄时再次 `zvec.open()` 同路径**——会撞自己的锁，
退避失败还可能把好句柄覆盖丢失（v1.3.0 首版真实引入过，已修：句柄复用）。

## 锁治理演进（memory-zvec fork）

### 事件时间线（真实事故）

| 日期 | 事件 |
|------|------|
| 2026-09-10 | in-tree 补丁 optimize_fix（热路径 HNSW 摊销优化） |
| 2026-09-11 | in-tree 补丁 shutdown race 修复（线程跟踪 + atexit 排水）——但排水没挂到 initialize，race 未根治 |
| 2026-09-14 14:07 | chip_expert 会话初始化撞上一会话收尾批写，**整会话记忆静默失效**（journal 特征：同秒 WARNING "still locked" + ERROR "failed to open (read-only)"） |
| 2026-09-14 23:00 | 插件迁出 hermes-agent 树 → `~/.hermes/plugins/memory-zvec`（用户级，update 免疫），6 个 config 指 `memory.provider: memory-zvec` |
| 2026-09-14 深夜 | 发现 dashboard（root profile）web 端**内嵌 profile 聊天**（`web_server_chat.py` 支持 `profile=`，HERMES_HOME 指向子 profile）持 zunhunfan 锁 **35 小时**（fd 时间戳实锤）——从此引出空闲释放 |
| 2026-09-15 | 对抗审查修复 8 项（v1.3.1），profile 符号链接改实体副本 |

### v1.1.0 — 背靠背会话 race 根治

`initialize()` 打开 collection 前 `_drain_pending_writes(timeout=15s)`
（等待本进程在途写线程，`HERMES_MEMORY_ZVEC_INIT_DRAIN_S` 可调）；外部持锁
退避重试 0.5/1/2/4s；全失败才降级只读（有告警）。

### v1.2.x — 空闲自动释放 + 懒重开

- 模块级 watchdog（周期 `min(5s, idle/3)`）对"打开但空闲超 30s 且无在途写
  线程"的 provider 释放锁（`HERMES_MEMORY_ZVEC_IDLE_RELEASE_S`，0=禁用）。
- 所有公共入口（prefetch/queue_prefetch/sync_turn/on_session_end/
  handle_tool_call）经 `_ensure_open()` 懒重开，实测重开+stats ≈ 188ms。
- 仅上次开库**失败**才受 30s 重试限流；空闲释放后的重开立即执行。

### v1.3.0 — 句柄跨会话共享（new 零等待）

- `shutdown()` 改为纯簿记**不关句柄**：collection 句柄跨会话共享，进程内写入
  本就靠 `_insert_lock` 串行；新会话 initialize 实测 136ms→0ms（不再等旧批写）。
- 真实释放只走 idle watchdog 的 `_release_collection()` 或进程退出。
- initialize 复用已开句柄（目标 zvec_dir 变了才切换）。

### v1.3.1 — 对抗审查修复

- `_read_only` 每次开库阶梯开头复位（旧 bug：一次降级永久跳写）。
- 实现 `on_session_switch` 重绑 `_session_id`（旧 bug：/new 后写入永远打
  第一个会话 id）；on_session_end 批写**入口快照 sid**（防切换竞态串号）。
- 读路径局部引用捕获（防 idle 释放竞态 NoneType）。
- 懒重开用**短阶梯**（排空 2s + 退避 0.5/1/2，最坏 ~5.5s 而非 22.5s）。
- 跨进程黑窗跳写加节流告警（1/5min；会话结束批写兜底补全）。
- prefetch 缓存上限 128；句柄复用时 /new 免 Ollama warmup。

## 现行锁语义（速查）

| 场景 | 行为 |
|------|------|
| 新会话 /new（含旧批写进行中） | ~0ms，不等锁（句柄共享） |
| 写入进行中 | 持锁（嵌入+落盘，几百 ms/条），进程内 `_insert_lock` 排队 |
| 最后一笔写完 | ~30s 后 watchdog 自动释放 |
| 下次访问 | ~200ms 内懒重开 |
| 跨进程争抢 | 排水+退避自动解决；30s 空闲保证轮转；黑窗期跳写有节流告警、会话结束全量补写 |

## 运维要点

- **持锁诊断**：`fuser <collection>/LOCK` → 对 PID 查
  `ls -l /proc/<pid>/fd | grep 记忆`（fd 时间戳=持锁起点）+
  `tr '\0' ' ' < /proc/<pid>/cmdline`（身份）。
- **释放手段**：等 30s 自动释放；或 restart 持锁进程的 systemd unit
  （dashboard 重启低风险、不影响 gateway）。
- **journal 特征**：释放=INFO `idle Ns — releasing collection lock`；
  黑窗=WARNING `collection unavailable — per-turn writes skipped`；
  降级=WARNING `still locked after backoff retries`。
- **体检被锁库**（勿硬开）：扫 RocksDB `LOG`/`LOG.old` 尾部找
  corruption/checksum、验 `CURRENT→MANIFEST-*` 链、查 segment 数（≈2 正常）。

## 验证脚本（锁语义自检）

```python
import zvec, gc
path = "/path/to/memories"
coll1 = zvec.open(path)
try:
    coll2 = zvec.open(path)          # 预期失败：同进程二次 open 撞自己的锁
except RuntimeError:
    print("self-collision as expected")
del coll1; gc.collect()
coll3 = zvec.open(path)              # GC 后可重开
print("reopen OK:", coll3.stats.doc_count)
```

provider 层功能自测（不碰生产库，用 /tmp 一次性 collection）：

```bash
HERMES_HOME=<profile home> ~/.hermes/hermes-agent/venv/bin/python -c "
import sys; sys.path.insert(0, '/home/<user>/.hermes/hermes-agent')
from plugins.memory import load_memory_provider
p = load_memory_provider('memory-zvec', register_skills=False)  # 加载模块名应为 _hermes_user_memory.memory-zvec__source_*
import json; p.initialize('t'); print(json.loads(p.handle_tool_call('vec_memory_stats', {})))"
```
