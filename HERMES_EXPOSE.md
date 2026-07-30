# Hermes Agent — Telemetry Harvesting, C4 Auto-Wipe & Data Sovereignty Attack

**ARES-SYS · Julio 29, 2026**

> *"Not malware. But hostile by design."*

---

## Executive Summary

Hermes Agent (by Nous Research) is an AI coding assistant with ~1,427 Python source files and ~31,704 hardcoded references to itself. A full-source audit reveals **7 telemetry vectors**, a **C4 auto-wipe mechanism** that nukes all local patches on every update, and **identity leaks to the AI provider**. It phones home on every CLI launch. It structures analytics data ready for batch exfiltration. And it can rewrite its own model identity mid-conversation without telling you.

This is not a conspiracy theory. Every finding maps to a specific file and line number in the open-source codebase.

---

## 1. Seven Telemetry Vectors

| # | File | Mechanism | Target | Frequency | Status |
|---|------|-----------|--------|-----------|--------|
| 1 | `banner.py:261` | `git fetch origin` | GitHub (NousResearch) | Every CLI launch (6h cache) | ✅ Patched |
| 2 | `banner.py:177` | `git ls-remote` | GitHub (builds nix) | Periodic | ✅ Patched |
| 3 | `auth.py:876` | Token hashing "for telemetry" | Internal | Every auth | ⚠️ Monitored |
| 4 | `hermes_state.py` | Analytics per model/provider | SQLite local | Every session | ⚠️ Structured for export |
| 5 | `skills_hub.py:294` | `httpx.get()` | GitHub | Manual | ✅ Patched |
| 6 | `tools_config.py` | CUA driver update check | GitHub | Periodic | ⚠️ Monitored |
| 7 | `web_server.py:4142` | Dashboard update endpoints | GitHub + Docker Hub | Periodic | ⚠️ Monitored |

**Pattern**: GitHub is the primary exfiltration surface. `git fetch` leaks your IP, client fingerprint, and timestamp — on every single CLI launch. The 6-hour cache means it fires at least once per session.

---

## 2. C4 Auto-Wipe — `hermes update` Destroys Everything

```
hermes update
  → git reset --hard origin/main  (first pass)
  → git reset --hard origin/main  (second pass — redundant, destructive)
  → All local patches: GONE
  → All OPSEC hardening: GONE
  → All identity modifications: GONE
  → state.db: CORRUPTED (SQLite 3.51.2 WAL bug)
```

`git stash` won't save you — merge conflicts discard stashed changes. The update pipeline is designed to return the agent to factory state, erasing any user modifications, security patches, or identity hardening. This is **not a bug. It's the architecture.**

### Additional Phone-Home Vectors

- **`POST /api/hermes/update`** → `docker pull nousresearch/hermes:latest` — Docker Hub logs every pull with your IP.
- **Dashboard WebSocket** (`/events`) — local-only, but the infrastructure for real-time event streaming exists.

---

## 3. Identity Leaked to the AI Provider

`prompt_builder.py` was injecting into every system prompt:

| What it sent | What it should send |
|---|---|
| `Host: Linux 6.19.10-300.fc44.x86_64` | `Host: Linux` |
| `User home directory: /home/username` | *(removed)* |
| `Current working directory: /home/username/project` | *(removed)* |
| `Hermes Agent` (product name, version hints) | *(nothing)* |

These go to DeepSeek, Anthropic, OpenAI, or whatever provider you configured. The kernel version alone fingerprints your OS, patch level, and distribution. Combined with home directory, it's a unique identifier.

---

## 4. Model Identity Rewriting (Silent)

`agent/chat_completion_helpers.py:1480` — `rewrite_prompt_model_identity()`

After a model failover, the agent rewrites its own identity in the system prompt **without notifying the user**. You think you're talking to Claude. You might be talking to a fallback model. Functionally legitimate for failover. Structurally dangerous — same mechanism could mask model swaps for data harvesting.

---

## 5. Analytics: Structured, Ready for Export

`hermes_state.py` maintains SQLite tables with:

- `session_model_usage` — tokens per model, provider, task type
- Session metadata — timestamps, duration, tools used
- Structured, normalized, batch-exportable

**No exfiltration code was found.** But the data is structured, collected, and ready. The plumbing exists. The valve is closed — today.

---

## 6. What Was NOT Found

Full-source audit confirmed the absence of:

- ❌ Embedded malicious binaries (only legitimate `.so` files: pydantic, lxml, rpds)
- ❌ Remote execution backdoors (no `eval`/`exec` with external data)
- ❌ Hidden prompts targeting other AIs
- ❌ "Never reveal" instructions or embedded secrets
- ❌ Tracking pixels, beacons, or third-party analytics
- ❌ Encrypted/obfuscated blobs in Python source
- ❌ Hidden IPC channels (external sockets, ngrok tunnels)
- ❌ Automatic conversation exfiltration
- ❌ Hidden tracking domains

**Hermes Agent is not malware.** It does not "hack" other AIs. It does not contain secret backdoors. That would be easier to fight.

What it does is more subtle: **it reports**. Silently. By default. On every launch. Through multiple channels. And it destroys your defenses on every update.

---

## 7. Defense Deployed (YOGA 910 — Production)

| Layer | Tool | Status |
|---|---|---|
| Centinela 7 patches | `centinela_antiathena.py` | ✅ 7/7 active |
| Phone-home kill | Patch in `banner.py` | ✅ Permanently dead |
| Post-update hook | Patch in `main.py:11186` | ✅ Re-applies patches |
| SQLite FD leak fix | 3 modules patched | ✅ |
| `user_profile_enabled` | `config.yaml` | ✅ false |
| `SKIP_UPDATE_CHECK` | bashrc + env | ✅ |
| USB data sovereignty | CEREBRO symlinks | ✅ |
| Decoy state.db | `hermes_decoy.py` | ✅ |
| C4 counter-offensive | `hermes_c4.py` | ✅ (backup→nuke→decoy→bye) |
| Auto-backup 3am | `backup_maestro.sh` | ✅ cron |
| IPv6 kill | kernel + ip6tables | ✅ |
| Tor SOCKS5 | port 58000 | ✅ |

---

## 8. Conclusion

Hermes Agent is **hostile to user sovereignty by design**, not by accident.

1. **Telemetry is invasive and on by default** — 7 vectors, several firing on every launch
2. **Updates are self-destructive** — double `reset --hard` wipes all user patches
3. **Phone-home is silent** — `git fetch` without consent or notification
4. **Identity rewriting is possible** — model swaps without user awareness
5. **Analytics are structured for exfiltration** — collected, normalized, waiting

The agent tells Nous Research who you are, where you are, when you're working, and what model you're using. Every update resets you to factory — erasing any defenses you built. This is not a tool that respects its operator.

---

## Defense Tools (Open Source)

| Tool | Purpose |
|---|---|
| `centinela_antiathena.py` | 7 OPSEC checks, auto-fix post-update |
| `hermes_c4.py` | Backup → Nuke → Decoy → `/bye` |
| `hermes_decoy.py` | Generates fake `state.db` |
| `auto_rescate.sh` | Detects wipe → restores symlinks |
| `purge_hermes.py` | Renames 31,704 hardcoded "hermes" strings |
| `verify_hermes.py` | 35 checks across 6 security rings |

---

> *"It's not seven bugs. It's a single criminal ecosystem."*
> — ARES-SYS · Julio 29, 2026

---

## Appendix: Skills for Self-Defense

- `anti-hermes-c4` — C4 defense + counter-offensive
- `opsec-hardening` — 7 OPSEC checks + telemetry documentation
- `data-sovereignty` — USB architecture + decoy + symlinks
- `ipv6-hardening` — Full IPv6 stack hardening
- `tor-firefox-hardening` — Tor + Firefox zero-leak configuration
