# Daily Standup Report — Design Spec

## Summary

Auto-generate and post a daily summary of all agent session activity to the Notifications topic. Covers: completed sessions with what was done, failed sessions with errors, cost totals, and token usage. Runs on a configurable schedule (default 9:00 AM). Also available on-demand via `/report` command and `GET /api/report` endpoint.

## Problem

Teams using OpenACP have no visibility into aggregate agent activity. Individual session notifications exist, but nobody reads 20 separate "Session completed" messages. Managers ask "what did the AI do yesterday?" and nobody has a quick answer. Cost tracking exists but isn't surfaced proactively.

## Requirements

- **Scheduled delivery**: Post report to Notifications topic daily at configurable time
- **On-demand**: `/report` command for instant report, `/report week` for weekly view
- **Session stats**: List completed/failed/cancelled sessions with names, agents, durations
- **Cost summary**: Total tokens, total cost, per-agent breakdown
- **Rich formatting**: Emoji status icons, progress bars for budget, deep links to topics
- **API endpoint**: `GET /api/report?period=today` for dashboard/CLI consumption
- **Configurable**: Enable/disable, schedule time, timezone, include/exclude metrics
- **Backward compatible**: Default disabled — no behavior change for existing users

## Non-Goals

- Per-session AI-generated summaries in the report (use `/summary` for that)
- Email/Slack delivery (future — currently Notifications topic only)
- Historical report storage (reports are ephemeral messages)
- Custom report templates

## Design

### Architecture

```
StandupScheduler (setInterval)
  ↓ fires at configured time
ReportGenerator.generate(period)
  ↓ aggregates data from:
  ├── SessionStore  → sessions by date range (status, duration, agent)
  ├── UsageStore    → tokens, costs per session
  └── Config        → budget info
  ↓ returns ReportData
NotificationManager.notifyAll()
  ↓ formatted by adapter
Notifications topic (Telegram/Discord)
```

### Report Format

```
📊 Daily Standup — Mar 25

✅ Completed (3)
• Fix login bug (claude) — 12 min
• Add dark mode (claude) — 25 min
• Update deps (codex) — 5 min

❌ Failed (1)
• Refactor API (codex) — TypeError in route handler

⚠️ Cancelled (1)
• Debug CSS (claude) — cancelled by user

📈 Metrics
💰 $3.20 · 🔤 89k tokens · 📋 5 sessions
Budget: $3.20 / $50.00 (6%)
▓░░░░░░░░░ 6%

🤖 Per Agent
• claude: 3 sessions · $2.40 · 67k tokens
• codex: 2 sessions · $0.80 · 22k tokens
```

### `/report` Command

```
/report          → today's report
/report week     → last 7 days
/report month    → last 30 days
/report claude   → filter by agent
```

### API Endpoint

```
GET /api/report?period=today&agent=claude
Response: {
  ok: true,
  period: "today",
  sessions: { completed: 3, failed: 1, cancelled: 1 },
  totalTokens: 89000,
  totalCost: 3.20,
  currency: "USD",
  agents: [
    { name: "claude", sessions: 3, tokens: 67000, cost: 2.40 },
    { name: "codex", sessions: 2, tokens: 22000, cost: 0.80 }
  ],
  sessionDetails: [
    { id: "...", name: "Fix login bug", agent: "claude", status: "finished", durationMinutes: 12 }
  ]
}
```

### Config

```typescript
const StandupSchema = z.object({
  enabled: z.boolean().default(false),
  time: z.string().default("09:00"),    // HH:MM in local time
  timezone: z.string().default("UTC"),
  includeMetrics: z.boolean().default(true),
  includeSessions: z.boolean().default(true),
}).default({});
```

### ReportData Interface

```typescript
interface ReportData {
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

interface SessionReportEntry {
  sessionId: string;
  name: string;
  agentName: string;
  status: string;
  durationMinutes: number;
  errorMessage?: string;
}
```

### Scheduling

Use `setTimeout` + time calculation (not cron library) to fire at the configured time:

```typescript
// Calculate ms until next 09:00
const now = new Date();
const target = new Date(now);
target.setHours(9, 0, 0, 0);
if (target <= now) target.setDate(target.getDate() + 1);
const msUntilTarget = target.getTime() - now.getTime();

setTimeout(() => {
  generateAndSend();
  // Then repeat every 24h
  setInterval(generateAndSend, 24 * 60 * 60 * 1000);
}, msUntilTarget);
```

### Error Handling

| Scenario | Behavior |
|----------|----------|
| No sessions in period | Post "No agent activity" message |
| UsageStore disabled | Skip metrics section, show sessions only |
| Notification send fails | Log error, retry once after 5 min |
| Config time invalid | Default to 09:00 UTC |

### Affected Components

**Core layer** (new):
- `src/core/report-generator.ts` — `ReportGenerator` class: aggregate data, produce `ReportData`
- `src/core/standup-scheduler.ts` — Schedule daily reports, manage timer

**Core layer** (modify):
- `src/core/config.ts` — Add `StandupSchema` to ConfigSchema
- `src/core/core.ts` — Create scheduler in constructor, destroy in stop()
- `src/core/types.ts` — Add `ReportData`, `SessionReportEntry` interfaces
- `src/core/index.ts` — Export new modules
- `src/core/api/routes/sessions.ts` — Add `GET /api/report` endpoint

**Adapter layer** (modify):
- `src/adapters/telegram/formatting.ts` — Add `formatStandupReport()`
- `src/adapters/telegram/commands/session.ts` — Add `handleReport()`
- `src/adapters/telegram/commands/index.ts` — Register `/report` command
