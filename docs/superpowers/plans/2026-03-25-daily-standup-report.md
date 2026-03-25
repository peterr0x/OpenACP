# Daily Standup Report Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Auto-post daily summary of all agent session activity to Notifications topic. Also available on-demand via `/report` command and `GET /api/report`.

**Architecture:** `ReportGenerator` aggregates data from `SessionStore` + `UsageStore`. `StandupScheduler` fires at configured time. Report delivered via `NotificationManager`. On-demand via `/report` command and API.

**Tech Stack:** TypeScript, vitest

**Spec:** `docs/superpowers/specs/2026-03-25-daily-standup-report-design.md`

---

## File Structure

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `src/core/report-generator.ts` | Aggregate session + usage data into `ReportData` |
| Create | `src/core/standup-scheduler.ts` | Schedule daily report delivery |
| Create | `src/__tests__/report-generator.test.ts` | Tests for data aggregation |
| Create | `src/__tests__/standup-scheduler.test.ts` | Tests for scheduling logic |
| Create | `src/__tests__/report-command.test.ts` | Tests for /report command + formatting |
| Modify | `src/core/types.ts` | Add `ReportData`, `SessionReportEntry` interfaces |
| Modify | `src/core/config.ts` | Add `StandupSchema` |
| Modify | `src/core/core.ts` | Create scheduler, destroy in stop() |
| Modify | `src/core/index.ts` | Export new modules |
| Modify | `src/core/api/routes/sessions.ts` | Add `GET /api/report` endpoint |
| Modify | `src/adapters/telegram/formatting.ts` | Add `formatStandupReport()` |
| Modify | `src/adapters/telegram/commands/session.ts` | Add `handleReport()` |
| Modify | `src/adapters/telegram/commands/index.ts` | Register `/report` command |

---

### Task 1: Types and Config

**Files:**
- Modify: `src/core/types.ts`
- Modify: `src/core/config.ts`

- [ ] **Step 1: Add report types to types.ts**

Append at the end of `src/core/types.ts`:

```typescript
export interface SessionReportEntry {
  sessionId: string;
  name: string;
  agentName: string;
  status: string;
  durationMinutes: number;
  errorMessage?: string;
}

export interface ReportData {
  period: string;
  startDate: string;
  endDate: string;
  sessions: {
    completed: SessionReportEntry[];
    failed: SessionReportEntry[];
    cancelled: SessionReportEntry[];
  };
  metrics: {
    totalTokens: number;
    totalCost: number;
    currency: string;
    sessionCount: number;
    totalDurationMinutes: number;
  };
  perAgent: Array<{
    name: string;
    sessionCount: number;
    tokens: number;
    cost: number;
  }>;
  budget?: {
    used: number;
    limit: number;
    percent: number;
  };
}
```

- [ ] **Step 2: Add StandupSchema to config.ts**

In `src/core/config.ts`, add after the `UsageSchema` block:

```typescript
const StandupSchema = z
  .object({
    enabled: z.boolean().default(false),
    time: z.string().default("09:00"),
    timezone: z.string().default("UTC"),
    includeMetrics: z.boolean().default(true),
    includeSessions: z.boolean().default(true),
  })
  .default({});

export type StandupConfig = z.infer<typeof StandupSchema>;
```

Add `standup: StandupSchema,` to `ConfigSchema` (after `usage` field).

- [ ] **Step 3: Verify build**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 4: Commit**

```bash
git add src/core/types.ts src/core/config.ts
git commit -m "feat(types): add ReportData types and StandupSchema config"
```

---

### Task 2: ReportGenerator

**Files:**
- Create: `src/core/report-generator.ts`
- Create: `src/__tests__/report-generator.test.ts`

- [ ] **Step 1: Write failing tests**

Create `src/__tests__/report-generator.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import { ReportGenerator } from "../core/report-generator.js";
import type { SessionRecord } from "../core/types.js";

function makeRecord(overrides: Partial<SessionRecord> = {}): SessionRecord {
  return {
    sessionId: "sess-1",
    agentSessionId: "agent-1",
    agentName: "claude",
    workingDir: "/tmp",
    channelId: "telegram",
    status: "finished",
    createdAt: new Date(Date.now() - 12 * 60000).toISOString(), // 12 min ago
    lastActiveAt: new Date().toISOString(),
    name: "Fix login bug",
    platform: {},
    ...overrides,
  };
}

describe("ReportGenerator", () => {
  it("generates report with completed sessions", () => {
    const sessions = [
      makeRecord({ sessionId: "s1", name: "Fix login", status: "finished" }),
      makeRecord({ sessionId: "s2", name: "Add feature", status: "finished", agentName: "codex" }),
    ];

    const generator = new ReportGenerator();
    const report = generator.generate(sessions, [], "today");

    expect(report.sessions.completed).toHaveLength(2);
    expect(report.sessions.completed[0].name).toBe("Fix login");
    expect(report.metrics.sessionCount).toBe(2);
  });

  it("separates by status", () => {
    const sessions = [
      makeRecord({ sessionId: "s1", status: "finished" }),
      makeRecord({ sessionId: "s2", status: "error", name: "Broken" }),
      makeRecord({ sessionId: "s3", status: "cancelled", name: "Cancelled" }),
    ];

    const generator = new ReportGenerator();
    const report = generator.generate(sessions, [], "today");

    expect(report.sessions.completed).toHaveLength(1);
    expect(report.sessions.failed).toHaveLength(1);
    expect(report.sessions.cancelled).toHaveLength(1);
  });

  it("calculates per-agent breakdown", () => {
    const sessions = [
      makeRecord({ sessionId: "s1", agentName: "claude", status: "finished" }),
      makeRecord({ sessionId: "s2", agentName: "claude", status: "finished" }),
      makeRecord({ sessionId: "s3", agentName: "codex", status: "finished" }),
    ];
    const usage = [
      { sessionId: "s1", tokensUsed: 1000, cost: { amount: 0.05, currency: "USD" } },
      { sessionId: "s2", tokensUsed: 2000, cost: { amount: 0.10, currency: "USD" } },
      { sessionId: "s3", tokensUsed: 500, cost: { amount: 0.02, currency: "USD" } },
    ];

    const generator = new ReportGenerator();
    const report = generator.generate(sessions, usage as any, "today");

    expect(report.perAgent).toHaveLength(2);
    const claude = report.perAgent.find(a => a.name === "claude")!;
    expect(claude.sessionCount).toBe(2);
    expect(claude.tokens).toBe(3000);
    expect(claude.cost).toBeCloseTo(0.15);
  });

  it("calculates duration from timestamps", () => {
    const now = new Date();
    const start = new Date(now.getTime() - 15 * 60000); // 15 min ago
    const sessions = [
      makeRecord({ createdAt: start.toISOString(), lastActiveAt: now.toISOString() }),
    ];

    const generator = new ReportGenerator();
    const report = generator.generate(sessions, [], "today");

    expect(report.sessions.completed[0].durationMinutes).toBe(15);
  });

  it("returns empty report when no sessions", () => {
    const generator = new ReportGenerator();
    const report = generator.generate([], [], "today");

    expect(report.sessions.completed).toHaveLength(0);
    expect(report.sessions.failed).toHaveLength(0);
    expect(report.metrics.sessionCount).toBe(0);
    expect(report.metrics.totalTokens).toBe(0);
  });

  it("includes budget info when provided", () => {
    const generator = new ReportGenerator();
    const report = generator.generate([], [], "today", { used: 5, limit: 50, percent: 10 });

    expect(report.budget).toEqual({ used: 5, limit: 50, percent: 10 });
  });

  it("filters sessions by date range", () => {
    const now = new Date();
    const yesterday = new Date(now.getTime() - 2 * 24 * 60 * 60 * 1000); // 2 days ago
    const sessions = [
      makeRecord({ sessionId: "old", lastActiveAt: yesterday.toISOString() }),
      makeRecord({ sessionId: "new", lastActiveAt: now.toISOString() }),
    ];

    const generator = new ReportGenerator();
    const report = generator.generate(sessions, [], "today");

    expect(report.metrics.sessionCount).toBe(1);
  });
});
```

- [ ] **Step 2: Implement ReportGenerator**

Create `src/core/report-generator.ts`:

```typescript
import type { SessionRecord, ReportData, SessionReportEntry } from "./types.js";

interface UsageEntry {
  sessionId: string;
  tokensUsed: number;
  cost?: { amount: number; currency: string };
}

export class ReportGenerator {
  generate(
    allSessions: SessionRecord[],
    usageRecords: UsageEntry[],
    period: string,
    budget?: { used: number; limit: number; percent: number },
  ): ReportData {
    const now = new Date();
    const cutoff = this.getCutoff(period, now);

    // Filter sessions by date range
    const sessions = allSessions.filter(s => {
      const activeAt = new Date(s.lastActiveAt || s.createdAt);
      return activeAt.getTime() >= cutoff;
    });

    // Group by status
    const completed: SessionReportEntry[] = [];
    const failed: SessionReportEntry[] = [];
    const cancelled: SessionReportEntry[] = [];

    for (const s of sessions) {
      const entry: SessionReportEntry = {
        sessionId: s.sessionId,
        name: s.name || `Session ${s.sessionId.slice(0, 6)}`,
        agentName: s.agentName,
        status: s.status,
        durationMinutes: Math.round(
          (new Date(s.lastActiveAt || s.createdAt).getTime() - new Date(s.createdAt).getTime()) / 60000,
        ),
      };

      switch (s.status) {
        case "finished":
          completed.push(entry);
          break;
        case "error":
          failed.push(entry);
          break;
        case "cancelled":
          cancelled.push(entry);
          break;
      }
    }

    // Usage aggregation per session
    const sessionUsage = new Map<string, { tokens: number; cost: number }>();
    for (const u of usageRecords) {
      const existing = sessionUsage.get(u.sessionId) || { tokens: 0, cost: 0 };
      existing.tokens += u.tokensUsed;
      existing.cost += u.cost?.amount ?? 0;
      sessionUsage.set(u.sessionId, existing);
    }

    // Filter usage to sessions in this period
    const sessionIds = new Set(sessions.map(s => s.sessionId));
    let totalTokens = 0;
    let totalCost = 0;
    for (const [sid, usage] of sessionUsage) {
      if (sessionIds.has(sid)) {
        totalTokens += usage.tokens;
        totalCost += usage.cost;
      }
    }

    // Per-agent breakdown
    const agentMap = new Map<string, { sessionCount: number; tokens: number; cost: number }>();
    for (const s of sessions) {
      const agent = agentMap.get(s.agentName) || { sessionCount: 0, tokens: 0, cost: 0 };
      agent.sessionCount++;
      const usage = sessionUsage.get(s.sessionId);
      if (usage) {
        agent.tokens += usage.tokens;
        agent.cost += usage.cost;
      }
      agentMap.set(s.agentName, agent);
    }

    const currency = usageRecords.find(u => u.cost?.currency)?.cost?.currency ?? "USD";
    const totalDuration = [...completed, ...failed, ...cancelled].reduce((sum, e) => sum + e.durationMinutes, 0);

    return {
      period,
      startDate: new Date(cutoff).toISOString(),
      endDate: now.toISOString(),
      sessions: { completed, failed, cancelled },
      metrics: {
        totalTokens,
        totalCost,
        currency,
        sessionCount: sessions.length,
        totalDurationMinutes: totalDuration,
      },
      perAgent: [...agentMap.entries()].map(([name, data]) => ({ name, ...data })),
      budget,
    };
  }

  private getCutoff(period: string, now: Date): number {
    switch (period) {
      case "today": {
        const start = new Date(now);
        start.setHours(0, 0, 0, 0);
        return start.getTime();
      }
      case "week":
        return now.getTime() - 7 * 24 * 60 * 60 * 1000;
      case "month":
        return new Date(now.getFullYear(), now.getMonth(), 1).getTime();
      default:
        return 0;
    }
  }
}
```

- [ ] **Step 3: Run tests**

Run: `pnpm test src/__tests__/report-generator.test.ts`

- [ ] **Step 4: Commit**

```bash
git add src/core/report-generator.ts src/__tests__/report-generator.test.ts
git commit -m "feat(core): add ReportGenerator for session activity aggregation"
```

---

### Task 3: StandupScheduler

**Files:**
- Create: `src/core/standup-scheduler.ts`
- Create: `src/__tests__/standup-scheduler.test.ts`

- [ ] **Step 1: Write failing tests**

Create `src/__tests__/standup-scheduler.test.ts`:

```typescript
import { describe, it, expect, vi, afterEach } from "vitest";
import { StandupScheduler } from "../core/standup-scheduler.js";

describe("StandupScheduler", () => {
  afterEach(() => {
    vi.restoreAllMocks();
  });

  it("calculates correct ms until target time", () => {
    const scheduler = new StandupScheduler();
    const ms = scheduler.msUntilTime("09:00", "UTC");
    expect(ms).toBeGreaterThan(0);
    expect(ms).toBeLessThanOrEqual(24 * 60 * 60 * 1000);
  });

  it("schedules callback execution", () => {
    vi.useFakeTimers();
    const callback = vi.fn();
    const scheduler = new StandupScheduler();
    scheduler.start(callback, "09:00", "UTC");

    // Should have set a timer
    expect(callback).not.toHaveBeenCalled();

    scheduler.stop();
    vi.useRealTimers();
  });

  it("stops cleanly", () => {
    const scheduler = new StandupScheduler();
    scheduler.start(vi.fn(), "09:00", "UTC");
    scheduler.stop();
    // Should not throw
  });

  it("handles stop when not started", () => {
    const scheduler = new StandupScheduler();
    scheduler.stop(); // Should not throw
  });
});
```

- [ ] **Step 2: Implement StandupScheduler**

Create `src/core/standup-scheduler.ts`:

```typescript
import { createChildLogger } from "./log.js";

const log = createChildLogger({ module: "standup-scheduler" });

export class StandupScheduler {
  private initialTimer: ReturnType<typeof setTimeout> | null = null;
  private repeatInterval: ReturnType<typeof setInterval> | null = null;

  start(callback: () => void, time: string, _timezone: string): void {
    const ms = this.msUntilTime(time, _timezone);
    log.info({ time, msUntilFirst: ms }, "Standup scheduler started");

    this.initialTimer = setTimeout(() => {
      callback();
      this.repeatInterval = setInterval(callback, 24 * 60 * 60 * 1000);
    }, ms);
  }

  stop(): void {
    if (this.initialTimer) clearTimeout(this.initialTimer);
    if (this.repeatInterval) clearInterval(this.repeatInterval);
    this.initialTimer = null;
    this.repeatInterval = null;
  }

  msUntilTime(time: string, _timezone: string): number {
    const [hours, minutes] = time.split(":").map(Number);
    const now = new Date();
    const target = new Date(now);
    target.setHours(hours, minutes, 0, 0);
    if (target.getTime() <= now.getTime()) {
      target.setDate(target.getDate() + 1);
    }
    return target.getTime() - now.getTime();
  }
}
```

- [ ] **Step 3: Run tests**

Run: `pnpm test src/__tests__/standup-scheduler.test.ts`

- [ ] **Step 4: Commit**

```bash
git add src/core/standup-scheduler.ts src/__tests__/standup-scheduler.test.ts
git commit -m "feat(core): add StandupScheduler for daily report delivery"
```

---

### Task 4: Wire into Core

**Files:**
- Modify: `src/core/core.ts`
- Modify: `src/core/index.ts`

- [ ] **Step 1: Add imports and properties to core.ts**

Add imports:

```typescript
import { ReportGenerator } from "./report-generator.js";
import { StandupScheduler } from "./standup-scheduler.js";
```

Add properties (after `usageBudget`):

```typescript
  private readonly reportGenerator = new ReportGenerator();
  private readonly standupScheduler = new StandupScheduler();
```

- [ ] **Step 2: Initialize scheduler in constructor**

After usage tracking initialization, add:

```typescript
    // Daily standup report
    const standupConfig = config.standup;
    if (standupConfig.enabled) {
      this.standupScheduler.start(
        () => this.sendDailyReport(),
        standupConfig.time,
        standupConfig.timezone,
      );
    }
```

- [ ] **Step 3: Add sendDailyReport method**

Add to `OpenACPCore`:

```typescript
  async sendDailyReport(): Promise<void> {
    try {
      const report = this.generateReport("today");
      const lines: string[] = [];
      const d = new Date();
      lines.push(`📊 <b>Daily Standup — ${d.toLocaleDateString("en-US", { month: "short", day: "numeric" })}</b>\n`);

      if (report.sessions.completed.length > 0) {
        lines.push(`✅ <b>Completed (${report.sessions.completed.length})</b>`);
        for (const s of report.sessions.completed) {
          lines.push(`• ${s.name} (${s.agentName}) — ${s.durationMinutes} min`);
        }
        lines.push("");
      }

      if (report.sessions.failed.length > 0) {
        lines.push(`❌ <b>Failed (${report.sessions.failed.length})</b>`);
        for (const s of report.sessions.failed) {
          lines.push(`• ${s.name} (${s.agentName})${s.errorMessage ? ` — ${s.errorMessage}` : ""}`);
        }
        lines.push("");
      }

      if (report.sessions.cancelled.length > 0) {
        lines.push(`⚠️ <b>Cancelled (${report.sessions.cancelled.length})</b>`);
        for (const s of report.sessions.cancelled) {
          lines.push(`• ${s.name} (${s.agentName})`);
        }
        lines.push("");
      }

      if (report.metrics.sessionCount === 0) {
        lines.push("No agent activity today.");
      } else {
        lines.push(`📈 <b>Metrics</b>`);
        lines.push(`💰 $${report.metrics.totalCost.toFixed(2)} · 🔤 ${Math.round(report.metrics.totalTokens / 1000)}k tokens · 📋 ${report.metrics.sessionCount} sessions`);
        if (report.budget && report.budget.limit > 0) {
          const filled = Math.round(Math.min(report.budget.percent / 100, 1) * 10);
          const bar = "▓".repeat(filled) + "░".repeat(10 - filled);
          lines.push(`Budget: $${report.budget.used.toFixed(2)} / $${report.budget.limit.toFixed(2)} (${report.budget.percent}%)`);
          lines.push(`${bar} ${report.budget.percent}%`);
        }
      }

      if (report.perAgent.length > 1) {
        lines.push("");
        lines.push(`🤖 <b>Per Agent</b>`);
        for (const a of report.perAgent) {
          lines.push(`• ${a.name}: ${a.sessionCount} sessions · $${a.cost.toFixed(2)} · ${Math.round(a.tokens / 1000)}k tokens`);
        }
      }

      await this.notificationManager.notifyAll({
        sessionId: "system",
        sessionName: "Daily Report",
        type: "completed",
        summary: lines.join("\n"),
      });
    } catch (err) {
      log.error({ err }, "Failed to send daily standup report");
    }
  }

  generateReport(period: string, agent?: string): import("./types.js").ReportData {
    const allSessions = this.sessionStore?.list() ?? [];
    const filteredSessions = agent
      ? allSessions.filter(s => s.agentName === agent)
      : allSessions;

    const usageRecords = this.usageStore
      ? (this.usageStore as any).records ?? []
      : [];

    const budget = this.usageBudget
      ? this.usageBudget.getStatus()
      : undefined;

    return this.reportGenerator.generate(
      filteredSessions,
      usageRecords,
      period,
      budget ? { used: budget.used, limit: budget.budget, percent: budget.percent } : undefined,
    );
  }
```

- [ ] **Step 4: Cleanup in stop()**

In `stop()`, after usage store cleanup:

```typescript
    // 5. Stop standup scheduler
    this.standupScheduler.stop();
```

- [ ] **Step 5: Export from index.ts**

Add to `src/core/index.ts`:

```typescript
export { ReportGenerator } from './report-generator.js'
export { StandupScheduler } from './standup-scheduler.js'
```

- [ ] **Step 6: Verify build**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 7: Commit**

```bash
git add src/core/core.ts src/core/index.ts
git commit -m "feat(core): wire StandupScheduler and ReportGenerator into OpenACPCore"
```

---

### Task 5: `/report` command and formatting

**Files:**
- Modify: `src/adapters/telegram/commands/session.ts`
- Modify: `src/adapters/telegram/commands/index.ts`
- Create: `src/__tests__/report-command.test.ts`

- [ ] **Step 1: Add handleReport to session.ts**

```typescript
export async function handleReport(ctx: Context, core: OpenACPCore): Promise<void> {
  const rawMatch = (ctx as Context & { match: unknown }).match;
  const arg = typeof rawMatch === "string" ? rawMatch.trim().toLowerCase() : "";

  let period = "today";
  let agent: string | undefined;

  if (arg === "week" || arg === "month") {
    period = arg;
  } else if (arg) {
    agent = arg;
  }

  const report = core.generateReport(period, agent);

  const lines: string[] = [];
  const periodLabel = period === "today" ? "Today" : period === "week" ? "This Week" : "This Month";
  lines.push(`📊 <b>Report — ${periodLabel}${agent ? ` (${escapeHtml(agent)})` : ""}</b>\n`);

  if (report.sessions.completed.length > 0) {
    lines.push(`✅ <b>Completed (${report.sessions.completed.length})</b>`);
    for (const s of report.sessions.completed.slice(0, 10)) {
      lines.push(`• ${escapeHtml(s.name)} (${escapeHtml(s.agentName)}) — ${s.durationMinutes} min`);
    }
    if (report.sessions.completed.length > 10) {
      lines.push(`  ... and ${report.sessions.completed.length - 10} more`);
    }
    lines.push("");
  }

  if (report.sessions.failed.length > 0) {
    lines.push(`❌ <b>Failed (${report.sessions.failed.length})</b>`);
    for (const s of report.sessions.failed.slice(0, 5)) {
      lines.push(`• ${escapeHtml(s.name)} (${escapeHtml(s.agentName)})`);
    }
    lines.push("");
  }

  if (report.sessions.cancelled.length > 0) {
    lines.push(`⚠️ <b>Cancelled (${report.sessions.cancelled.length})</b>`);
    for (const s of report.sessions.cancelled.slice(0, 5)) {
      lines.push(`• ${escapeHtml(s.name)} (${escapeHtml(s.agentName)})`);
    }
    lines.push("");
  }

  if (report.metrics.sessionCount === 0) {
    lines.push("No agent activity in this period.");
  } else {
    lines.push(`📈 <b>Metrics</b>`);
    lines.push(`💰 $${report.metrics.totalCost.toFixed(2)} · 🔤 ${Math.round(report.metrics.totalTokens / 1000)}k tokens · 📋 ${report.metrics.sessionCount} sessions`);

    if (report.perAgent.length > 1) {
      lines.push("");
      lines.push(`🤖 <b>Per Agent</b>`);
      for (const a of report.perAgent) {
        lines.push(`• ${escapeHtml(a.name)}: ${a.sessionCount} sessions · $${a.cost.toFixed(2)}`);
      }
    }
  }

  await ctx.reply(lines.join("\n"), { parse_mode: "HTML" });
}
```

- [ ] **Step 2: Register in index.ts**

Update import:
```typescript
import { ..., handleReport, ... } from "./session.js";
```

Register command:
```typescript
  bot.command("report", (ctx) => handleReport(ctx, core));
```

Add to STATIC_COMMANDS:
```typescript
  { command: "report", description: "View session activity report (/report, /report week, /report month)" },
```

- [ ] **Step 3: Write tests**

Create `src/__tests__/report-command.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import { handleReport } from "../adapters/telegram/commands/session.js";

describe("handleReport", () => {
  it("generates today report by default", async () => {
    const ctx = {
      message: { message_thread_id: 1 },
      match: "",
      reply: vi.fn(() => Promise.resolve()),
    } as any;
    const core = {
      generateReport: vi.fn(() => ({
        sessions: { completed: [], failed: [], cancelled: [] },
        metrics: { totalTokens: 0, totalCost: 0, currency: "USD", sessionCount: 0, totalDurationMinutes: 0 },
        perAgent: [],
      })),
    } as any;

    await handleReport(ctx, core);
    expect(core.generateReport).toHaveBeenCalledWith("today", undefined);
    expect(ctx.reply).toHaveBeenCalledWith(
      expect.stringContaining("Today"),
      expect.any(Object),
    );
  });

  it("accepts week period", async () => {
    const ctx = {
      message: { message_thread_id: 1 },
      match: "week",
      reply: vi.fn(() => Promise.resolve()),
    } as any;
    const core = {
      generateReport: vi.fn(() => ({
        sessions: { completed: [], failed: [], cancelled: [] },
        metrics: { totalTokens: 0, totalCost: 0, currency: "USD", sessionCount: 0, totalDurationMinutes: 0 },
        perAgent: [],
      })),
    } as any;

    await handleReport(ctx, core);
    expect(core.generateReport).toHaveBeenCalledWith("week", undefined);
  });

  it("accepts agent filter", async () => {
    const ctx = {
      message: { message_thread_id: 1 },
      match: "claude",
      reply: vi.fn(() => Promise.resolve()),
    } as any;
    const core = {
      generateReport: vi.fn(() => ({
        sessions: { completed: [{ name: "Test", agentName: "claude", durationMinutes: 5 }], failed: [], cancelled: [] },
        metrics: { totalTokens: 1000, totalCost: 0.05, currency: "USD", sessionCount: 1, totalDurationMinutes: 5 },
        perAgent: [{ name: "claude", sessionCount: 1, tokens: 1000, cost: 0.05 }],
      })),
    } as any;

    await handleReport(ctx, core);
    expect(core.generateReport).toHaveBeenCalledWith("today", "claude");
    expect(ctx.reply).toHaveBeenCalledWith(
      expect.stringContaining("claude"),
      expect.any(Object),
    );
  });
});
```

- [ ] **Step 4: Run tests**

Run: `pnpm test src/__tests__/report-command.test.ts`

- [ ] **Step 5: Commit**

```bash
git add src/adapters/telegram/commands/session.ts src/adapters/telegram/commands/index.ts src/__tests__/report-command.test.ts
git commit -m "feat(telegram): add /report command for on-demand session activity reports"
```

---

### Task 6: API endpoint

**Files:**
- Modify: `src/core/api/routes/sessions.ts`

- [ ] **Step 1: Add GET /api/report route**

Add in the sessions route file:

```typescript
  router.get("/api/report", async (req, res) => {
    const url = new URL(req.url ?? "", "http://localhost");
    const period = url.searchParams.get("period") || "today";
    const agent = url.searchParams.get("agent") || undefined;
    const report = deps.core.generateReport(period, agent);
    deps.sendJson(res, 200, { ok: true, ...report });
  });
```

- [ ] **Step 2: Verify build**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 3: Commit**

```bash
git add src/core/api/routes/sessions.ts
git commit -m "feat(api): add GET /api/report endpoint"
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
- `ReportGenerator` and `StandupScheduler` exported from `index.ts`
- `/report` registered in `setupCommands`
- `STATIC_COMMANDS` includes `report`
- `generateReport()` and `sendDailyReport()` on OpenACPCore
- `GET /api/report` in API routes
- `StandupSchema` in ConfigSchema

- [ ] **Step 4: Final commit (if any fixups needed)**

```bash
git add -A
git commit -m "chore: final cleanup for daily standup report feature"
```
