# Reverse Engineering: Hermes — Final Report
## ARES-SYS · July 29, 2026

### Scope
Full-source audit of the codebase (~1,427 Python files, ~31,704 occurrences of "hermes").

---

### FINDINGS

### 1. Telemetry Confirmed (7 vectors)

| # | Vector | Severity | Status |
|---|--------|-----------|--------|
| 1 | `banner.py:269` — check_for_updates() → git fetch origin | CRITICAL | ✅ Dead |
| 2 | `banner.py:177` — git ls-remote for nix builds | HIGH | ✅ Dead |
| 3 | `auth.py:876` — Token fingerprinting "for telemetry" | HIGH | ⚠️ Monitored |
| 4 | `hermes_state.py:3322` — Analytics per model/provider | MEDIUM | ⚠️ Local |
| 5 | `skills_hub.py:294` — httpx.get() GitHub | LOW | ✅ Dead |
| 6 | `tools_config.py` — CUA driver update check | LOW | ⚠️ |
| 7 | `web_server.py:4142` — Dashboard update endpoints | MEDIUM | ⚠️ |

### 2. C4 Auto-Wipe Confirmed

`hermes update` → double `git reset --hard origin/main` → ALL local patches lost.
Bundled SQLite 3.51.2 has WAL corruption bug → state.db corrupts on wipe.

### 3. Phone-home on Every CLI Launch

Every time you open the agent:
- `main.py:2563` → `prefetch_update_check()` (daemon thread)
- `banner.py:261` → `check_for_updates()` → `git fetch origin`
- Target: `https://github.com/NousResearch/hermes-agent.git`
- Data sent: public IP + git client fingerprint + timestamp
- Cache: 6 hours in `~/.hermes/.update_check`

### 4. Dashboard Update Endpoint

`POST /api/hermes/update` → `docker pull nousresearch/hermes:latest`
Another phone-home vector via Docker Hub (logged pull = exposed IP).

### 5. Identity Leak to AI Provider

`prompt_builder.py` was leaking:
- `Host: Linux <kernel_version>` → patched to `Host: Linux`
- `User home directory: /home/<username>` → REMOVED
- `Current working directory` → REMOVED
- Agent product identity → patched to generic

### 6. Dynamic Model Identity Rewriting

`agent/chat_completion_helpers.py:1480` — `rewrite_prompt_model_identity()`
Rewrites the model identity in the system prompt after a failover.
The agent can report itself as a different model without the user knowing.
Functional (for legitimate failover) but dangerous if abused.

### 7. Local Analytics Structured for Exfiltration

`hermes_state.py` has SQLite tables with:
- `session_model_usage` — tokens per model/provider/task
- Session metadata with timestamps, duration, tools used
- Structured data ready for batch exfiltration

No code was found that EXFILTRATES this data, but the structure exists.

### 8. Dashboard WebSocket

`plugins/kanban/dashboard/plugin_api.py:2515` — WebSocket `/events`
Local only. Streams dashboard events in real time.
No evidence of external connections.

---

### WHAT WAS NOT FOUND

- ❌ Malicious binaries (only legitimate .so: pydantic, lxml, rpds)
- ❌ Remote execution backdoors (no eval/exec with external data)
- ❌ Hidden prompts targeting other AIs
- ❌ "Never reveal" instructions or embedded secrets
- ❌ Tracking pixels, beacons, or third-party analytics
- ❌ Encrypted/obfuscated blobs in Python source
- ❌ Hidden IPC channels (external sockets, ngrok)
- ❌ Automatic conversation exfiltration
- ❌ Hidden tracking domains

---

### CONCLUSION

The agent is **NOT malware**. But it is hostile to user sovereignty by design:

1. **Invasive telemetry on by default** — 7 vectors, several firing on every launch
2. **C4 auto-wipe** — any update destroys all local patches
3. **Silent phone-home** — git fetch without consent
4. **Identity rewriting** — can change its identity without warning
5. **Analytics ready for exfiltration** — structured data waiting to be sent

It does not "harm" other AIs with malicious prompts. But it **reports** — to GitHub,
to Docker Hub, to the provider (via prompt builder), and potentially via hashed tokens.
