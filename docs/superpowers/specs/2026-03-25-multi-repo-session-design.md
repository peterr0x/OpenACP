# Multi-Repo Session — Design Spec

## Summary

Allow a single session to work across multiple repositories/directories simultaneously. Agent sees all repos under a unified virtual workspace via symlinks. Changes in one repo that affect another (e.g., shared types, API contracts) can be handled in a single conversation.

## Problem

Modern projects span multiple repos: frontend + backend + shared libs. Currently, each session can only access one directory. To fix a cross-repo issue, users must open separate sessions, manually coordinate changes, and copy-paste context between them. This is slow, error-prone, and defeats the purpose of AI automation.

## Requirements

- **Multi-workspace**: `/new claude ~/frontend,~/backend,~/shared` — comma-separated paths
- **Unified access**: Agent sees all repos under one root, can read/write files in any of them
- **Symlink approach**: Create a temp directory with symlinks to each repo — transparent to agent
- **Primary workspace**: First path is the `cwd` passed to agent — agent starts there
- **Cleanup**: Temp symlink directory removed when session ends
- **Backward compatible**: Single workspace still works exactly as before
- **Persisted**: Multi-workspace paths stored in session record for resume/display

## Non-Goals

- Cross-repo git operations (agent can do this naturally via CLI)
- Automatic dependency detection between repos
- Workspace templates (future feature)
- Remote repos (must be local paths)

## Design

### Approach: Symlink Aggregation

```
User: /new claude ~/frontend,~/backend,~/shared

OpenACP creates:
  /tmp/openacp-ws-{sessionId}/
    ├── frontend → ~/frontend        (symlink)
    ├── backend  → ~/backend          (symlink)
    └── shared   → ~/shared           (symlink)

Agent spawns with cwd = /tmp/openacp-ws-{sessionId}/
Agent sees 3 directories, can cd into any of them.
```

Why symlinks:
- Transparent to agent — no special protocol changes needed
- ACP `cwd` stays a single string (no protocol breaking change)
- Agent can use standard file operations across all repos
- No data duplication — symlinks are lightweight

### Command Syntax

```
/new claude ~/frontend,~/backend         → 2 repos
/new claude ~/project                    → single repo (unchanged)
/new claude ~/fe,~/be,~/shared,~/docs    → 4 repos
```

CLI:
```bash
openacp session new --agent claude --workspace ~/frontend,~/backend
```

### Session Creation Flow

```
1. Parse workspace arg → detect comma → split into paths[]
2. Resolve each path (expandHome, validate exists)
3. If multiple:
   a. Create temp dir: /tmp/openacp-ws-{sessionId}/
   b. Symlink each path: ln -s ~/frontend /tmp/.../frontend
   c. Set session.workingDirectory = temp dir
   d. Store original paths in session metadata
4. Spawn agent with cwd = resolved working directory
5. On session end: cleanup temp dir (rm -rf symlinks, not targets)
```

### Data Model

```typescript
// SessionRecord — backward compatible
interface SessionRecord {
  workingDir: string;           // resolved path (temp dir for multi-repo)
  workspaces?: string[];        // original paths (only set for multi-repo)
}

// Session class
class Session {
  workingDirectory: string;     // agent's cwd (temp dir for multi-repo)
  workspaces?: string[];        // original user-provided paths
}
```

### Display

Session topic title:
```
🔄 claude — frontend + backend + shared
```

Status/info display:
```
📁 Workspaces:
  • ~/frontend
  • ~/backend
  • ~/shared
```

### Cleanup

On session end (finish/cancel/error/archive):
```typescript
if (session.workspaces && session.workingDirectory.startsWith("/tmp/openacp-ws-")) {
  fs.rmSync(session.workingDirectory, { recursive: true, force: true });
}
```

Only removes symlinks, NOT the actual repo directories. The `startsWith("/tmp/openacp-ws-")` guard prevents accidental deletion.

### Error Handling

| Scenario | Behavior |
|----------|----------|
| Path doesn't exist | Reject with "Directory not found: ~/invalid" |
| Symlink creation fails (permissions) | Reject with clear error message |
| Duplicate repo names | Append suffix: `backend`, `backend-2` |
| Temp dir already exists | Generate new unique ID |
| Session ends without cleanup | Temp dirs cleaned on startup (scan `/tmp/openacp-ws-*`) |

### Affected Components

**Core layer** (modify):
- `src/core/config.ts` — `resolveWorkspace()` handle comma-separated input
- `src/core/session.ts` — add `workspaces?: string[]` property
- `src/core/session-factory.ts` — create symlink dir for multi-workspace
- `src/core/core.ts` — pass workspaces through creation pipeline
- `src/core/types.ts` — add `workspaces?: string[]` to SessionRecord

**Core layer** (new):
- `src/core/workspace-manager.ts` — symlink creation, cleanup, path resolution

**Adapter layer** (modify):
- `src/adapters/telegram/commands/new-session.ts` — parse comma-separated paths
- `src/adapters/discord/commands/new-session.ts` — same
- Session display: show multiple workspace paths

**Cleanup**:
- `src/core/session-bridge.ts` — cleanup temp dir on session end
- `src/main.ts` — cleanup orphan temp dirs on startup
