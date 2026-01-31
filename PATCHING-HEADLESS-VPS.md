# Patching happy-cli for Headless Linux VPS (Daemon Mode)

## Problem Summary

When running `happy daemon start` on a headless Linux VPS (no TTY), sessions spawn
and connect to the server, but **user messages from the phone never arrive**. Sessions
get stuck at `MessageQueue2 Waiting for messages...`.

Observed behavior:
- Sessions spawn and create socket connections successfully
- Socket receives `update-session` events (metadata updates from CLI itself)
- Socket does NOT receive `new-message` events (user messages from phone)
- Same server + phone work fine when `happy` runs interactively on macOS

## Architecture: Message Flow

```
Phone App                    Happy Server                  VPS (happy daemon)
   │                              │                              │
   │  "Fix the bug"              │                              │
   │─────────────────────────────>│                              │
   │                              │  spawn-happy-session RPC     │
   │                              │─────────────────────────────>│  daemon/run.ts
   │                              │                              │
   │                              │                              │  spawns:
   │                              │                              │  node dist/index.mjs claude
   │                              │                              │    --happy-starting-mode remote
   │                              │                              │    --started-by daemon
   │                              │                              │
   │                              │    POST /v1/sessions         │  runClaude.ts:104
   │                              │<─────────────────────────────│  creates NEW session (id=Y)
   │                              │                              │
   │                              │    socket.connect(sid=Y)     │  apiSession.ts:82
   │                              │<═════════════════════════════│  WebSocket connected
   │                              │                              │
   │                              │    update-session (self)     │
   │                              │════════════════════════════=>│  ✓ WORKS (own metadata)
   │                              │                              │
   │  "Fix the bug" (to sid=???) │                              │
   │─────────────────────────────>│    new-message               │
   │                              │════════════════════════════=>│  ✗ NEVER ARRIVES
   │                              │                              │
   │                              │                              │  MessageQueue2:
   │                              │                              │  "Waiting for messages..."
```

The question mark: does the phone send to the session the CLI created (Y), or to
a different session?

## Root Cause Analysis

### Theory 1: Session Routing (Most Likely)

The daemon receives a `sessionId` from the server but **ignores it**:

```typescript
// daemon/run.ts:498-499
// TODO: In future, sessionId could be used with --resume to continue existing sessions
// For now, we ignore it - each spawn creates a new session
```

The spawned CLI creates a brand new session with `getOrCreateSession()` (api.ts:30),
which generates **new encryption keys** and gets a **new session ID**. If the phone
targets the server-created session and the CLI creates a different one, messages are
routed to the wrong session.

**However** — this may actually work if the server/phone auto-discovers the CLI's new
session. The daemon webhook system (run.ts:177-216) reports the CLI's session ID back
to the server. If the phone UI shows the new session and the user sends to it, routing
should work.

### Theory 2: Socket Room Membership

The CLI socket connects with `clientType: 'session-scoped'` and `sessionId: Y`. The
server joins this socket to a room for session Y. Phone messages targeting session Y
should be routed to that room.

But maybe the server has separate routing logic for `new-message` vs `update-session`
events. The CLI sees `update-session` (which are self-generated metadata updates), but
`new-message` events from the phone might require different room membership or routing
that headless mode doesn't satisfy.

### Theory 3: Phone Never Discovers CLI Session

The phone might be sending messages to a server-created "pending" session that the
daemon was supposed to join (but didn't, because it created its own). The phone never
switches to the CLI's new session.

### Known Non-Root-Cause Issues

**TTY check** (`claudeRemoteLauncher.ts:30`):
```typescript
const hasTTY = process.stdout.isTTY && process.stdin.isTTY;
// Returns `undefined` on headless Linux (not `false`)
```
This only affects Ink UI rendering and stdin raw mode — the remote message flow
(`queue → nextMessage → claudeRemote`) doesn't depend on TTY.

**childStdin null** (`sdk/query.ts:176`): Affects tool permission handling when Claude
tries to use tools, but doesn't affect initial message delivery.

## Patch Plan

### START HERE: Patch 1 — Diagnostic Logging

Deploy this first to identify the exact failure point before writing the fix.

**File: `src/api/apiSession.ts`**

Add at the top of the `update` handler (line ~123):

```typescript
this.socket.on('update', (data: Update) => {
    try {
        // DIAGNOSTIC: Log ALL incoming socket events
        console.error(`[DIAG] Socket event: body.t=${data.body?.t}, sid=${this.sessionId}`);
```

Add inside the `new-message` branch (line ~132):

```typescript
if (data.body.t === 'new-message' && data.body.message.content.t === 'encrypted') {
    console.error(`[DIAG] new-message received! Attempting decrypt...`);
    const body = decrypt(this.encryptionKey, this.encryptionVariant, decodeBase64(data.body.message.content.c));
    console.error(`[DIAG] Decrypted. Parsing as UserMessage...`);
    const userResult = UserMessageSchema.safeParse(body);
    console.error(`[DIAG] Parse success=${userResult.success}, hasCallback=${!!this.pendingMessageCallback}`);
```

Add a catch-all for events NOT matching known types, after the else-if chain (line ~163):

```typescript
} else {
    console.error(`[DIAG] Unknown event type: ${data.body?.t}`);
    this.emit('message', data.body);
}
```

**File: `src/claude/runClaude.ts`**

Add at line ~250 inside the onUserMessage callback:

```typescript
session.onUserMessage((message) => {
    console.error(`[DIAG] onUserMessage fired! text="${message.content?.text?.substring(0, 80)}"`);
```

**File: `src/claude/claudeRemoteLauncher.ts`**

Add at line ~346 inside the nextMessage callback:

```typescript
nextMessage: async () => {
    console.error(`[DIAG] nextMessage() called. pending=${!!pending}, queueSize=${session.queue.size()}`);
```

Also add at line ~305 when the remote launcher starts:

```typescript
logger.debug('[remote]: launch');
console.error(`[DIAG] claudeRemoteLauncher: starting. sessionId=${session.sessionId}`);
```

**File: `src/daemon/run.ts`**

Add after spawn (line ~528):

```typescript
logger.debug(`[DAEMON RUN] Spawned process with PID ${happyProcess.pid}`);
console.error(`[DIAG] Daemon spawned session. ignored-sessionId=${sessionId}, pid=${happyProcess.pid}`);
```

#### Deploy & Test

```bash
# Build
cd packages/happy-cli
npm run build

# On VPS, restart daemon
happy daemon stop
DEBUG=1 happy daemon start-sync 2>&1 | tee /tmp/happy-diag.log
```

Send a message from phone. Check `/tmp/happy-diag.log`:

| What you see | Meaning |
|---|---|
| `Socket event: body.t=update-session` only | Socket works but phone targets wrong session |
| `Socket event: body.t=new-message` then `Parse success=false` | Message arrives but schema mismatch or decryption issue |
| `Socket event: body.t=new-message` then `Parse success=true` then `hasCallback=false` | `onUserMessage()` wasn't called in time — race condition |
| `Socket event: body.t=new-message` then `onUserMessage fired!` | Full flow works — issue is downstream |
| No `Socket event` lines at all | Socket didn't connect or handler not registered |

### Patch 2: Fix TTY Boolean (Quick Win)

**File: `src/claude/claudeRemoteLauncher.ts`, line 30:**

```diff
- const hasTTY = process.stdout.isTTY && process.stdin.isTTY;
+ const hasTTY = !!(process.stdout.isTTY && process.stdin.isTTY);
```

### Patch 3: Fix Silent Control Request Drop

**File: `src/claude/sdk/query.ts`, line ~175-179:**

```diff
  private async handleControlRequest(request: CanUseToolControlRequest): Promise<void> {
      if (!this.childStdin) {
-         logDebug('Cannot handle control request - no stdin available')
-         return
+         logDebug('Cannot handle control request - no stdin available')
+         throw new Error('No stdin available - headless mode cannot handle interactive tool permissions')
      }
```

This makes the error visible instead of silently dropping permission requests.

### Patch 4: Session ID Passthrough (If Diagnostics Confirm Theory 1)

Only apply this after Patch 1 diagnostics confirm session ID mismatch.

**This is complex** because `getOrCreateSession()` generates new encryption keys
each time (api.ts:42-56). You can't simply pass a session ID — the CLI needs the
same encryption context as the phone/server session.

Two approaches:

#### Approach A: Server-side join endpoint

Add a `GET /v1/sessions/:id` endpoint to happy-server that returns an existing
session for the CLI to join. The CLI would call this instead of creating a new one.

This requires patching **both** happy-server and happy-cli.

#### Approach B: Pass tag from daemon

Instead of creating a new random `tag` (runClaude.ts:51), pass the server's
`sessionId` as the tag to `getOrCreateSession()`. If the server uses tags to
deduplicate, this might return the existing session.

**File: `src/daemon/run.ts` (tmux path, line ~391):**

```diff
- const fullCommand = `node ... ${agent} --happy-starting-mode remote --started-by daemon`;
+ const sidArg = sessionId ? ` --happy-session-tag ${sessionId}` : '';
+ const fullCommand = `node ... ${agent} --happy-starting-mode remote --started-by daemon${sidArg}`;
```

**File: `src/daemon/run.ts` (regular path, line ~492-496):**

```diff
  const args = [
      agentCommand,
      '--happy-starting-mode', 'remote',
-     '--started-by', 'daemon'
+     '--started-by', 'daemon',
+     ...(sessionId ? ['--happy-session-tag', sessionId] : [])
  ];
```

**File: `src/index.ts` (arg parsing, after line ~496):**

```typescript
} else if (arg === '--happy-session-tag') {
    options.sessionTag = args[++i]
```

**File: `src/claude/runClaude.ts` (line ~51 and ~104):**

```diff
- const sessionTag = randomUUID();
+ const sessionTag = options.sessionTag || randomUUID();
```

#### Approach C: Direct env var passthrough

Pass the session ID as an environment variable that the server recognizes:

```diff
// daemon/run.ts, in the env block
env: {
    ...process.env,
    ...extraEnv,
+   HAPPY_SESSION_ID: sessionId || ''
}
```

Then in `runClaude.ts`, check for this env var and use it as the tag.

**Recommendation:** Start with Approach B (tag passthrough) as it requires the
fewest changes and doesn't need server modifications.

## File Reference

| File | Line(s) | What it does | Patch |
|------|---------|-------------|-------|
| `src/api/apiSession.ts` | 82-96 | Socket connection with session-scoped auth | 1: Diag logs |
| `src/api/apiSession.ts` | 123-169 | Socket `update` event handler (routes new-message) | 1: Diag logs |
| `src/api/apiSession.ts` | 183-188 | `onUserMessage()` sets callback, drains pending | 1: Diag logs |
| `src/api/api.ts` | 30-89 | `getOrCreateSession()` — creates session + encryption | 4: Session reuse |
| `src/claude/runClaude.ts` | 51 | `sessionTag = randomUUID()` | 4: Use daemon tag |
| `src/claude/runClaude.ts` | 104 | `api.getOrCreateSession({tag: sessionTag, ...})` | 4: Pass tag |
| `src/claude/runClaude.ts` | 250 | `session.onUserMessage()` callback → queue | 1: Diag logs |
| `src/claude/claudeRemoteLauncher.ts` | 30 | TTY check | 2: Boolean fix |
| `src/claude/claudeRemoteLauncher.ts` | 346 | `nextMessage()` reads from queue | 1: Diag logs |
| `src/claude/claudeRemote.ts` | 85 | Gets first message from nextMessage() | Read-only |
| `src/claude/loop.ts` | 49-67 | Creates Session, dispatches launcher | Read-only |
| `src/claude/sdk/query.ts` | 175-179 | childStdin null → silent drop | 3: Throw error |
| `src/daemon/run.ts` | 218-222 | Spawn options (receives sessionId from server) | 4: Pass through |
| `src/daemon/run.ts` | 391 | tmux spawn command | 4: Add session arg |
| `src/daemon/run.ts` | 492-496 | Regular spawn args | 4: Add session arg |
| `src/daemon/run.ts` | 498-499 | TODO comment about sessionId | 4: Implement |
| `src/index.ts` | 490-496 | CLI arg parsing | 4: Parse new arg |

## Testing Order

1. Apply Patches 1 + 2 + 3 (diagnostics + TTY fix + childStdin fix)
2. Deploy to VPS, run daemon with `DEBUG=1`
3. Send message from phone, read `/tmp/happy-diag.log`
4. Based on diagnostic output, apply Patch 4 if needed
5. If Patch 4 doesn't resolve, investigate happy-server routing
