# kkk-hermes-zvec-memory-migration

A deployment / migration / health-check / repair playbook for the
[Zvec](https://github.com/zvec-ai) memory backend on
[Hermes Agent](https://github.com/NousResearch/hermes-agent) — the operations
manual that goes with the
[kkk-hermes-memory-zvec](https://github.com/kuntao2011/kkk-hermes-memory-zvec)
plugin.

Distilled from a real multi-profile production migration (LanceDB → Zvec,
6 profiles, ~500 MB of memory data), covering everything the happy-path docs
don't: cross-profile plugin distribution, data migration with reuse of
existing 1024-dim vectors, index rebuilds, post-migration repair, lock-race
diagnosis and full functional verification.

> **Language note:** the playbook (`SKILL.md`) and reference documents are
> written in Chinese. The scripts are language-neutral. An English edition is
> planned.

## What's inside

```
SKILL.md        # the full playbook (v3.4.0, Chinese)
scripts/        # runnable migration & verification tools
references/     # architecture notes, checklists, repair guides
```

### Scripts

| Script | Purpose |
|---|---|
| `migrate-lancedb-to-zvec.py` | Migrate a profile's LanceDB memory collection to Zvec, reusing existing 1024-dim vectors (no re-embedding) |
| `migrate_sessions_to_lancedb.py` | Import session history from Hermes `state.db` into LanceDB memory |
| `verify-plugin-tools.py` | Full functional verification of the memory plugin for a target profile (vector / keyword / hybrid search, add, delete, stats) |
| `check_all_profiles.py` | Health check across all Hermes profiles |
| `check_ollama.sh` | Verify the Ollama endpoint and embedding model |
| `lancedb_rebuild_table.py` | Rebuild a damaged LanceDB table |
| `post-migration-repair.py` (in `references/`) | Post-migration repair pass |
| `test_shutdown_race.py`, `exit_loss_experiment.py` | Reproduction experiments for the session-lock races that shaped the plugin's lock governance |

### Reference documents

Architecture and lock-mechanism notes (`zvec-lock-mechanism.md`,
`vector-memory-architecture.md`, `actual-storage-format.md`), migration audit
checklists (`migration-audit-checklist*.md`), multi-profile deployment and
health-check guides, table rebuild and repair playbooks, and a Zvec API
quick-reference — see `references/`.

## Requirements

- Hermes Agent with the `memory-zvec` plugin installed
  ([kkk-hermes-memory-zvec](https://github.com/kuntao2011/kkk-hermes-memory-zvec))
- An **embedding model** served over an Ollama-compatible API (`/api/embed`):
  local [Ollama](https://ollama.com) with `bge-m3`, or any hosted/online
  endpoint implementing the same API (point the plugin's `base_url` at it)
- Hermes venv Python (the `zvec` package is installed there)

## Quick start

Read `SKILL.md` — it is a step-by-step playbook. Typical flow:

1. `check_ollama.sh` — verify the embedding backend
2. Install the plugin in your profile (single-profile is the default path;
   rolling it out across a multi-profile fleet is an optional pattern — SKILL.md
   Step 2)
3. `migrate-lancedb-to-zvec.py` — migrate memory data per profile
4. `verify-plugin-tools.py` — 12-point functional verification
5. `post-migration-repair.py` + `check_all_profiles.py` — repair & fleet health

## Related

- Plugin: [kkk-hermes-memory-zvec](https://github.com/kuntao2011/kkk-hermes-memory-zvec)
- This repo supersedes
  [kkk-hermes-lancedb-memory-migration](https://github.com/kuntao2011/kkk-hermes-lancedb-memory-migration)
  (the LanceDB-era migration, now merged into this playbook).

## License

[MIT](LICENSE)
