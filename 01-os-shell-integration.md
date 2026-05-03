# OS & Shell Integration

> Covers §29–35: PTY management, subprocess lifecycle, bash risk classification, background
> processes, OS signals, terminal control, and platform abstractions.
>
> Sources: Gemini CLI (`shell.ts`, `shellBackgroundTools.ts`), Claurst (`bash.rs`, `pty_bash.rs`)

---

## §29 — Shell Execution Architecture

Every production agent needs two shell execution modes: **direct subprocess** (for non-interactive
commands) and **PTY** (for interactive commands, color output, and commands that check `isatty`).

```
User prompt → BashTool
  ├─ Interactive? (vim, ssh, sudo) ── PTY execution ─────────────────┐
  └─ Non-interactive ── Direct subprocess ────────────────────────────┤
                                                                       ↓
                                                            RiskClassifier (pre-flight)
                                                                   ↓
                                                    PermissionLevel (Allow/Deny/Ask)
                                                                   ↓
                                                         spawn_child + StreamHandler
                                                                   ↓
                                                       Chunked output → model context
```

### Direct Subprocess Pattern

```typescript
// TypeScript (Gemini CLI pattern)
interface ShellExecOptions {
  command: string;
  cwd: string;
  timeout?: number;        // ms; default 300_000
  env?: Record<string, string>;
  signal?: AbortSignal;    // for user cancellation
}

interface ShellExecResult {
  stdout: string;
  stderr: string;
  exitCode: number;
  timedOut: boolean;
  signal?: string;         // e.g., "SIGTERM"
}

async function executeShell(opts: ShellExecOptions): Promise<ShellExecResult> {
  const proc = spawn('bash', ['-c', opts.command], {
    cwd: opts.cwd,
    env: { ...process.env, ...opts.env, TERM: 'xterm-256color' },
    detached: false,
  });

  const timeout = opts.timeout ?? 300_000;
  let timedOut = false;

  const timer = setTimeout(() => {
    timedOut = true;
    proc.kill('SIGTERM');
    // escalate to SIGKILL if still running after 5s
    setTimeout(() => { if (!proc.killed) proc.kill('SIGKILL'); }, 5000);
  }, timeout);

  opts.signal?.addEventListener('abort', () => {
    clearTimeout(timer);
    proc.kill('SIGTERM');
  });

  const [stdout, stderr] = await Promise.all([
    collect(proc.stdout),
    collect(proc.stderr),
  ]);

  clearTimeout(timer);
  const exitCode = await waitForExit(proc);

  return { stdout, stderr, exitCode, timedOut };
}
```

```rust
// Rust (Claurst pattern)
pub async fn execute_bash(cmd: &str, ctx: &ToolContext) -> ToolResult {
    let child = tokio::process::Command::new("bash")
        .arg("-c").arg(cmd)
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()?;

    let timeout = ctx.config.bash_timeout;
    match tokio::time::timeout(timeout, collect_output(child)).await {
        Ok(output) => ToolResult::success(format_output(output)),
        Err(_) => ToolResult::error(format!("Command timed out after {}s", timeout.as_secs())),
    }
}
```

---

## §30 — PTY (Pseudo-Terminal) Integration

Use PTY when the command:
- Uses `isatty()` to detect terminal presence (test runners, formatters, `cargo test`)
- Renders with ANSI escape codes that collapse without a PTY
- Requires interactive stdin (`ssh`, `sudo`, TUI applications)

### PTY with xterm headless (TypeScript)

```typescript
import { spawn as ptySpawn } from '@lydell/node-pty';
import { Terminal } from '@xterm/headless';

const OUTPUT_UPDATE_INTERVAL_MS = 1000;

async function executePty(command: string, opts: PtyOptions): Promise<PtyResult> {
  const term = new Terminal({ cols: 220, rows: 50, allowProposedApi: true });
  const pty = ptySpawn('bash', ['-c', command], {
    name: 'xterm-256color',
    cols: 220, rows: 50,
    cwd: opts.cwd,
    env: {
      ...process.env,
      TERM: 'xterm-256color',
      COLUMNS: '220', LINES: '50',
      // disable bash features that produce inconsistent output
    },
  });

  const outputChunks: string[] = [];
  let lastFlush = Date.now();

  pty.onData((data) => {
    term.write(data);
    if (Date.now() - lastFlush > OUTPUT_UPDATE_INTERVAL_MS) {
      const text = serializeTerminal(term);
      opts.onProgress?.(text);
      lastFlush = Date.now();
    }
  });

  await new Promise<void>((resolve) => pty.onExit(() => resolve()));

  return { output: serializeTerminal(term), exitCode: pty.exitCode };
}

// Serialize terminal buffer: handle wrapped lines, strip ANSI codes
function serializeTerminal(term: Terminal): string {
  const lines: string[] = [];
  for (let i = 0; i < term.buffer.active.length; i++) {
    const line = term.buffer.active.getLine(i);
    if (!line) continue;
    const text = line.translateToString(true);  // true = trim trailing whitespace
    // skip empty trailing lines
    if (text || lines.length > 0) lines.push(text);
  }
  return lines.join('\n').trimEnd();
}
```

### PTY (Rust with tokio-pty)

```rust
use tokio_pty_process::{PtyMaster, AsyncPtyMaster, Child};

pub async fn execute_pty(cmd: &str, cwd: &Path) -> ToolResult {
    let master = AsyncPtyMaster::open()?;
    let child = Command::new("bash")
        .arg("-c").arg(cmd)
        .current_dir(cwd)
        .spawn_pty(&master.as_raw_fd())?;

    let mut output = String::new();
    let mut buf = [0u8; 4096];

    loop {
        match tokio::time::timeout(
            Duration::from_millis(100),
            master.read(&mut buf),
        ).await {
            Ok(Ok(n)) if n > 0 => {
                output.push_str(&strip_ansi(&buf[..n]));
            }
            _ => break,
        }
    }

    ToolResult::success(output)
}
```

---

## §31 — Background Process Management

Long-running commands (dev servers, watchers, build jobs) must run in the background and be
tracked across turns.

### Session-Scoped Process Registry

```typescript
// Map: sessionId → Set of background PIDs
const backgroundProcesses = new Map<string, Map<number, BackgroundProcess>>();

interface BackgroundProcess {
  pid: number;
  pgid: number;       // process group — send signals to entire group
  command: string;
  logFile: string;    // writes stdout/stderr here
  startedAt: Date;
}

async function spawnBackground(
  command: string,
  sessionId: string,
  tmpDir: string,
): Promise<BackgroundProcess> {
  const logFile = path.join(tmpDir, `bg-${Date.now()}.log`);
  const logFd = await fs.open(logFile, 'w');

  const proc = spawn('bash', ['-c', command], {
    detached: true,               // create new process group
    stdio: ['ignore', logFd.fd, logFd.fd],
  });
  proc.unref();                   // don't keep Node alive

  const pgid = proc.pid!;         // detached → pid == pgid

  const record: BackgroundProcess = {
    pid: proc.pid!,
    pgid,
    command,
    logFile,
    startedAt: new Date(),
  };

  getSessionProcesses(sessionId).set(proc.pid!, record);
  return record;
}

// Reading background logs — validate path is within allowed tmpDir
async function readBackgroundLog(
  pid: number,
  sessionId: string,
  tmpDir: string,
): Promise<string> {
  const proc = getSessionProcesses(sessionId).get(pid);
  if (!proc) throw new Error(`PID ${pid} not found in session ${sessionId}`);

  // Security: resolve symlinks, verify within tmpDir
  const realPath = await fs.realpath(proc.logFile);
  if (!realPath.startsWith(tmpDir + '/')) {
    throw new Error('Log file outside allowed directory');
  }

  return fs.readFile(proc.logFile, 'utf-8');
}
```

### Termination

```typescript
// Kill entire process group (kills subprocesses too)
function killBackground(proc: BackgroundProcess, signal: 'SIGTERM' | 'SIGKILL' = 'SIGTERM') {
  try {
    process.kill(-proc.pgid, signal);   // negative pgid → entire group
  } catch (e) {
    if ((e as NodeJS.ErrnoException).code !== 'ESRCH') throw e;
    // ESRCH = already dead; safe to ignore
  }
}

// Graceful shutdown: SIGTERM → wait 5s → SIGKILL
async function terminateBackground(proc: BackgroundProcess) {
  killBackground(proc, 'SIGTERM');
  await sleep(5000);
  try { killBackground(proc, 'SIGKILL'); } catch {}
}
```

---

## §32 — Bash Risk Classification

**Pre-execution** classification prevents dangerous commands before they run. This is simpler and
safer than sandboxing because it fails fast without side effects.

### Risk Levels

| Level | Examples | Action |
|-------|----------|--------|
| `Safe` | `ls`, `cat`, `grep`, `pwd` | Execute immediately |
| `Moderate` | `cp`, `mv`, file edits | Execute with confirmation in non-trusted mode |
| `Risky` | `apt install`, `pip install`, `npm install` | Ask user |
| `Dangerous` | `sudo`, `chmod 777`, network modifications | Require explicit allow |
| `Forbidden` | `rm -rf /`, `dd if=/dev/zero`, `:(){ ... }` | Never execute; return error |

### Classifier Implementation

```typescript
interface ClassificationResult {
  level: RiskLevel;
  reason: string;
  matchedPattern?: string;
}

// Order matters: more specific patterns first
const RISK_PATTERNS: Array<{ pattern: RegExp; level: RiskLevel; reason: string }> = [
  // Forbidden: data destruction, fork bombs, rootkit patterns
  {
    pattern: /rm\s+-[rf]+\s+\/(?!tmp|home\/[^/]+\/[^/]+)/,
    level: 'Forbidden',
    reason: 'rm of root filesystem paths',
  },
  { pattern: /:\(\)\s*\{.*\}.*:/, level: 'Forbidden', reason: 'fork bomb pattern' },
  { pattern: /dd\s+.*of=\/dev\/(sda|hda|nvme)/, level: 'Forbidden', reason: 'disk overwrite' },
  { pattern: /mkfs\b/, level: 'Forbidden', reason: 'filesystem format' },

  // Dangerous: privilege escalation, system modification
  { pattern: /\bsudo\b/, level: 'Dangerous', reason: 'privilege escalation' },
  { pattern: /chmod\s+[0-7]*7[0-7]{2}/, level: 'Dangerous', reason: 'world-writable permissions' },
  { pattern: /\b(crontab|at)\b/, level: 'Dangerous', reason: 'scheduled job modification' },
  { pattern: /\/etc\/(passwd|shadow|sudoers)/, level: 'Dangerous', reason: 'system file modification' },

  // Risky: package managers, network, process control
  { pattern: /\b(apt|yum|dnf|brew|pacman)\s+(install|remove|purge)/, level: 'Risky', reason: 'package manager' },
  { pattern: /\bkill\s+-9/, level: 'Risky', reason: 'forced process termination' },
  { pattern: /\bcurl\b.*\|\s*(bash|sh)/, level: 'Risky', reason: 'remote code execution via pipe' },

  // Moderate: file moves/copies, writes
  { pattern: /\b(mv|cp)\b.*\//, level: 'Moderate', reason: 'file system modification' },
];

function classifyBash(command: string): ClassificationResult {
  for (const { pattern, level, reason, ...rest } of RISK_PATTERNS) {
    const match = command.match(pattern);
    if (match) {
      return { level, reason, matchedPattern: match[0] };
    }
  }
  return { level: 'Safe', reason: 'no risk patterns matched' };
}
```

```rust
// Rust pattern (async-safe, no side effects)
pub fn classify_bash(cmd: &str) -> RiskLevel {
    static FORBIDDEN: Lazy<Vec<Regex>> = Lazy::new(|| vec![
        Regex::new(r"rm\s+-[rf]+\s+/(?!tmp)").unwrap(),
        Regex::new(r":\(\)\s*\{.*\}.*:").unwrap(),
    ]);
    static DANGEROUS: Lazy<Vec<Regex>> = Lazy::new(|| vec![
        Regex::new(r"\bsudo\b").unwrap(),
        Regex::new(r"mkfs\b").unwrap(),
    ]);

    for re in FORBIDDEN.iter() { if re.is_match(cmd) { return RiskLevel::Forbidden; } }
    for re in DANGEROUS.iter() { if re.is_match(cmd) { return RiskLevel::Dangerous; } }
    RiskLevel::Safe
}
```

---

## §33 — OS Signal Handling

Agents must handle OS signals cleanly: save sessions, flush logs, terminate child processes.

```typescript
// Clean shutdown on SIGINT (Ctrl+C), SIGTERM (system stop), SIGHUP (terminal close)
function registerShutdownHandlers(session: AgentSession) {
  const signals: NodeJS.Signals[] = ['SIGINT', 'SIGTERM', 'SIGHUP'];
  let shutdownInProgress = false;

  async function handleShutdown(signal: string) {
    if (shutdownInProgress) return;
    shutdownInProgress = true;

    process.stderr.write(`\nReceived ${signal}, shutting down...\n`);

    // 1. Cancel current LLM stream
    session.abortController.abort();

    // 2. Terminate all background processes
    for (const proc of session.backgroundProcesses.values()) {
      await terminateBackground(proc);
    }

    // 3. Save session state
    await session.save();

    // 4. Flush logs
    await logger.flush();

    process.exit(0);
  }

  for (const sig of signals) {
    process.on(sig, () => handleShutdown(sig));
  }

  // Prevent unhandled promise rejections from crashing mid-session
  process.on('unhandledRejection', (reason) => {
    logger.error('Unhandled rejection', { reason });
    // don't exit — let the agent loop handle it
  });
}
```

```rust
// Rust with tokio
use tokio::signal::unix::{signal, SignalKind};

pub async fn run_with_signal_handling(session: Arc<AgentSession>) {
    let mut sigterm = signal(SignalKind::terminate()).unwrap();
    let mut sigint = signal(SignalKind::interrupt()).unwrap();

    let agent = tokio::spawn(session.clone().run());

    tokio::select! {
        _ = sigterm.recv() => { session.shutdown().await; }
        _ = sigint.recv()  => { session.shutdown().await; }
        result = agent     => { handle_result(result); }
    }
}
```

---

## §34 — Environment & Platform Abstractions

### Environment Variable Safety

```typescript
// Sanitize env vars before passing to child processes
function sanitizeEnv(env: NodeJS.ProcessEnv): Record<string, string> {
  const BLOCKED_VARS = new Set([
    'AWS_SECRET_ACCESS_KEY',
    'ANTHROPIC_API_KEY',
    'OPENAI_API_KEY',
    'GOOGLE_APPLICATION_CREDENTIALS',
    'DATABASE_URL',
  ]);

  return Object.fromEntries(
    Object.entries(env)
      .filter(([key]) => !BLOCKED_VARS.has(key))
      .filter(([, val]) => val !== undefined)
      .map(([key, val]) => [key, val as string]),
  );
}

// Identify agent processes in the environment (for nested agent detection)
const AGENT_ENV_MARKERS = [
  'CLAUDE_CODE',
  'GEMINI_CLI',
  'CLAW_CODE',
];

function isRunningInsideAgent(): boolean {
  return AGENT_ENV_MARKERS.some(v => process.env[v] !== undefined);
}
```

### Platform Detection

```typescript
type Platform = 'linux' | 'darwin' | 'windows' | 'unknown';

function getPlatform(): Platform {
  switch (process.platform) {
    case 'linux': return 'linux';
    case 'darwin': return 'darwin';
    case 'win32': return 'windows';
    default: return 'unknown';
  }
}

// Shell selection per platform
function getDefaultShell(): string {
  if (process.platform === 'win32') return 'powershell.exe';
  return process.env.SHELL ?? '/bin/bash';
}
```

---

## §35 — Terminal Output Normalization

When capturing terminal output for injection into LLM context:

1. **Strip ANSI escape codes** — models don't benefit from color codes
2. **Normalize line endings** — convert `\r\n` to `\n`
3. **Collapse empty lines** — remove runs of >2 blank lines
4. **Truncate** — cap at 50 KB to avoid context bloat; keep first + last portions

```typescript
const ANSI_STRIP = /\x1B\[[0-9;]*[a-zA-Z]|\x1B[()]./g;

function normalizeTerminalOutput(raw: string, maxBytes = 50_000): string {
  let text = raw
    .replace(ANSI_STRIP, '')
    .replace(/\r\n/g, '\n')
    .replace(/\r/g, '\n')
    .replace(/\n{3,}/g, '\n\n')
    .trimEnd();

  const bytes = Buffer.byteLength(text);
  if (bytes <= maxBytes) return text;

  // Keep first 20KB and last 30KB with ellipsis in the middle
  const head = text.slice(0, 20_000);
  const tail = text.slice(-30_000);
  return `${head}\n\n[... ${bytes - 50_000} bytes truncated ...]\n\n${tail}`;
}
```

---

## Integration Checklist

- [ ] PTY mode for interactive/color-output commands, direct subprocess for scripts
- [ ] Risk classification **before** execution (not during); `Forbidden` → immediate error
- [ ] Process group tracking (negative PID for kill) to clean up entire tree
- [ ] Background processes scoped to session; log files within validated tmpDir
- [ ] Signal handlers: cancel LLM stream → terminate children → save session → flush logs
- [ ] Sanitize environment variables before subprocess spawn
- [ ] Strip ANSI codes + truncate output before injection into LLM context
- [ ] Graceful SIGTERM → 5s wait → SIGKILL escalation for background processes
- [ ] Detect nested-agent execution via env-marker check
