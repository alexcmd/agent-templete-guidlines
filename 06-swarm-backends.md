# Swarm: Multi-Agent Backend System

> §1–6: Backend abstraction, tmux/iTerm2/in-process execution, file-based IPC mailbox,
> permission synchronization, reconnection, and novel swarm patterns.
>
> Source: Claude Code (`utils/swarm/`, `utils/teammate.ts`, `bridge/`)

---

## §1 — Backend Abstraction

A production swarm system needs to run sub-agents in different execution contexts: terminal
panes (for visibility), separate OS processes (for isolation), or in-process (for speed). The
same agent code should work in all three without modification.

```typescript
// Backend type registry
type BackendType = 'tmux' | 'iterm2' | 'in-process';

// Unified interface all backends implement
interface TeammateExecutor {
  spawn(config: TeammateSpawnConfig): Promise<TeammateHandle>;
  kill(handle: TeammateHandle): Promise<void>;
  isAlive(handle: TeammateHandle): Promise<boolean>;
  sendInput(handle: TeammateHandle, text: string): Promise<void>;
}

interface TeammateSpawnConfig {
  name: string;           // unique agent name in swarm
  prompt: string;         // initial task prompt
  systemPromptMode: 'inherit' | 'custom';
  customSystemPrompt?: string;
  permissionMode: PermissionMode;
  worktreePath?: string;  // isolated git worktree
  model?: string;
  maxTurns?: number;
}

interface TeammateHandle {
  id: string;
  backendType: BackendType;
  paneId?: string;          // tmux/iTerm2
  pid?: number;             // in-process
  mailboxPath: string;      // IPC directory
}
```

### Backend Detection & Selection

```typescript
// Priority: inside-tmux → iTerm2 → tmux standalone → in-process fallback
async function detectAndGetBackend(): Promise<TeammateExecutor> {
  if (await isInsideTmux()) {
    return getTmuxBackend();
  }
  if (await hasITermWithIt2CLI()) {
    return getITermBackend();
  }
  if (await tmuxIsAvailable()) {
    return getTmuxBackend({ standalone: true });
  }
  // Final fallback: same Node.js process with AsyncLocalStorage isolation
  return getInProcessBackend();
}

// Self-registering modules avoid circular imports
// Each backend file executes: registerBackend('tmux', new TmuxBackend())
const backendRegistry = new Map<BackendType, TeammateExecutor>();
export function registerBackend(type: BackendType, executor: TeammateExecutor) {
  backendRegistry.set(type, executor);
}
```

---

## §2 — Tmux Backend

Tmux splits: leader pane + stacked teammate panes.

```typescript
class TmuxBackend implements TeammateExecutor {
  // Prevent race conditions on concurrent spawns
  private spawnLock = new Mutex();

  async spawn(config: TeammateSpawnConfig): Promise<TeammateHandle> {
    return this.spawnLock.runExclusive(async () => {
      const paneId = await this.createPane(config.name);

      // Wait for rc files to load (prevents partial environment)
      await sleep(200);

      // Send the startup command
      const cmd = buildTeammateCommand(config);
      await tmuxSend(paneId, cmd);

      return { id: config.name, backendType: 'tmux', paneId, mailboxPath: getMailboxPath(config.name) };
    });
  }

  private async createPane(name: string): Promise<string> {
    if (this.isInsideLeader) {
      // Split leader's pane: 30% for leader, 70% for teammates stacked below
      return execTmux(`split-window -v -p 70 -d`);
    } else {
      // Standalone swarm session
      return execTmux(`new-window -t claude-swarm:`);
    }
  }

  async kill(handle: TeammateHandle): Promise<void> {
    if (!handle.paneId) return;
    await execTmux(`kill-pane -t ${handle.paneId}`);
  }
}

// Layout for standalone mode (no active leader session)
// Creates: claude-swarm session → swarm-view window → tiled pane layout
async function createStandaloneSwarmSession(names: string[]): Promise<string> {
  await execTmux(`new-session -d -s claude-swarm -n swarm-view`);
  const panes: string[] = [];
  for (const name of names) {
    const pane = await execTmux(`split-window -t claude-swarm:swarm-view -h`);
    panes.push(pane);
  }
  await execTmux(`select-layout -t claude-swarm:swarm-view tiled`);
  return 'claude-swarm:swarm-view';
}
```

---

## §3 — In-Process Backend (AsyncLocalStorage Isolation)

Run teammates in the same Node.js process. Use `AsyncLocalStorage` to provide each teammate
its own context without thread-local variables.

```typescript
import { AsyncLocalStorage } from 'async_hooks';

// Each async call tree gets its own teammate context
const teammateContext = new AsyncLocalStorage<TeammateContext>();

interface TeammateContext {
  agentName: string;
  sessionId: string;
  abortController: AbortController;  // independent lifecycle
  permissionHandler: PermissionHandler;
  costTracker: SharedCostTracker;     // shared with leader
}

async function spawnInProcessTeammate(config: TeammateSpawnConfig): Promise<TeammateHandle> {
  const ctx: TeammateContext = {
    agentName: config.name,
    sessionId: newSessionId(),
    abortController: new AbortController(),  // NOT linked to leader's
    permissionHandler: createTeammatePermissionHandler(config),
    costTracker: leader.costTracker,  // shared budget
  };

  // Run teammate loop inside isolated context
  const task = teammateContext.run(ctx, async () => {
    await teammateMainLoop(config.prompt, ctx);
  });

  return {
    id: config.name,
    backendType: 'in-process',
    pid: process.pid,
    mailboxPath: getMailboxPath(config.name),
    task,
  };
}

// Access current teammate's context from anywhere in the call stack
function getCurrentTeammateContext(): TeammateContext | undefined {
  return teammateContext.getStore();
}
```

### In-Process Teammate Main Loop

```typescript
async function teammateMainLoop(initialPrompt: string, ctx: TeammateContext) {
  const mailbox = new TeammateMailbox(ctx.agentName);

  // Process initial prompt
  await runAgentTurn(initialPrompt, ctx);
  await mailbox.notifyIdle({ reason: 'completed initial task' });

  // Poll for more work
  while (!ctx.abortController.signal.aborted) {
    const message = await mailbox.waitForMessage({ timeoutMs: 30_000 });
    if (!message) continue;

    if (isShutdownRequest(message)) {
      // Let model decide whether to accept shutdown
      const shouldAccept = await askModelAboutShutdown(message.summary, ctx);
      await mailbox.sendShutdownResponse(shouldAccept);
      if (shouldAccept) break;
      continue;
    }

    await runAgentTurn(message.text, ctx);
    await mailbox.notifyIdle({ reason: 'completed task', completedTaskId: message.taskId });
  }
}
```

---

## §4 — File-Based IPC Mailbox

The mailbox is the universal IPC layer — works the same for tmux, iTerm2, and in-process backends.

```typescript
// Directory structure: ~/.claude/teams/{teamName}/{agentName}/
// ├── inbox/          ← leader writes, teammate reads
// ├── outbox/         ← teammate writes, leader reads
// ├── pending/        ← permission requests from teammate
// ├── resolved/       ← permission responses from leader
// └── .lock           ← directory-level lockfile

interface MailboxMessage {
  id: string;           // UUID for deduplication
  from: string;
  text: string;
  timestamp: number;
  color?: string;       // terminal color for display
  taskId?: string;      // task tracker reference
  summary?: string;     // for shutdown request approval
}

class TeammateMailbox {
  private lockfile: Lockfile;

  constructor(
    private agentName: string,
    private teamName: string,
  ) {
    this.lockfile = new Lockfile(this.mailboxDir);
  }

  // Leader → Teammate
  async send(message: Omit<MailboxMessage, 'id' | 'timestamp'>): Promise<void> {
    const full: MailboxMessage = {
      ...message,
      id: uuid(),
      timestamp: Date.now(),
    };
    await this.lockfile.acquire();
    try {
      const file = path.join(this.mailboxDir, 'inbox', `${full.id}.json`);
      await fs.writeFile(file, JSON.stringify(full), { flag: 'wx' }); // fail if exists
    } finally {
      this.lockfile.release();
    }
  }

  // Teammate polls inbox
  async waitForMessage(opts: { timeoutMs: number }): Promise<MailboxMessage | null> {
    const deadline = Date.now() + opts.timeoutMs;
    while (Date.now() < deadline) {
      const messages = await this.readInbox();
      if (messages.length > 0) {
        const [first] = messages.sort((a, b) => a.timestamp - b.timestamp);
        await this.deleteMessage(first.id);
        return first;
      }
      await sleep(500);
    }
    return null;
  }

  // Teammate → Leader idle notification
  async notifyIdle(status: IdleStatus): Promise<void> {
    const file = path.join(this.mailboxDir, 'outbox', `idle-${Date.now()}.json`);
    await fs.writeFile(file, JSON.stringify(status));
  }
}
```

---

## §5 — Permission Synchronization

When a teammate needs permission for a tool, it must ask the leader (who owns the user-facing
prompt). This works the same for all backends.

```typescript
// Leader registers handlers during its own init
function registerLeaderPermissionHandlers(
  confirmQueue: ToolUseConfirmQueue,
  setPermissionContext: SetPermissionContextFn,
) {
  leaderConfirmQueue = confirmQueue;
  leaderSetPermissionContext = setPermissionContext;
}

// Teammate's permission handler — routes to leader
async function teammatePermissionHandler(
  toolName: string,
  input: unknown,
  suggestions: string[],
): Promise<PermissionDecision> {
  // Path 1: In-process → use leader's direct queue
  if (leaderConfirmQueue) {
    return leaderConfirmQueue.push({ toolName, input, suggestions, workerName });
  }

  // Path 2: Pane-based → file-based permission IPC
  return fileBasedPermissionRequest({ toolName, input, suggestions });
}

async function fileBasedPermissionRequest(req: PermissionRequest): Promise<PermissionDecision> {
  const reqFile = path.join(mailboxDir, 'pending', `${req.id}.json`);
  await fs.writeFile(reqFile, JSON.stringify(req));

  // Poll for resolution
  const resFile = path.join(mailboxDir, 'resolved', `${req.id}.json`);
  while (true) {
    try {
      const data = await fs.readFile(resFile, 'utf-8');
      await fs.unlink(resFile);  // consume
      return JSON.parse(data) as PermissionDecision;
    } catch (e) {
      if ((e as NodeJS.ErrnoException).code !== 'ENOENT') throw e;
      await sleep(500);
    }
  }
}

// Cleanup stale resolved files (>1h old)
async function cleanupStalePermissions(mailboxDir: string): Promise<void> {
  const resolvedDir = path.join(mailboxDir, 'resolved');
  const files = await fs.readdir(resolvedDir);
  const cutoff = Date.now() - 3_600_000;
  for (const f of files) {
    const stat = await fs.stat(path.join(resolvedDir, f));
    if (stat.mtimeMs < cutoff) await fs.unlink(path.join(resolvedDir, f));
  }
}
```

---

## §6 — Reconnection & Team Discovery

```typescript
// Team file is the source of truth for swarm state
interface TeamFile {
  id: string;           // session UUID
  leadAgentId: string;
  members: TeamMember[];
  isActive: boolean;
  createdAt: number;
}

interface TeamMember {
  name: string;
  agentName: string;
  backendType: BackendType;
  paneId?: string;
  worktreePath?: string;
  isActive: boolean;
  lastSeenAt: number;
}

// Reconnect to an existing swarm on REPL resume
async function reconnectToSwarm(sessionId: string): Promise<SwarmContext | null> {
  const teamFile = await loadTeamFile(sessionId);
  if (!teamFile) return null;

  const members: ReconnectedTeammate[] = [];
  for (const member of teamFile.members.filter(m => m.isActive)) {
    const isAlive = await checkTeammateAlive(member);
    if (isAlive) {
      members.push({ ...member, mailbox: new TeammateMailbox(member.agentName) });
    } else {
      // Prune dead teammates from file
      await markTeammateDead(teamFile, member.name);
    }
  }

  return { teamFile, members };
}

// Get current status of all teammates (for status display)
async function getTeammateStatuses(teamName: string): Promise<TeammateStatus[]> {
  const teamFile = await loadTeamFile(teamName);
  return teamFile.members.map(member => ({
    name: member.name,
    isActive: member.isActive,
    lastSeenAt: member.lastSeenAt,
    backendType: member.backendType,
    paneId: member.paneId,
    worktreePath: member.worktreePath,
  }));
}
```

---

## §7 — Novel Swarm Patterns

### Per-Agent Worktree Isolation

Each teammate gets its own git worktree, preventing file conflicts when working on the same repo.

```typescript
// Spawn teammate with isolated worktree
async function spawnWithWorktree(config: TeammateSpawnConfig): Promise<TeammateHandle> {
  const worktreePath = await createWorktree({
    branch: `swarm/${config.name}`,
    baseBranch: 'main',
    sparsePaths: config.focusPaths,  // optional: only checkout relevant files
  });

  const handle = await backend.spawn({
    ...config,
    worktreePath,
    env: { GIT_WORK_TREE: worktreePath },
  });

  handle.cleanupWorktree = () => removeWorktree(worktreePath);
  return handle;
}
```

### Task Auto-Claim

In-process teammates can auto-claim unclaimed tasks from a shared task list.

```typescript
async function claimNextTask(teamName: string, agentName: string): Promise<Task | null> {
  const tasks = await loadTeamTaskList(teamName);
  const unclaimed = tasks.filter(t => t.status === 'pending' && !t.assignee);

  if (unclaimed.length === 0) return null;

  // Atomic claim: write assignee + optimistic lock on task version
  const task = unclaimed[0];
  const claimed = await claimTask(task.id, agentName, task.version);
  return claimed;
}
```

### Bounded Echo Deduplication (Bridge)

Prevents re-delivering messages when server replays history.

```typescript
class BoundedUUIDSet {
  private capacity: number;
  private set = new Set<string>();
  private queue: string[] = [];  // FIFO order for eviction

  constructor(capacity = 1000) { this.capacity = capacity; }

  add(id: string): boolean {
    if (this.set.has(id)) return false;  // already seen
    if (this.queue.length >= this.capacity) {
      const evicted = this.queue.shift()!;
      this.set.delete(evicted);
    }
    this.set.add(id);
    this.queue.push(id);
    return true;  // new, process it
  }
}
```

---

## Integration Checklist

- [ ] Implement `TeammateExecutor` interface for each backend (tmux, iTerm2, in-process)
- [ ] Self-register backends via side-effect imports to avoid circular dependencies
- [ ] Use `AsyncLocalStorage` for in-process teammate context isolation
- [ ] File-based mailbox with directory-level lock for atomic multi-agent writes
- [ ] Spawn lock (Mutex) in tmux/iTerm2 to prevent race conditions on concurrent spawns
- [ ] Leader permission bridge: in-process uses direct queue; pane-based uses file IPC
- [ ] Clean up stale permission files (>1h TTL)
- [ ] Per-agent git worktrees with `--no-checkout` + sparse paths for large repos
- [ ] `BoundedUUIDSet` for message deduplication on history replay
- [ ] Team file as authoritative swarm state; prune dead members on reconnect
- [ ] Independent `AbortController` per teammate (not linked to leader lifecycle)
- [ ] 200ms init delay after pane creation for shell rc file loading
