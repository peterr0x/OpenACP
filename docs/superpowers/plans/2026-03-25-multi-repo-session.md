# Multi-Repo Session Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow a single session to access multiple repositories via symlink aggregation. Agent sees all repos under a unified temp directory.

**Architecture:** `WorkspaceManager` creates `/tmp/openacp-ws-{sessionId}/` with symlinks to each repo. Agent spawns with temp dir as `cwd`. Cleanup on session end + orphan cleanup on startup.

**Tech Stack:** TypeScript, Node.js fs (symlinks), vitest

**Spec:** `docs/superpowers/specs/2026-03-25-multi-repo-session-design.md`

---

## File Structure

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `src/core/workspace-manager.ts` | Symlink dir creation, cleanup, path resolution |
| Create | `src/__tests__/workspace-manager.test.ts` | Tests for symlink creation, cleanup, edge cases |
| Create | `src/__tests__/multi-repo-command.test.ts` | Tests for command parsing |
| Modify | `src/core/types.ts` | Add `workspaces?: string[]` to SessionRecord |
| Modify | `src/core/session.ts` | Add `workspaces?: string[]` property |
| Modify | `src/core/core.ts` | Pass workspaces through creation pipeline, cleanup |
| Modify | `src/core/session-bridge.ts` | Cleanup temp dir on session end |
| Modify | `src/core/index.ts` | Export WorkspaceManager |
| Modify | `src/adapters/telegram/commands/new-session.ts` | Parse comma-separated workspace paths |
| Modify | `src/main.ts` | Cleanup orphan temp dirs on startup |

---

### Task 1: WorkspaceManager

**Files:**
- Create: `src/core/workspace-manager.ts`
- Create: `src/__tests__/workspace-manager.test.ts`

- [ ] **Step 1: Write failing tests**

Create `src/__tests__/workspace-manager.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import fs from "node:fs";
import path from "node:path";
import os from "node:os";
import { WorkspaceManager } from "../core/workspace-manager.js";

describe("WorkspaceManager", () => {
  let tmpDir: string;
  let repo1: string;
  let repo2: string;

  beforeEach(() => {
    tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), "openacp-ws-test-"));
    repo1 = path.join(tmpDir, "frontend");
    repo2 = path.join(tmpDir, "backend");
    fs.mkdirSync(repo1);
    fs.mkdirSync(repo2);
    // Create a file in each to verify symlink works
    fs.writeFileSync(path.join(repo1, "index.ts"), "export default 1;");
    fs.writeFileSync(path.join(repo2, "server.ts"), "export default 2;");
  });

  afterEach(() => {
    fs.rmSync(tmpDir, { recursive: true, force: true });
  });

  it("creates symlink directory for multiple paths", () => {
    const manager = new WorkspaceManager();
    const result = manager.create("sess-1", [repo1, repo2]);

    expect(result.rootDir).toContain("openacp-ws-sess-1");
    expect(fs.existsSync(result.rootDir)).toBe(true);
    expect(fs.existsSync(path.join(result.rootDir, "frontend"))).toBe(true);
    expect(fs.existsSync(path.join(result.rootDir, "backend"))).toBe(true);

    // Verify symlinks point to real dirs
    const frontendTarget = fs.readlinkSync(path.join(result.rootDir, "frontend"));
    expect(frontendTarget).toBe(repo1);

    // Verify files accessible through symlinks
    const content = fs.readFileSync(path.join(result.rootDir, "frontend", "index.ts"), "utf-8");
    expect(content).toBe("export default 1;");

    manager.cleanup(result.rootDir);
  });

  it("handles duplicate directory names", () => {
    const sub1 = path.join(tmpDir, "app");
    const sub2 = path.join(tmpDir, "other", "app");
    fs.mkdirSync(sub1);
    fs.mkdirSync(path.join(tmpDir, "other"));
    fs.mkdirSync(sub2);

    const manager = new WorkspaceManager();
    const result = manager.create("sess-2", [sub1, sub2]);

    // Should have app and app-2
    expect(fs.existsSync(path.join(result.rootDir, "app"))).toBe(true);
    expect(fs.existsSync(path.join(result.rootDir, "app-2"))).toBe(true);

    manager.cleanup(result.rootDir);
  });

  it("returns single path unchanged for single workspace", () => {
    const manager = new WorkspaceManager();
    const result = manager.resolve([repo1]);

    expect(result.rootDir).toBe(repo1);
    expect(result.workspaces).toEqual([repo1]);
    expect(result.isMulti).toBe(false);
  });

  it("throws when path does not exist", () => {
    const manager = new WorkspaceManager();
    expect(() => manager.create("sess-3", ["/nonexistent/path"])).toThrow("not found");
  });

  it("cleanup removes only symlinks, not targets", () => {
    const manager = new WorkspaceManager();
    const result = manager.create("sess-4", [repo1, repo2]);

    manager.cleanup(result.rootDir);

    // Temp dir should be gone
    expect(fs.existsSync(result.rootDir)).toBe(false);
    // Original repos should still exist
    expect(fs.existsSync(repo1)).toBe(true);
    expect(fs.existsSync(repo2)).toBe(true);
    expect(fs.existsSync(path.join(repo1, "index.ts"))).toBe(true);
  });

  it("cleanup is safe on non-temp paths", () => {
    const manager = new WorkspaceManager();
    // Should not delete arbitrary paths
    manager.cleanup(repo1); // not a temp dir
    expect(fs.existsSync(repo1)).toBe(true); // still exists
  });

  it("cleanupOrphans removes old temp dirs", () => {
    const manager = new WorkspaceManager();
    const result = manager.create("orphan-1", [repo1]);

    // Verify it exists
    expect(fs.existsSync(result.rootDir)).toBe(true);

    // Cleanup orphans
    manager.cleanupOrphans();

    // Should be cleaned up (no active session owns it)
    // Note: in real impl, would check against active session IDs
  });
});
```

- [ ] **Step 2: Implement WorkspaceManager**

Create `src/core/workspace-manager.ts`:

```typescript
import fs from "node:fs";
import path from "node:path";
import os from "node:os";
import { createChildLogger } from "./log.js";

const log = createChildLogger({ module: "workspace-manager" });

const TEMP_PREFIX = "openacp-ws-";

interface WorkspaceResult {
  rootDir: string;
  workspaces: string[];
  isMulti: boolean;
}

export class WorkspaceManager {
  resolve(paths: string[]): WorkspaceResult {
    if (paths.length <= 1) {
      return { rootDir: paths[0], workspaces: paths, isMulti: false };
    }
    // Multi-workspace needs create()
    return { rootDir: paths[0], workspaces: paths, isMulti: true };
  }

  create(sessionId: string, paths: string[]): WorkspaceResult {
    // Validate all paths exist
    for (const p of paths) {
      if (!fs.existsSync(p)) {
        throw new Error(`Workspace path not found: ${p}`);
      }
    }

    if (paths.length <= 1) {
      return { rootDir: paths[0], workspaces: paths, isMulti: false };
    }

    // Create temp root directory
    const rootDir = path.join(os.tmpdir(), `${TEMP_PREFIX}${sessionId}`);
    fs.mkdirSync(rootDir, { recursive: true });

    // Track used names to handle duplicates
    const usedNames = new Set<string>();

    for (const p of paths) {
      let name = path.basename(p);
      if (usedNames.has(name)) {
        let counter = 2;
        while (usedNames.has(`${name}-${counter}`)) counter++;
        name = `${name}-${counter}`;
      }
      usedNames.add(name);

      const linkPath = path.join(rootDir, name);
      fs.symlinkSync(p, linkPath, "dir");
    }

    log.info({ sessionId, rootDir, workspaces: paths }, "Multi-workspace created");
    return { rootDir, workspaces: paths, isMulti: true };
  }

  cleanup(rootDir: string): void {
    // Safety: only remove temp dirs created by us
    if (!rootDir.includes(TEMP_PREFIX)) {
      return;
    }
    try {
      fs.rmSync(rootDir, { recursive: true, force: true });
      log.debug({ rootDir }, "Workspace temp dir cleaned up");
    } catch (err) {
      log.warn({ err, rootDir }, "Failed to cleanup workspace temp dir");
    }
  }

  cleanupOrphans(): void {
    try {
      const tmpDir = os.tmpdir();
      const entries = fs.readdirSync(tmpDir);
      let cleaned = 0;
      for (const entry of entries) {
        if (entry.startsWith(TEMP_PREFIX)) {
          const fullPath = path.join(tmpDir, entry);
          try {
            fs.rmSync(fullPath, { recursive: true, force: true });
            cleaned++;
          } catch { /* best effort */ }
        }
      }
      if (cleaned > 0) {
        log.info({ cleaned }, "Cleaned up orphan workspace temp dirs");
      }
    } catch (err) {
      log.warn({ err }, "Failed to scan for orphan workspace dirs");
    }
  }

  static parseWorkspacePaths(input: string): string[] {
    if (!input.includes(",")) return [input.trim()];
    return input.split(",").map(p => p.trim()).filter(Boolean);
  }
}
```

- [ ] **Step 3: Run tests**

Run: `pnpm test src/__tests__/workspace-manager.test.ts`

- [ ] **Step 4: Commit**

```bash
git add src/core/workspace-manager.ts src/__tests__/workspace-manager.test.ts
git commit -m "feat(core): add WorkspaceManager for multi-repo symlink aggregation"
```

---

### Task 2: Types and Session updates

**Files:**
- Modify: `src/core/types.ts`
- Modify: `src/core/session.ts`

- [ ] **Step 1: Add workspaces to SessionRecord**

In `src/core/types.ts`, add `workspaces` field to `SessionRecord`:

```typescript
export interface SessionRecord<P = Record<string, unknown>> {
  // ... existing fields ...
  workspaces?: string[];  // original paths for multi-repo sessions
}
```

- [ ] **Step 2: Add workspaces to Session class**

In `src/core/session.ts`, add property:

```typescript
  workspaces?: string[];
```

- [ ] **Step 3: Verify build**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 4: Commit**

```bash
git add src/core/types.ts src/core/session.ts
git commit -m "feat(types): add workspaces field to SessionRecord and Session"
```

---

### Task 3: Wire into session creation pipeline

**Files:**
- Modify: `src/core/core.ts`
- Modify: `src/core/index.ts`

- [ ] **Step 1: Add WorkspaceManager to core**

Import and add property:

```typescript
import { WorkspaceManager } from "./workspace-manager.js";
```

```typescript
  readonly workspaceManager = new WorkspaceManager();
```

- [ ] **Step 2: Update handleNewSession**

In `handleNewSession()`, update workspace resolution to handle comma-separated paths:

```typescript
  async handleNewSession(
    channelId: string,
    agentName?: string,
    workspacePath?: string,
    options?: { createThread?: boolean },
  ): Promise<Session> {
    const config = this.configManager.get();
    const resolvedAgent = agentName || config.defaultAgent;
    const agentDef = this.agentCatalog.resolve(resolvedAgent);

    // Parse multi-workspace paths
    const rawPath = workspacePath || agentDef?.workingDirectory;
    const paths = rawPath ? WorkspaceManager.parseWorkspacePaths(rawPath) : [];
    const resolvedPaths = paths.map(p => this.configManager.resolveWorkspace(p));

    let workingDirectory: string;
    let workspaces: string[] | undefined;

    if (resolvedPaths.length > 1) {
      const result = this.workspaceManager.create(nanoid(8), resolvedPaths);
      workingDirectory = result.rootDir;
      workspaces = result.workspaces;
    } else {
      workingDirectory = this.configManager.resolveWorkspace(rawPath);
    }

    const session = await this.createSession({
      channelId,
      agentName: resolvedAgent,
      workingDirectory,
      createThread: options?.createThread,
    });

    if (workspaces) {
      session.workspaces = workspaces;
    }

    return session;
  }
```

> **Note:** Check current `handleNewSession` signature and adapt — the above shows the key changes. Also store `workspaces` in the patchRecord call.

- [ ] **Step 3: Persist workspaces in session record**

In `createSession()`, where `patchRecord` is called, add:

```typescript
    if (session.workspaces) {
      platform.workspaces = session.workspaces;
    }
```

Or better, pass it at the top level of patchRecord:

```typescript
    await this.sessionManager.patchRecord(session.id, {
      // ... existing fields ...
      workspaces: session.workspaces,
    });
```

- [ ] **Step 4: Export from index.ts**

```typescript
export { WorkspaceManager } from './workspace-manager.js'
```

- [ ] **Step 5: Verify build**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 6: Commit**

```bash
git add src/core/core.ts src/core/index.ts
git commit -m "feat(core): wire WorkspaceManager into session creation pipeline"
```

---

### Task 4: Cleanup on session end + orphan cleanup

**Files:**
- Modify: `src/core/session-bridge.ts`
- Modify: `src/main.ts`

- [ ] **Step 1: Cleanup temp dir on session end**

In `src/core/session-bridge.ts`, find the `onDisconnect` or status change handler. Add workspace cleanup when session reaches a terminal state:

In the lifecycle handler where status changes to finished/cancelled/error:

```typescript
    // Cleanup multi-workspace temp dir
    if (this.session.workspaces && this.session.workingDirectory.includes("openacp-ws-")) {
      const { WorkspaceManager } = await import("./workspace-manager.js");
      new WorkspaceManager().cleanup(this.session.workingDirectory);
    }
```

> **Note:** Find the exact location where session terminal states are handled. Import WorkspaceManager statically if the module is already imported.

- [ ] **Step 2: Orphan cleanup on startup**

In `src/main.ts`, add cleanup at startup (before adapter registration):

```typescript
import { WorkspaceManager } from "./core/workspace-manager.js";

// Early in main():
new WorkspaceManager().cleanupOrphans();
```

- [ ] **Step 3: Verify build**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 4: Commit**

```bash
git add src/core/session-bridge.ts src/main.ts
git commit -m "feat(core): cleanup multi-workspace temp dirs on session end and startup"
```

---

### Task 5: Command parsing for multi-repo

**Files:**
- Modify: `src/adapters/telegram/commands/new-session.ts`
- Create: `src/__tests__/multi-repo-command.test.ts`

- [ ] **Step 1: Write tests**

Create `src/__tests__/multi-repo-command.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { WorkspaceManager } from "../core/workspace-manager.js";

describe("WorkspaceManager.parseWorkspacePaths", () => {
  it("returns single path unchanged", () => {
    expect(WorkspaceManager.parseWorkspacePaths("~/project")).toEqual(["~/project"]);
  });

  it("splits comma-separated paths", () => {
    expect(WorkspaceManager.parseWorkspacePaths("~/fe,~/be,~/shared")).toEqual(["~/fe", "~/be", "~/shared"]);
  });

  it("trims whitespace", () => {
    expect(WorkspaceManager.parseWorkspacePaths("~/fe , ~/be")).toEqual(["~/fe", "~/be"]);
  });

  it("filters empty strings", () => {
    expect(WorkspaceManager.parseWorkspacePaths("~/fe,,~/be")).toEqual(["~/fe", "~/be"]);
  });
});
```

- [ ] **Step 2: Update Telegram new-session command**

In `src/adapters/telegram/commands/new-session.ts`, find where workspace argument is parsed. Update to pass comma-separated paths through to `handleNewSession`:

The workspace arg should be passed as-is — `handleNewSession` in core.ts now handles the parsing via `WorkspaceManager.parseWorkspacePaths()`.

Update the session topic name for multi-repo:

```typescript
const workspacePaths = WorkspaceManager.parseWorkspacePaths(workspace);
const topicName = workspacePaths.length > 1
  ? `${agentName} — ${workspacePaths.map(p => path.basename(p)).join(" + ")}`
  : `${agentName} — ${workspace}`;
```

- [ ] **Step 3: Run tests**

Run: `pnpm test src/__tests__/multi-repo-command.test.ts`

- [ ] **Step 4: Commit**

```bash
git add src/adapters/telegram/commands/new-session.ts src/__tests__/multi-repo-command.test.ts
git commit -m "feat(telegram): support comma-separated workspace paths in /new command"
```

---

### Task 6: Smoke Test & Final Verification

**Files:** None (verification only)

- [ ] **Step 1: Full type check**

Run: `pnpm exec tsc --noEmit`

- [ ] **Step 2: Run all tests**

Run: `pnpm test`

- [ ] **Step 3: Verify registration**

Grep to confirm:
- `WorkspaceManager` exported from `index.ts`
- `workspaces` field in SessionRecord
- `parseWorkspacePaths` static method
- Orphan cleanup in `main.ts`
- Session bridge cleanup for multi-workspace
- Backward compat: single workspace unchanged

- [ ] **Step 4: Final commit (if any fixups needed)**

```bash
git add -A
git commit -m "chore: final cleanup for multi-repo session feature"
```
