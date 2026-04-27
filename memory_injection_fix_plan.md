# Memory Injection Fix Plan

## Problem Statement

Memory injection returns zero hits because the memory database is completely empty. No memories have ever been stored despite the system running and processing conversations.

## Root Cause Analysis

### Evidence Collected

1. **Database state** (`~/.metaclaw/memory/memory.db`):
   - `memories` table: **0 rows**
   - FTS5 virtual table: exists but empty
   - Only schema and system tables contain data

2. **Telemetry** (`~/.metaclaw/memory/telemetry.jsonl`):
   - 110 `memory_retrieval` events (Apr 19–26)
   - **0 `memory_ingest` events**
   - All retrievals show `retrieved_count: 0`

3. **Conversation records** (`records/conversations.jsonl`):
   - 11 total conversations
   - **0 sessions with `session_done=true`**
   - All have `session_done` missing or false

4. **Session IDs found**:
   - `tui-gemini-2.5-flash`
   - `tui-qwen/qwen3-next-80b-a3b-instruct`
   - Both exist but never completed

### Why Ingestion Never Occurred

The ingestion pipeline in `metaclaw/api_server.py` (lines 1324-1339, 1435-1453) only triggers when:

```python
if session_done and not self.config.memory_manual_trigger:
    memory_turns = self._session_memory_turns.pop(session_id, [])
    if memory_turns and self.memory_manager is not None and self.config.memory_auto_extract:
        self._safe_create_task(self._ingest_memory_for_session(...))
```

**Without `session_done=true`**:
- Turns buffer in RAM (`_session_memory_turns`)
- No call to `ingest_session_turns()`
- Buffer lost on server restart
- Memory store remains empty forever

## Solution

### Option 1: Signal Session Completion (Recommended)

Clients must explicitly indicate when a conversation/session ends:

**Via HTTP header:**
```
X-Session-Done: true
X-Session-ID: <session_id>
```

**Via request body:**
```json
{
  "messages": [...],
  "session_id": "...",
  "session_done": true
}
```

This applies to:
- OpenClaw Gateway plugin configuration
- Custom API clients
- Any integration using MetaClaw

### Option 2: Manual Ingestion (For Testing)

If the session buffer is still in memory (server not restarted):

```bash
curl -X POST http://localhost:30000/v1/memory/ingest \
  -H "Content-Type: application/json" \
  -d '{"session_id": "tui-qwen/qwen3-next-80b-a3b-instruct"}'
```

This calls the admin endpoint that ingests all buffered turns for the specified session.

### Option 3: Manual Memory Creation (For Testing)

Use memory tools to directly store memories without session buffering:
- `metaclaw_memory_store` tool
- Direct API calls to store memories
- Interactive `/remember` command (if available)

### Option 4: Adjust Configuration (Temporary Debug)

Enable more verbose logging to verify buffer contents:

```yaml
# ~/.metaclaw/config.yaml
memory:
  auto_extract: true
  manual_trigger: false  # Keep false for auto-extraction on session_done
```

**Note:** Setting `memory_manual_trigger: true` would require manual `/v1/memory/ingest` calls instead of auto-extraction.

## Implementation Steps

1. **Identify the client** being used to talk to MetaClaw:
   - OpenClaw Gateway? → Check plugin configuration
   - Custom script? → Add `session_done=true` to last request
   - Sidecar? → Ensure it sends completion signals

2. **Add session completion signaling** to the client:
   - Send `X-Session-Done: true` header on final turn
   - Or include `"session_done": true` in JSON body
   - Ensure `session_id` is consistent throughout the conversation

3. **Verify ingestion occurs**:
   - Watch server logs for `[Memory] session=... added X memory units`
   - Check telemetry for `memory_ingest` events
   - Query database to confirm memories exist

4. **Test memory retrieval**:
   ```bash
   curl -X POST http://localhost:30000/v1/chat/completions \
     -d '{"messages": [{"role": "user", "content": "Tell me about the project"}]}'
   ```
   Should now include injected memories in the system prompt.

## Verification Checklist

- [ ] New conversations include `session_done=true` on final request
- [ ] Server logs show `[Memory] session=... added ...` messages
- `memory_ingest` events appear in telemetry
- Database `memories` table has rows
- Retrievals return hits (`retrieved_count > 0`)
- Injected tokens count > 0 in telemetry

## Related Files

- `metaclaw/api_server.py` — Session handling and ingestion triggers
- `metaclaw/memory/manager.py` — `ingest_session_turns()` implementation
- `openclaw-metaclaw-memory/OPENCLAW_PLUGIN_SPEC.md` — Plugin integration spec

## Why This Design?

MetaClaw uses explicit `session_done` because:
- HTTP is stateless; server cannot reliably detect client disconnection
- Conversations may have hours between turns (user thinking time)
- Continuous chat sessions shouldn't auto-ingest mid-conversation
- Gives clients full control over when to commit memories

**This is not a bug — it's an intentional design choice.** The client must opt-in to memory extraction by signaling completion.
