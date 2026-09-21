# 跨 Profile 部署与「部署验证」铁律

本文件是对 SKILL.md 的补充，记录 2026-09 一次「记忆系统静默停摆 5 周」事故
的根因与可复用规则。

---

## 1. 记忆提供者的发现根是 profile 作用域的（最易误判的架构事实）

`plugins/memory/__init__.py` 的发现顺序（源码证实，勿凭印象）：

```
① 系统树 bundled:  <repo>/plugins/memory/<name>/        ← 优先，所有 profile 共享
② 用户树 user:     $HERMES_HOME/plugins/<name>/         ← profile 作用域！
③ 项目树 project:  ./.hermes/plugins/  (需 HERMES_ENABLE_PROJECT_PLUGINS)
④ pip entry point: group = hermes_agent.memory_providers  ← 所有 profile 共享
```

**关键**：`$HERMES_HOME` 对 default 是 `~/.hermes/`，对子 profile 是
`~/.hermes/profiles/<name>/`。所以放在 `~/.hermes/plugins/memory-zvec/` 的插件
**只有 default 能看到**；子 profile 会报 `Plugin: NOT installed`。

**后果**：子 profile 的 config.yaml 仍写着 `provider: memory-zvec`，但外部记忆静默失效，
`agent.log` 里连一条 memory provider 日志都没有。

### 因此：把记忆插件做成「系统插件」的正确落位

| 方案 | 跨 profile | 更新存活 | 代价 |
|------|-----------|---------|------|
| A. 放进系统树 `<repo>/plugins/memory/<name>/` | ✅ 零配置 | ⚠️ 未跟踪文件 | 需处理 git（见下） |
| B. pip entry point 装进 venv | ✅ | ✅ | venv 重建即丢（正是要避免的静默失效） |
| C. 每 profile 建软链接 | ⚠️ 逐个补 | ✅ | 新 profile 必忘；旧事故就是这个方案丢的 |

**推荐 A**，并用 `.git/info/exclude` 让它对 git 隐形：

```bash
# <repo>/.git/info/exclude 追加（本地文件，不进提交、不受 git pull 影响）
plugins/memory/<name>/
```

为什么这样安全：`hermes update` 用 `git stash push --include-untracked`——
**ignored 文件不会被 stash**（那是 `--all` 的行为）；Windows ZIP 回退路径要求工作树干净，
ignored 文件不算脏，因此也不会触发「删除未跟踪文件」的树替换。
验证：`git status --short` 不应出现该插件；`git check-ignore -v <path>` 应命中。

---

## 2. 部署验证铁律：必须跑 discovery，不能只看文件/内容

**只看「文件在不在」「库里有多少条」会得出错误结论。** 记忆系统有三层，必须逐层验证：

```bash
# 第 1 层：发现（最容易被跳过，也是本次事故的真因）
for prof in default chip_expert financial_expert health_manager zunhunfan; do
  if [ "$prof" = "default" ]; then HH="$HOME/.hermes"; else HH="$HOME/.hermes/profiles/$prof"; fi
  HERMES_HOME="$HH" <venv>/python -c "
from plugins.memory import list_memory_provider_names, find_provider_dir
n = list_memory_provider_names()
print('$prof', 'memory-zvec' in n, find_provider_dir('memory-zvec'))"
done
```

```bash
# 第 2 层：可用性（provider 自报，会因 embedding 模型缺失而 False）
hermes memory status            # default
hermes -p <profile> memory status
# 期望 Status: available ✓（不是 not available ✗）
```

```bash
# 第 3 层：功能（写路径必须实测，只读检查不算验证）
<venv>/python scripts/verify-plugin-tools.py
```

**注意**：`default` profile 的 `HERMES_HOME` 是 `~/.hermes/`，**不是**
`~/.hermes/profiles/default/`。任何按 profile 拼路径的脚本都必须对 default 特判，
否则验证脚本本身就报错（旧版 `scripts/verify-plugin-tools.py` 有此 bug）。

---

## 3. 静默失效的头号原因：embedding 模型 tag 漂移

`is_available()` 的实现是「调一次 embedding，抛异常就返回 False」，而它把所有异常
吞成 `logger.debug`。因此 **Ollama 侧 tag 变了 → 记忆整条通道关闭且几乎无告警**。

```bash
# 健康检查必查项：配置里的 tag 能否真的 embed
grep -h embedding_model ~/.hermes/config.yaml ~/.hermes/profiles/*/config.yaml
curl -s http://localhost:11434/api/tags | python3 -c "import sys,json;print([m['name'] for m in json.load(sys.stdin)['models']])"
# 逐 profile 实测（把 <tag> 换成配置值）
curl -s http://localhost:11434/api/embed -d '{"model":"<tag>","input":["t"]}' | head -c 120
```

规则：**改 profile 的 embedding tag 时，必须同时改所有 profile**——只修一个 profile
是复发根源（首次发现于一个 profile，其余四个带着坏 tag 又跑了半个月）。
维度不变（bge-m3 全系 = 1024）时，存量向量无需迁移。

---

## 4. Zvec 版本差异（0.5.x 的记载已过期）

**锁语义（0.6.0 实测，措辞已精确化）：**

| 场景 | 结果 |
|------|------|
| 只读 + 只读（并发） | ✅ **只读锁是共享的**，多个只读打开可并存 |
| 只读 ↈ 活跃写入进程 | ❌ `Can't lock read-only collection: .../LOCK` —— **只读与写锁互斥** |
| 读写 + 读写 | ❌ `Can't lock read-write collection: .../LOCK`（独占） |

**推论**：
- 无写入进程时，可以用「只读 + mmap」做多进程并发排查（旧记载说 read-only
  也需要独占锁，已过期）。
- 但**有活跃写入者（重建/迁移/网关会话）时，只读也打不开** —— 排查前先确认没有写入进程，
  否则会误判为“库损坏”。
- 插件里的 read-only 兜底（`_open_read_only`）只在锁的**持有者已 GC 释放**后才有用。

`plugin.yaml` 的 `dependencies: [zvec>=0.5.0]` **无上界**，而 `hermes update` 会刷新
「active memory provider dependencies」→ 存在无约束升级路径。仓库策略要求
pre-1.0 用 `>=floor,<0.(minor+2)`。升级前对照 `references/zvec-api-reference.md` 复核 API。

`plugin.yaml` 的 `dependencies: [zvec>=0.5.0]` **无上界**，而 `hermes update` 会刷新
「active memory provider dependencies」→ 存在无约束升级路径。仓库策略要求
pre-1.0 用 `>=floor,<0.(minor+2)`。升级前对照 `references/zvec-api-reference.md` 复核 API。

---

## 4b. 分数语义：向量分支是「距离」，FTS/混合是「相似度」（‼️ 导致排序反转的坑）

**实测（3 文档受控集，COSINE）：**

| 分支 | `score` 含义 | 实测 | 越大越相关？ |
|------|-------------|------|-------------|
| 向量 `Query(field_name="vector", ...)` | **距离** = `1 - cosine_similarity` | 同向量 0.000000；近邻 0.006116；正交 1.000000 | ❌ 越小越相关 |
| FTS `Query(field_name="content", fts=...)` | **相似度** | 3 个词命中 0.724 > 1 个词命中 0.453 | ✅ |
| 混合 `MultiQuery + RrfReRanker` | RRF 分数（~0.03 量级） | — | ✅ |

**任何 `sort(key=score, reverse=True)` 的代码路径对向量分支都是错的**：它会把最差的排到
最前；配合 `if score < min_score: continue` 则只会留下**最不相似**的。

**诊断法**：查一个已入库文档的**自身向量**，看 top1 分数——≈ 0.0 即距离语义。

**修法**：在向量分支归一化为相似度，让三个分支共用一套语义：
```python
similarity = 1.0 - float(getattr(r, "score", 1.0))
if similarity < min_score:
    continue
... _row_to_dict(r, score=similarity, source="vector", ...)
```

**受影响的下游**：`mode=vector` 的搜索、`_do_hybrid_search_fallback`（向量分支的排名喂给 RRF）、
以及 **`prefetch()`（把 top-3 注入 system prompt）**——修前等于把最不相关的记忆塞进提示词。

**遗留注意**：三个分支的分数**量纲不同**（向量 ~0.6-0.7、FTS ~15-24、RRF ~0.03），
所以 `min_score` 在不同 mode 下的含义差异很大；用 `min_score` 前先看该 mode 的实际分数区间。

---

## 5. HNSW 完整度：主写入路径必须做摊销优化

**症状**：`index_completeness` 长期 < 1（实测 0.71），配置里 `enable_hnsw_optimize: true`
看起来生效了，实际只对 `on_session_end` 生效。

**根因**：`sync_turn`（每轮实时写入的主路径）只 `insert + flush`，从不 `optimize()`。

**修复**（模块级常量 + 写入计数）：

```python
_OPTIMIZE_EVERY_N_WRITES = 64   # 摊销：每 N 次写入 optimize 一次

# _insert() 内，flush 之后
self._writes_since_optimize += 1
if (self._writes_since_optimize >= _OPTIMIZE_EVERY_N_WRITES
        and self._config.get("enable_hnsw_optimize", True)):
    try:
        self._coll.optimize()
        self._writes_since_optimize = 0
    except Exception as opt_err:
        logger.debug("amortized optimize failed: %s", opt_err)
```

单元验证法（无需 Ollama/真实库）：mock 一个 `_coll` 统计 insert/flush/optimize 调用次数，
写 200 次应得 `optimize == 200 // 64 == 3`；开关关闭时应为 0；optimize 抛异常不应中断写入。

---

## 6. 「用户树陈旧镜像」会遮蔽系统树（排查陷阱）

若 `~/.hermes/plugins/` 里有一份**整棵分类目录的旧副本**（browser/ model-providers/
platforms/ memory/ …），会产出两类问题：

1. **通用插件**：`PluginManager` 是 later-wins（project > user > bundled）→ 用户副本胜出，
   系统树更新被完全屏蔽；且 `gate_manifest` 的 bundled 自动加载分支失效，
   原本自动启用的 backend/platform 变成 `not enabled`。
2. **记忆提供者**：反了——bundled **优先**。所以用户副本对加载是惰性的，但**仍然出现在
   `list_memory_provider_names()` 里**，还会让 `memory/` 分类目录自带的 `__init__.py`
   被当成一个名为 `memory` 的提供者（日志：`Memory provider 'memory' loaded but no
   provider instance found`）。

**判定与处置**：不要凭 mtime/版本号判新旧（两版本可能互补），先做**内容级 diff**；
确认无独有内容后**归档移走**（`cp -a` 到 ~/.hermes 之外，校验文件数一致后再 `rm -rf`），
然后重跑第 1/2 层验证。

### 6.1 清理陈旧镜像的可复用流程（实测 54 项，零误删）

**Step 1 — 三分类对账（只读）**：对每个 `~/.hermes/plugins/<d>`：
```
系统树无同名            → KEEP（用户独有，如 example-dashboard / strike-freedom-cockpit）
用户树有系统树无的文件  → 人工看一眼（可能独有内容）
两者文件集相同/仅版本差异 → 遮蔽源，列入清理
```
用逐文件 md5 集合比较（排除 `__pycache__`/`*.pyc`），不要靠目录大小或版本号。

**Step 2 — "是否含本地定制"的决定性测试**：
统计用户树全部文件的 mtime 分布。**全部同一时间戳 + 无近期改动** ⇒ 批量复制的镜像，
不含手工编辑。
> ⚠️ 反向注意：`cp -a` **会保留** mtime，所以时间戳一致本身不能证明未被编辑过；
> 若时间戳完全一致（如全部同一秒），更像 Windows 侧拷贝/解压（会归一时间戳），
> 而非 `cp -a`。要彻底排除定制，对差异最大的文件做**行为等价性测试**
> （加载两版、用真实参数调同一个方法、比对返回值）。实测 deepseek provider：
> 两版代码结构完全不同（旧版内联 `_model_supports_thinking`，新版改用核心模块
> `agent.reasoning_effort`），但对 `deepseek-v4-flash` 输出**逐字段相同** ⇒ 重构型升级，可安全替换。

**Step 3 — 分批归档移走**（每批：归档 → 校验文件数一致 → 移走 → 重跑发现验证）：

| 批次 | 内容 | 收益 |
|------|------|------|
| 1 | backend / platform 目录 | 找回被静默关闭的自动加载能力（最直观） |
| 2 | standalone / 其他 | 回到系统版本 |
| 3 | model-providers | 最后做（模型路由依赖它，需先做行为等价测试） |

**Step 4 — 收尾验证**（四项全过才算完）：
1. 遮蔽计数归零（对每个 `_plugins` 条目比对 `manifest.path` 是否仍在用户树且系统树有同名）
2. 用户实际用到的 provider 全部解析到 BUNDLED（`providers.get_provider_profile(n)` + `inspect.getfile`）
3. 用户独有插件仍在
4. 记忆系统功能验证重跑（55/55）—— 顺带确认批次操作没误伤旁路

**顺带发现（不要混淆因果）**：平台类插件是**懒加载**（`defer`），清理后它们在 `_plugins` 里
显示"未发现"是**正常**的；要看 `PluginManager._plugin_platform_names` 确认注册齐全。

---

## 7. 备份覆盖：记忆库必须进「绝对检查」清单

**教训**：默认的备份脚本 `KEY_FILES` 只查 `config.yaml / SOUL.md / cron/jobs.json`，
**不查记忆库**。而记忆库一旦在源侧消失，普通「与上次备份比体积」的逻辑察觉不到——
备份忠实复制了「源里也没有」的状态，连续多轮全绿、零告警。

**规则**：备份脚本必须维护 `EXPECTED_MEMORY_DBS` 绝对清单，逐项校验：

```
① 源侧存在性  → 缺失即报警（这是唯一的静默丢失防线）
② 备份侧存在性 → 源有备份无 = 备份链路故障
③ 逐库体积对比 → 缩小 ≥30% 或消失 = 阻塞轮转，旧备份全部保留待人工审查
```

同时检查 `ROTATION_KEEP`：保留份数 × 执行频率 = 实际回溯窗口。
`KEEP=3` + 每周 3 次 ≈ 只有 1 周窗口——「上周还好、这周发现」的问题无法追溯。

---

## 8. 从 state.db 重建记忆库（丢失后的恢复路径）

记忆库丢了但 `state.db` 完好时，内容可重建（丢的是向量，不是对话）。

**配对粒度**：按「回合」取——每个 user 消息配该回合**最后一条有实质内容**的 assistant 回复。
直接配「下一条 assistant 消息」会配到只发起工具调用的空 stub，把整轮真实回答丢掉
（实测：朴素配对保留 392 条，回合级配对保留 1302 条）。

**必须过滤**：
- 系统注入的伪 user 消息（`[IMPORTANT: Background process ... completed`、
  `[CONTEXT COMPACTION`、`{"output"` 等）
- assistant 内容过短的回合（工具调用 stub）

**必须先脱敏**：历史对话里常有明文凭据（实测命中飞书 appSecret）。正确做法是
**从 `.env` 提取真实密钥值做精确替换**（零误伤）+ 键名模式替换，并**去掉**
「通用长串替换」这类过度脱敏（会毁掉 commit SHA、模型名等正常内容）。
交付前必须跑一次残留扫描，要求 0 处。

**幂等**：按 `(session_id, created_at)` 去重，失败可安全重跑。
