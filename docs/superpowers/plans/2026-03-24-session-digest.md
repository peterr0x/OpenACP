# Session Digest Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Auto-generate session summaries on completion, persist them, post rich notifications, and expose via `/digest` command + API.

**Architecture:** `Session.generateDigest()` follows the `autoName()` pattern — pause emitter, inject summary prompt, capture response. `DigestStore` persists records to `~/.openacp/digests.json`. `SessionBridge` calls digest before disconnect. Telegram adapter adds `/digest` command.

**Tech Stack:** TypeScript, vitest

**Spec:** `docs/superpowers/specs/2026-03-24-session-digest-design.md`

---

## File Structure

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `src/core/digest-store.ts` | `DigestStore` class — append, query, cleanup, flush |
| Create | `src/__tests__/digest-store.test.ts` | Unit tests for DigestStore |
| Create | `src/__tests__/session-digest.test.ts` | Tests for generateDigest + SessionBridge integration |
| Create | `src/__tests__/digest-command.test.ts` | Tests for /digest formatting |
| Modify | `src/core/types.ts` | Add `DigestRecord` interface |
| Modify | `src/core/config.ts` | Add `DigestSchema` to ConfigSchema + DEFAULT_CONFIG |
| Modify | `src/core/session.ts` | Add `generateDigest()` method |
| Modify | `src/core/session-bridge.ts` | Call `generateDigest()` on session_end, post rich notification |
| Modify | `src/core/core.ts` | Create DigestStore in constructor, destroy in stop() |
| Modify | `src/core/index.ts` | Export DigestStore |
| Modify | `src/core/api/routes/sessions.ts` | Add `GET /api/digests` endpoint |
| Modify | `src/adapters/telegram/formatting.ts` | Add `formatDigestList()` and `formatDigestNotification()` |
| Modify | `src/adapters/telegram/commands/session.ts` | Add `handleDigest()` |
| Modify | `src/adapters/telegram/commands/index.ts` | Register `/digest` command |

---

### Task 1: DigestRecord type and DigestSchema config

**Files:**
- Modify: `src/core/types.ts`
- Modify: `src/core/config.ts`

- [ ] **Step 1: Add DigestRecord to types.ts**

Append at the end of `src/core/types.ts` (after `DiscordPlatformData`):

```typescript
export interface DigestRecord {
  id: string;
  sessionId: string;
  sessionName: string;
  agentName: string;
  workingDir: string;
  summary: string;
  startedAt: string;
  completedAt: string;
  promptCount: number;
  durationMinutes: number;
}
```

- [ ] **Step 2: Add DigestSchema to config.ts**

In `src/core/config.ts`, add after the `UsageSchema` block (and its `export type UsageConfig` line):

```typescript
const DigestSchema = z
  .object({
    enabled: z.boolean().default(true),
    retentionDays: z.number().default(90),
  })
  .default({});

export type DigestConfig = z.infer<typeof DigestSchema>;
```

Then add `digest: DigestSchema,` to `ConfigSchema` (after the `usage` field).

Also add `digest: {},` to `DEFAULT_CONFIG`.

- [ ] **Step 3: Verify build**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 4: Commit**

```bash
git add src/core/types.ts src/core/config.ts
git commit -m "feat(types): add DigestRecord type and DigestSchema config"
```

---

### Task 2: DigestStore

**Files:**
- Create: `src/core/digest-store.ts`
- Create: `src/__tests__/digest-store.test.ts`

- [ ] **Step 1: Write failing tests**

Create `src/__tests__/digest-store.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import fs from "node:fs";
import path from "node:path";
import os from "node:os";
import { DigestStore } from "../core/digest-store.js";
import type { DigestRecord } from "../core/types.js";

function makeDigest(overrides: Partial<DigestRecord> = {}): DigestRecord {
  return {
    id: "d-1",
    sessionId: "sess-1",
    sessionName: "Fix login bug",
    agentName: "claude",
    workingDir: "/tmp/project",
    summary: "Fixed auth bypass in auth.ts, added token validation.",
    startedAt: new Date().toISOString(),
    completedAt: new Date().toISOString(),
    promptCount: 5,
    durationMinutes: 12,
    ...overrides,
  };
}

describe("DigestStore", () => {
  let tmpDir: string;
  let filePath: string;
  let store: DigestStore;

  beforeEach(() => {
    tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), "openacp-digest-test-"));
    filePath = path.join(tmpDir, "digests.json");
    store = new DigestStore(filePath, 90);
  });

  afterEach(() => {
    store.destroy();
    fs.rmSync(tmpDir, { recursive: true, force: true });
  });

  it("appends and queries digests", () => {
    store.append(makeDigest());
    const all = store.query();
    expect(all).toHaveLength(1);
    expect(all[0].summary).toContain("auth bypass");
  });

  it("queries with limit", () => {
    store.append(makeDigest({ id: "d-1" }));
    store.append(makeDigest({ id: "d-2" }));
    store.append(makeDigest({ id: "d-3" }));
    expect(store.query({ limit: 2 })).toHaveLength(2);
  });

  it("queries filtered by agent", () => {
    store.append(makeDigest({ id: "d-1", agentName: "claude" }));
    store.append(makeDigest({ id: "d-2", agentName: "codex" }));
    const results = store.query({ agent: "claude" });
    expect(results).toHaveLength(1);
    expect(results[0].agentName).toBe("claude");
  });

  it("queries filtered by since date", () => {
    const old = new Date();
    old.setDate(old.getDate() - 10);
    const recent = new Date();
    store.append(makeDigest({ id: "d-1", completedAt: old.toISOString() }));
    store.append(makeDigest({ id: "d-2", completedAt: recent.toISOString() }));

    const since = new Date();
    since.setDate(since.getDate() - 5);
    const results = store.query({ since: since.toISOString() });
    expect(results).toHaveLength(1);
    expect(results[0].id).toBe("d-2");
  });

  it("returns digests in reverse chronological order", () => {
    const t1 = new Date("2026-03-20T10:00:00Z");
    const t2 = new Date("2026-03-21T10:00:00Z");
    store.append(makeDigest({ id: "d-1", completedAt: t1.toISOString() }));
    store.append(makeDigest({ id: "d-2", completedAt: t2.toISOString() }));
    const results = store.query();
    expect(results[0].id).toBe("d-2"); // newer first
  });

  it("flushes to disk and loads back", () => {
    store.append(makeDigest());
    store.flushSync();
    store.destroy();

    const store2 = new DigestStore(filePath, 90);
    expect(store2.query()).toHaveLength(1);
    store2.destroy();
  });

  it("cleans up old digests", () => {
    const old = new Date();
    old.setDate(old.getDate() - 100);
    store.append(makeDigest({ id: "old", completedAt: old.toISOString() }));
    store.append(makeDigest({ id: "new" }));
    store.cleanup();
    expect(store.query()).toHaveLength(1);
    expect(store.query()[0].id).toBe("new");
  });

  it("handles corrupt file gracefully", () => {
    fs.writeFileSync(filePath, "NOT JSON{{{");
    const store2 = new DigestStore(filePath, 90);
    expect(store2.query()).toHaveLength(0);
    expect(fs.existsSync(filePath + ".bak")).toBe(true);
    store2.destroy();
  });

  it("getBySessionId returns digest for specific session", () => {
    store.append(makeDigest({ id: "d-1", sessionId: "sess-1" }));
    store.append(makeDigest({ id: "d-2", sessionId: "sess-2" }));
    const result = store.getBySessionId("sess-1");
    expect(result?.sessionId).toBe("sess-1");
  });

  it("destroy flushes pending data", () => {
    store.append(makeDigest());
    store.destroy();
    const raw = JSON.parse(fs.readFileSync(filePath, "utf-8"));
    expect(raw.digests).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm test src/__tests__/digest-store.test.ts`

- [ ] **Step 3: Implement DigestStore**

Create `src/core/digest-store.ts`:

```typescript
import fs from "node:fs";
import path from "node:path";
import type { DigestRecord } from "./types.js";
import { createChildLogger } from "./log.js";

const log = createChildLogger({ module: "digest-store" });

interface StoreFile {
  version: number;
  digests: DigestRecord[];
}

const DEBOUNCE_MS = 2000;

export class DigestStore {
  private digests: DigestRecord[] = [];
  private debounceTimer: ReturnType<typeof setTimeout> | null = null;
  private cleanupInterval: ReturnType<typeof setInterval> | null = null;
  private flushHandler: (() => void) | null = null;

  constructor(
    private filePath: string,
    private retentionDays: number,
  ) {
    this.load();
    this.cleanup();

    this.cleanupInterval = setInterval(
      () => this.cleanup(),
      24 * 60 * 60 * 1000,
    );

    this.flushHandler = () => {
      try {
        this.flushSync();
      } catch {
        // Best effort — don't block other exit handlers
      }
    };
    process.on("SIGTERM", this.flushHandler);
    process.on("SIGINT", this.flushHandler);
    process.on("exit", this.flushHandler);
  }

  append(record: DigestRecord): void {
    this.digests.push(record);
    this.scheduleDiskWrite();
  }

  query(opts?: { limit?: number; agent?: string; since?: string }): DigestRecord[] {
    let results = [...this.digests];

    if (opts?.agent) {
      results = results.filter((d) => d.agentName === opts.agent);
    }
    if (opts?.since) {
      const sinceTime = new Date(opts.since).getTime();
      results = results.filter((d) => new Date(d.completedAt).getTime() >= sinceTime);
    }

    // Reverse chronological (newest first)
    results.sort((a, b) => new Date(b.completedAt).getTime() - new Date(a.completedAt).getTime());

    if (opts?.limit) {
      results = results.slice(0, opts.limit);
    }

    return results;
  }

  getBySessionId(sessionId: string): DigestRecord | undefined {
    return this.digests.find((d) => d.sessionId === sessionId);
  }

  cleanup(): void {
    const cutoff = Date.now() - this.retentionDays * 24 * 60 * 60 * 1000;
    const before = this.digests.length;
    this.digests = this.digests.filter(
      (d) => new Date(d.completedAt).getTime() >= cutoff,
    );
    const removed = before - this.digests.length;
    if (removed > 0) {
      log.info({ removed }, "Cleaned up expired digest records");
      this.scheduleDiskWrite();
    }
  }

  flushSync(): void {
    if (this.debounceTimer) {
      clearTimeout(this.debounceTimer);
      this.debounceTimer = null;
    }
    const data: StoreFile = { version: 1, digests: this.digests };
    const dir = path.dirname(this.filePath);
    if (!fs.existsSync(dir)) fs.mkdirSync(dir, { recursive: true });
    fs.writeFileSync(this.filePath, JSON.stringify(data, null, 2));
  }

  destroy(): void {
    if (this.debounceTimer) this.flushSync();
    if (this.cleanupInterval) clearInterval(this.cleanupInterval);
    if (this.flushHandler) {
      process.removeListener("SIGTERM", this.flushHandler);
      process.removeListener("SIGINT", this.flushHandler);
      process.removeListener("exit", this.flushHandler);
      this.flushHandler = null;
    }
  }

  private load(): void {
    if (!fs.existsSync(this.filePath)) return;
    try {
      const raw = JSON.parse(fs.readFileSync(this.filePath, "utf-8")) as StoreFile;
      if (raw.version !== 1) {
        log.warn({ version: raw.version }, "Unknown digest store version, skipping load");
        return;
      }
      this.digests = raw.digests || [];
      log.info({ count: this.digests.length }, "Loaded digest records");
    } catch (err) {
      log.error({ err }, "Failed to load digest store, backing up corrupt file");
      try {
        fs.copyFileSync(this.filePath, this.filePath + ".bak");
      } catch { /* best effort */ }
      this.digests = [];
    }
  }

  private scheduleDiskWrite(): void {
    if (this.debounceTimer) clearTimeout(this.debounceTimer);
    this.debounceTimer = setTimeout(() => {
      this.flushSync();
    }, DEBOUNCE_MS);
  }
}
```

- [ ] **Step 4: Run tests**

Run: `pnpm test src/__tests__/digest-store.test.ts`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add src/core/digest-store.ts src/__tests__/digest-store.test.ts
git commit -m "feat(core): add DigestStore with JSON persistence, query, and cleanup"
```

---

### Task 3: Session.generateDigest()

**Files:**
- Modify: `src/core/session.ts`
- Create: `src/__tests__/session-digest.test.ts`

- [ ] **Step 1: Write failing tests**

Create `src/__tests__/session-digest.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { Session } from "../core/session.js";

function createMockSession(promptResponse: string, shouldTimeout = false) {
  const session = new Session({
    channelId: "telegram",
    agentName: "claude",
    agentInstance: {
      prompt: vi.fn(async () => {
        if (shouldTimeout) await new Promise(() => {}); // never resolves
      }),
      on: vi.fn(),
      off: vi.fn(),
      onSessionUpdate: null,
      onPermissionRequest: null,
      sessionId: "agent-1",
    } as any,
  });
  session.activate();

  // Mock the agent emitting text during prompt
  const originalPrompt = session.agentInstance.prompt;
  session.agentInstance.prompt = vi.fn(async (text: string) => {
    // Simulate agent response by emitting event
    if (!shouldTimeout) {
      const handlers = (session.agentInstance as any)._textHandlers || [];
      // The digest captures via agent_event listener
      // We simulate by making the prompt resolve after "capturing"
    }
    return originalPrompt.call(session.agentInstance, text);
  });

  return session;
}

describe("Session.generateDigest", () => {
  it("returns a digest string from agent", async () => {
    const session = new Session({
      channelId: "telegram",
      agentName: "claude",
      agentInstance: {
        prompt: vi.fn(async () => {}),
        on: vi.fn((event: string, handler: Function) => {
          if (event === "agent_event") {
            // Simulate agent responding with text
            setTimeout(() => handler({ type: "text", content: "Fixed auth bug in login.ts" }), 0);
          }
        }),
        off: vi.fn(),
        onSessionUpdate: null,
        onPermissionRequest: null,
        sessionId: "agent-1",
      } as any,
    });
    session.activate();

    const digest = await session.generateDigest();
    expect(digest).toContain("Fixed auth bug");
  });

  it("returns fallback on timeout", async () => {
    const session = new Session({
      channelId: "telegram",
      agentName: "claude",
      agentInstance: {
        prompt: vi.fn(() => new Promise(() => {})), // never resolves
        on: vi.fn(),
        off: vi.fn(),
        onSessionUpdate: null,
        onPermissionRequest: null,
        sessionId: "agent-1",
      } as any,
    });
    session.activate();

    const digest = await session.generateDigest(100); // 100ms timeout
    expect(digest).toBe("");
  });

  it("returns empty string on error", async () => {
    const session = new Session({
      channelId: "telegram",
      agentName: "claude",
      agentInstance: {
        prompt: vi.fn(() => Promise.reject(new Error("agent crashed"))),
        on: vi.fn(),
        off: vi.fn(),
        onSessionUpdate: null,
        onPermissionRequest: null,
        sessionId: "agent-1",
      } as any,
    });
    session.activate();

    const digest = await session.generateDigest();
    expect(digest).toBe("");
  });
});
```

- [ ] **Step 2: Implement generateDigest on Session**

In `src/core/session.ts`, add after the `autoName()` method (follows the same pause/capture/resume pattern):

```typescript
  async generateDigest(timeoutMs = 10000): Promise<string> {
    let summary = "";

    const captureHandler = (event: AgentEvent) => {
      if (event.type === "text") summary += event.content;
    };

    this.pause((event) => event !== "agent_event");
    this.agentInstance.on("agent_event", captureHandler);

    try {
      const promptPromise = this.agentInstance.prompt(
        "Summarize what you accomplished in this session in 2-3 sentences. Include: key files changed, decisions made, and current status. Reply ONLY with the summary, nothing else.",
      );
      const timeoutPromise = new Promise<void>((_, reject) =>
        setTimeout(() => reject(new Error("digest timeout")), timeoutMs),
      );
      await Promise.race([promptPromise, timeoutPromise]);
      return summary.trim().slice(0, 500);
    } catch {
      this.log.warn("Failed to generate session digest");
      return "";
    } finally {
      this.agentInstance.off("agent_event", captureHandler);
      this.clearBuffer();
      this.resume();
    }
  }
```

- [ ] **Step 3: Run tests**

Run: `pnpm test src/__tests__/session-digest.test.ts`

- [ ] **Step 4: Commit**

```bash
git add src/core/session.ts src/__tests__/session-digest.test.ts
git commit -m "feat(core): add Session.generateDigest() with timeout and fallback"
```

---

### Task 4: Wire digest into SessionBridge + Core

**Files:**
- Modify: `src/core/session-bridge.ts`
- Modify: `src/core/core.ts`
- Modify: `src/core/index.ts`

- [ ] **Step 1: Add DigestStore to core.ts**

Import DigestStore and add property (after `usageBudget`):

```typescript
import { DigestStore } from "./digest-store.js";
```

```typescript
  readonly digestStore: DigestStore | null = null;
```

In the constructor, after usage tracking initialization:

```typescript
    // Digest store
    const digestConfig = config.digest;
    if (digestConfig.enabled) {
      const digestPath = path.join(os.homedir(), ".openacp", "digests.json");
      (this as any).digestStore = new DigestStore(digestPath, digestConfig.retentionDays);
    }
```

In `stop()`, after usage store cleanup:

```typescript
    // 5. Cleanup digest store
    if (this.digestStore) {
      this.digestStore.destroy();
    }
```

- [ ] **Step 2: Wire digest generation in SessionBridge**

In `src/core/session-bridge.ts`, update the `session_end` case. Add digest generation **before** the existing completion notification.

Find the `case "session_end":` block and update it:

```typescript
        case "session_end": {
          this.session.finish(event.reason);
          this.adapter.cleanupSkillCommands(this.session.id);
          this.adapter.sendMessage(
            this.session.id,
            this.deps.messageTransformer.transform(event),
          );

          // Generate digest before notification (fire-and-forget, don't block)
          const digestSummary = await this.session.generateDigest().catch(() => "");

          // Store digest if store is available
          if (this.deps.digestStore && digestSummary) {
            const { nanoid } = await import("nanoid");
            this.deps.digestStore.append({
              id: nanoid(),
              sessionId: this.session.id,
              sessionName: this.session.name || this.session.id,
              agentName: this.session.agentName,
              workingDir: this.session.workingDirectory,
              summary: digestSummary,
              startedAt: this.session.createdAt.toISOString(),
              completedAt: new Date().toISOString(),
              promptCount: this.session.promptCount ?? 0,
              durationMinutes: Math.round(
                (Date.now() - this.session.createdAt.getTime()) / 60000,
              ),
            });
          }

          // Rich notification with digest
          const notifSummary = digestSummary
            ? `${this.session.name || this.session.id} — completed\n\n${digestSummary}`
            : `Session "${this.session.name || this.session.id}" completed`;

          this.deps.notificationManager.notify(this.session.channelId, {
            sessionId: this.session.id,
            sessionName: this.session.name,
            type: "completed",
            summary: notifSummary,
          });
          break;
        }
```

> **Note:** `BridgeDeps` interface needs `digestStore?: DigestStore` added. Also `Session` needs a `promptCount` property — add a simple counter that increments in `enqueuePrompt()`.

Update `BridgeDeps` in session-bridge.ts:

```typescript
import type { DigestStore } from "./digest-store.js";

export interface BridgeDeps {
  messageTransformer: MessageTransformer;
  notificationManager: NotificationManager;
  sessionManager: SessionManager;
  eventBus?: import("./event-bus.js").EventBus;
  fileService?: FileService;
  digestStore?: DigestStore;
}
```

And pass `digestStore` in `core.ts` `createBridge()`:

```typescript
  createBridge(session: Session, adapter: ChannelAdapter): SessionBridge {
    return new SessionBridge(session, adapter, {
      messageTransformer: this.messageTransformer,
      notificationManager: this.notificationManager,
      sessionManager: this.sessionManager,
      eventBus: this.eventBus,
      fileService: this.fileService,
      digestStore: this.digestStore ?? undefined,
    });
  }
```

- [ ] **Step 3: Add promptCount to Session**

In `src/core/session.ts`, add property:

```typescript
  promptCount: number = 0;
```

In `enqueuePrompt()`, increment it:

```typescript
  async enqueuePrompt(text: string, attachments?: Attachment[]): Promise<void> {
    this.promptCount++;
    // ... rest of existing code
```

- [ ] **Step 4: Export from index.ts**

Add to `src/core/index.ts`:

```typescript
export { DigestStore } from './digest-store.js'
```

- [ ] **Step 5: Verify build**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 6: Commit**

```bash
git add src/core/session-bridge.ts src/core/core.ts src/core/session.ts src/core/index.ts
git commit -m "feat(core): wire digest generation into SessionBridge on session completion"
```

---

### Task 5: `/digest` command and formatting

**Files:**
- Modify: `src/adapters/telegram/formatting.ts`
- Modify: `src/adapters/telegram/commands/session.ts`
- Modify: `src/adapters/telegram/commands/index.ts`
- Create: `src/__tests__/digest-command.test.ts`

- [ ] **Step 1: Write failing tests for formatting**

Create `src/__tests__/digest-command.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import { formatDigestList } from "../adapters/telegram/formatting.js";
import type { DigestRecord } from "../core/types.js";

function makeDigest(overrides: Partial<DigestRecord> = {}): DigestRecord {
  return {
    id: "d-1",
    sessionId: "sess-1",
    sessionName: "Fix login bug",
    agentName: "claude",
    workingDir: "/tmp/project",
    summary: "Fixed auth bypass in auth.ts, added token validation.",
    startedAt: new Date().toISOString(),
    completedAt: new Date().toISOString(),
    promptCount: 5,
    durationMinutes: 12,
    ...overrides,
  };
}

describe("formatDigestList", () => {
  it("formats multiple digests", () => {
    const digests = [
      makeDigest({ sessionName: "Fix login", summary: "Fixed auth." }),
      makeDigest({ id: "d-2", sessionName: "Add dark mode", summary: "Added theme toggle.", agentName: "codex" }),
    ];
    const result = formatDigestList(digests);
    expect(result).toContain("Session Digests");
    expect(result).toContain("Fix login");
    expect(result).toContain("Fixed auth.");
    expect(result).toContain("Add dark mode");
    expect(result).toContain("codex");
  });

  it("shows empty message when no digests", () => {
    const result = formatDigestList([]);
    expect(result).toContain("No session digests");
  });

  it("shows duration and prompt count", () => {
    const result = formatDigestList([makeDigest({ durationMinutes: 12, promptCount: 5 })]);
    expect(result).toContain("12 min");
    expect(result).toContain("5 prompts");
  });
});

describe("handleDigest", () => {
  it("shows guidance when digest store is disabled", async () => {
    const { handleDigest } = await import("../adapters/telegram/commands/session.js");
    const ctx = {
      message: { message_thread_id: 1 },
      reply: vi.fn(() => Promise.resolve()),
    } as any;
    const core = { digestStore: null } as any;
    await handleDigest(ctx, core);
    expect(ctx.reply).toHaveBeenCalledWith(
      expect.stringContaining("disabled"),
      expect.any(Object),
    );
  });
});
```

- [ ] **Step 2: Implement formatDigestList in formatting.ts**

Add to `src/adapters/telegram/formatting.ts` (before `splitMessage`):

```typescript
export function formatDigestList(digests: import('../../core/types.js').DigestRecord[]): string {
  if (digests.length === 0) {
    return '📋 <b>Session Digests</b>\n\nNo session digests yet.'
  }

  const lines: string[] = ['📋 <b>Session Digests</b>']

  for (const d of digests) {
    const ago = formatTimeAgo(new Date(d.completedAt))
    lines.push('')
    lines.push(`── <b>${escapeHtml(d.sessionName)}</b> (${escapeHtml(d.agentName)}) · ${ago} ──`)
    lines.push(escapeHtml(d.summary))
    lines.push(`⏱ ${d.durationMinutes} min · 💬 ${d.promptCount} prompts`)
  }

  return lines.join('\n')
}

function formatTimeAgo(date: Date): string {
  const mins = Math.round((Date.now() - date.getTime()) / 60000)
  if (mins < 60) return `${mins}m ago`
  const hours = Math.round(mins / 60)
  if (hours < 24) return `${hours}h ago`
  const days = Math.round(hours / 24)
  return `${days}d ago`
}
```

- [ ] **Step 3: Add handleDigest to session.ts**

In `src/adapters/telegram/commands/session.ts`, add:

```typescript
export async function handleDigest(ctx: Context, core: OpenACPCore): Promise<void> {
  if (!core.digestStore) {
    await ctx.reply("📋 Session digest is disabled.", { parse_mode: "HTML" });
    return;
  }

  const rawMatch = (ctx as Context & { match: unknown }).match;
  const arg = typeof rawMatch === "string" ? rawMatch.trim() : "";

  // Parse: /digest [count|agent_name]
  let limit = 5;
  let agent: string | undefined;

  if (arg) {
    const num = parseInt(arg, 10);
    if (!isNaN(num) && num > 0) {
      limit = Math.min(num, 20);
    } else {
      agent = arg;
    }
  }

  const digests = core.digestStore.query({ limit, agent });
  const text = formatDigestList(digests);
  await ctx.reply(text, { parse_mode: "HTML" });
}
```

Add import for `formatDigestList` at the top of session.ts.

- [ ] **Step 4: Register in index.ts**

Update import:
```typescript
import { ..., handleDigest, ... } from "./session.js";
```

Register command:
```typescript
  bot.command("digest", (ctx) => handleDigest(ctx, core));
```

Add to STATIC_COMMANDS:
```typescript
  { command: "digest", description: "View recent session summaries" },
```

- [ ] **Step 5: Verify build + run tests**

Run: `pnpm exec tsc --noEmit && pnpm test src/__tests__/digest-command.test.ts`

- [ ] **Step 6: Commit**

```bash
git add src/adapters/telegram/formatting.ts src/adapters/telegram/commands/session.ts src/adapters/telegram/commands/index.ts src/__tests__/digest-command.test.ts
git commit -m "feat(telegram): add /digest command with formatted session summaries"
```

---

### Task 6: API endpoint

**Files:**
- Modify: `src/core/api/routes/sessions.ts` (or create dedicated route file)

- [ ] **Step 1: Add GET /api/digests route**

In the sessions route file (or api router), add:

```typescript
// GET /api/digests
if (method === 'GET' && url.startsWith('/api/digests')) {
  const params = new URL(url, 'http://localhost').searchParams;
  const limit = parseInt(params.get('limit') || '20', 10);
  const agent = params.get('agent') || undefined;
  const since = params.get('since') || undefined;

  if (!core.digestStore) {
    return res.json({ ok: false, error: 'Digest store is disabled' });
  }

  const digests = core.digestStore.query({ limit, agent, since });
  return res.json({ ok: true, digests });
}
```

> **Note:** Check the current API routing pattern in `src/core/api/routes/sessions.ts` or `src/core/api/router.ts` and follow the same style.

- [ ] **Step 2: Verify build**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 3: Commit**

```bash
git add src/core/api/
git commit -m "feat(api): add GET /api/digests endpoint"
```

---

### Task 7: Smoke Test & Final Verification

**Files:** None (verification only)

- [ ] **Step 1: Full type check**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 2: Run all tests**

Run: `pnpm test`

- [ ] **Step 3: Verify registration**

Grep to confirm:
- `DigestStore` exported from `index.ts`
- `/digest` registered in `setupCommands`
- `STATIC_COMMANDS` includes `digest`
- `digestStore` in `BridgeDeps` and `createBridge()`
- `generateDigest()` on Session class
- `GET /api/digests` in API routes

- [ ] **Step 4: Final commit (if any fixups needed)**

```bash
git add -A
git commit -m "chore: final cleanup for session digest feature"
```
