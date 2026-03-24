# Session Digest — Design Spec

## Summary

Auto-generate a summary of what the agent accomplished when a session ends. Store digests persistently, post rich notifications to the Notifications topic, and expose via `/digest` command + API endpoint. Gives users and teams visibility into agent work without reading full conversation logs.

## Problem

When a session completes, OpenACP posts a generic "Session X completed" notification. Users running multiple sessions lose track of what each agent did. Teams have no visibility into agent work. There's no way to review past sessions without scrolling through Telegram topics.

## Requirements

- **Auto-digest on completion**: When a session ends (status → `finished`), ask the agent to summarize what was accomplished before disconnecting
- **Persistent storage**: Store digests in `~/.openacp/digests.json` with session metadata
- **Rich notifications**: Replace the generic "Session completed" with the actual digest summary in the Notifications topic
- **`/digest` command**: View recent session digests in Telegram (paginated, filterable by agent)
- **API endpoint**: `GET /api/digests` for the UI dashboard
- **Graceful degradation**: If agent fails to summarize (timeout, error), fall back to generic message — never block session cleanup
- **Configurable**: Can disable auto-digest in config; adjustable retention period

## Non-Goals

- Full conversation transcript export (future feature)
- Digest generation for cancelled/error sessions (only `finished`)
- AI-powered search across digests (future)
- Daily/weekly email or scheduled digests (future enhancement)

## Design

### Architecture

```
Session finishes
  → SessionBridge.onSessionEnd()
  → Before disconnecting, inject digest prompt into agent (like autoName pattern)
  → Capture agent's summary response
  → Store in DigestStore
  → Post rich notification with summary
  → Disconnect bridge normally
```

### Digest Generation (Session.ts)

Follows the same pattern as `autoName()`:
1. Pause session event emitter (prevent digest output from reaching adapter)
2. Send summary prompt to agent: *"Summarize what you accomplished in this session in 2-3 sentences. Include: key files changed, decisions made, and current status. Reply ONLY with the summary."*
3. Capture text response (max 500 chars)
4. Resume and continue normal cleanup
5. Timeout after 10 seconds — fall back to generic summary

```typescript
// On Session class
async generateDigest(): Promise<string> {
  // Similar to autoName() — pause, prompt, capture, resume
}
```

### DigestStore

Simple JSON file persistence, similar to UsageStore/SessionStore patterns:

```typescript
interface DigestRecord {
  id: string;
  sessionId: string;
  sessionName: string;
  agentName: string;
  workingDir: string;
  summary: string;
  startedAt: string;
  completedAt: string;
  // Quick stats from the session
  promptCount: number;
  durationMinutes: number;
}

interface DigestStoreFile {
  version: 1;
  digests: DigestRecord[];
}
```

- File: `~/.openacp/digests.json`
- Retention: configurable (default 90 days), cleanup on startup + daily interval
- Debounced writes (2s), flush on shutdown

### Rich Notifications

Current notification on session_end:
```
✅ Session "Fix login bug" completed
```

With digest:
```
✅ Fix login bug — completed

Fixed the authentication bypass in auth.ts by adding proper token
validation. Updated 3 test cases. Login flow now correctly rejects
expired tokens.

⏱ 12 min · 🔤 45k tokens · 📁 ~/myproject
```

### Telegram `/digest` Command

```
/digest         → show last 5 digests
/digest 10      → show last 10
/digest claude  → filter by agent name
```

Output:
```
📋 Recent Session Digests

── Fix login bug (claude) · 12 min ago ──
Fixed auth bypass in auth.ts, added token validation, updated 3 tests.

── Add dark mode (claude) · 2h ago ──
Implemented dark mode toggle in settings, added CSS variables for
theming, updated 8 components.

── Refactor API (codex) · yesterday ──
Extracted route handlers into separate files, added input validation
middleware.
```

### API Endpoint

```
GET /api/digests
  ?limit=10         (default 20)
  ?agent=claude     (filter by agent)
  ?since=2026-03-20 (filter by date)

Response: { ok: true, digests: DigestRecord[] }
```

### Config

```typescript
// In ConfigSchema, after usage:
const DigestSchema = z.object({
  enabled: z.boolean().default(true),
  retentionDays: z.number().default(90),
}).default({});
```

### Error Handling

| Scenario | Behavior |
|----------|----------|
| Agent fails to summarize | Fall back to generic "Session completed" — log warning |
| Agent times out (>10s) | Abort prompt, use generic summary |
| Session ended by error/cancel | Skip digest (only for `finished` status) |
| DigestStore write fails | Log error, continue — don't block session cleanup |
| Corrupt digests.json | Backup to `.bak`, start fresh (same as UsageStore) |

### Affected Components

**Core layer** (new):
- `src/core/digest-store.ts` — `DigestStore` class (append, query, cleanup)
- `src/core/session.ts` — add `generateDigest()` method (like autoName)

**Core layer** (modify):
- `src/core/session-bridge.ts` — call `generateDigest()` before disconnect on session_end
- `src/core/core.ts` — create DigestStore in constructor, destroy in stop(), expose property
- `src/core/config.ts` — add `DigestSchema` to ConfigSchema
- `src/core/types.ts` — add `DigestRecord` interface
- `src/core/index.ts` — export DigestStore
- `src/core/api/routes/sessions.ts` — add `GET /api/digests` endpoint

**Adapter layer** (modify):
- `src/adapters/telegram/commands/session.ts` — add `handleDigest()` command
- `src/adapters/telegram/commands/index.ts` — register `/digest` command + STATIC_COMMANDS
- `src/adapters/telegram/formatting.ts` — add `formatDigestList()` and `formatDigestNotification()`

**No changes needed**:
- `notification.ts` — existing NotificationMessage.summary field is sufficient for rich content
- `session-store.ts` — digests are separate from session records
