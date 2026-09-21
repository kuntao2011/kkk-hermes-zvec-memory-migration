# Memory Write Hooks — Hermes 侧编排逻辑

## 写入路径

```
run_conversation() 完成
  → finalize_turn()                          # turn_finalizer.py:441
    → _sync_external_memory_for_turn(...)
      → if interrupted: return                # 中断轮次跳过 (run_agent.py:3112)
      → memory_manager.sync_all(user, response)  # 后台线程
        → provider.sync_turn(...)                 # role="turn", daemon 线程
      → memory_manager.queue_prefetch_all(...)    # 预热下一轮检索

/new, /reset, CLI exit, session expiry
  → commit_memory_session(messages)             # session 轮转 (run_agent.py:3060)
    → memory_manager.on_session_end(messages)   # 后台线程
      → provider.on_session_end(...)             # role="session_end"

上下文压缩（conversation_compression.py:525）
  → agent.commit_memory_session(messages)        # 压缩前触发，确保被压缩的 turns 已入库

真正关闭（gateway shutdown / CLI exit）
  → shutdown_memory_provider(messages)
    → memory_manager.on_session_end(messages)   # 同上
    → memory_manager.shutdown_all()
      → _drain_sync_executor(timeout=5s)         # 等待后台 sync 线程完成
      → provider.shutdown()                       # del self._coll + gc.collect()
```

## sync_turn vs on_session_end

| 维度 | sync_turn | on_session_end |
|------|-----------|-----------------|
| 触发 | 每轮完成后 | /new /reset /exit / compression |
| 内容 | 单轮 user+assistant | 全 session 所有轮次 |
| role | "turn" | "session_end" |
| 线程 | daemon=True | MemoryManager 后台 worker |
| 被中断跳过 | 是（`if interrupted: return`） | N/A（结束时才触发） |
| Ollama 故障时 | 整体 try/except 静默失败（该轮丢失） | 逐条 try/except continue（跳过失败条目） |
| 双写 | 可能 + on_session_end 重叠 | 是，同一对话存两条 |

## 中断轮次处理

`_mirror_to_memory()` 源码 (run_agent.py ~3112)：
```python
if interrupted:
    return  # 整个函数跳过，sync 和 prefetch 都不执行
```

原因：中断的轮次不是"完成的对话"，写入会污染记忆。
但注意：如果用户主动中断一个**有价值的**回复，该轮记忆丢失。
后续 `/new` 触发的 `on_session_end` 会补写（因为它遍历完整 messages 列表）。

## on_session_end 的逐条跳过行为

`on_session_end` 的批量写入循环（`__init__.py` 第 643-646 行）：
```python
try:
    vec = _ollama_embed_single(combined, self.base_url, self.model)
except Exception:
    continue  # 静默跳过，不重试、不记录
```

单条 embedding 失败时直接 `continue`。不重试，不 log warning。
实际影响低（sync_turn 已兜底），但如果 sync_turn 和 on_session_end 都失败则该轮完全丢失。

## shutdown 中的锁释放

`shutdown_all()` 调用每个 provider 的 `shutdown()`。
memory-zvec 的 shutdown 做了：
```python
def shutdown(self) -> None:
    if self._coll is not None:
        try:
            del self._coll
            import gc; gc.collect()  # 释放 Zvec LOCK 文件
        except Exception:
            pass
    self._coll = None
```

Zvec Collection 无 `close()`/`release()` 方法，锁靠 Python GC 释放。
`del` + `gc.collect()` 是经过实验验证的唯一可靠释放方式。
实验确认：`del coll` 后如果没有 `gc.collect()`，锁不一定立即释放。

## Gateway 优雅关闭的记忆安全路径

`gateway/run.py` `_stop_impl()` 的记忆保障序列：

```
① _drain_active_agents(timeout)
   — 等待所有运行中 turn 完成（含 finalize_turn → sync_turn）
   — drain timeout 后中断未完成的 turn（标记 resume_pending）

② _finalize_shutdown_agents(active_agents)
   — 对每个 drain 期间活跃的 agent 调用 _cleanup_agent_resources
   — → shutdown_memory_provider(messages) → on_session_end + shutdown_all

③ 遍历 _agent_cache（idle cached agents）  ← 关键！源码 6740-6755 行
   for _entry in _idle_agents:
       _agent = _entry[0] if isinstance(_entry, tuple) else _entry
       self._cleanup_agent_resources(_agent)
   — 确保所有 idle agents 的 memory provider 也正确触发 on_session_end

④ 断开 adapters、清理资源、退出
```

**步骤 ③ 是本次源码审计的核心确认**：gateway 优雅关闭不会遗漏 idle cached agents。
但注意 idle cache 的 LRU cap / TTL sweep 驱逐（运行时，非关闭时）走
`_release_evicted_agent_soft` → `release_clients()`，**不调** `on_session_end`。
被驱逐 agent 的 memory provider 实例仍在内存中，等待下次正常关闭时统一清理。

## prefetch 机制

`queue_prefetch_all()` 在每轮完成后预热下一轮检索上下文。
用当前轮的 user_text 搜索，缓存结果 30s。
下一轮开始时 `prefetch()` 直接返回缓存，减少延迟。
