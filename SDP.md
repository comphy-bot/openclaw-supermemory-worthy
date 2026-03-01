# Software Development Plan: openclaw-supermemory-worthy

## Problem
The npm-published `@supermemory/openclaw-supermemory` (v1.0.5) auto-captures **every** `agent_end` event — including cron jobs. With ~20 cron jobs running multiple times daily, supermemory fills with junk entries like "REPLY_SKIP", "HEARTBEAT_OK", "completed successfully".

Upstream `main` has a fix (not yet released to npm) that skips capture for `cron-event` and `exec-event` providers. This fork applies that fix plus a sessionKey-based fallback for OpenClaw versions where `messageProvider` may be undefined.

## Goal
Fork of `@supermemory/openclaw-supermemory` with two additions:
1. **Provider guard** (upstream fix): skip capture for `cron-event` and `exec-event` providers
2. **SessionKey fallback**: skip capture when sessionKey matches cron/isolated patterns (e.g., `agent:*:cron:*`)
3. **ENTITY_CONTEXT**: tell Supermemory cloud to only extract user-side facts, not assistant responses

## Files Changed (vs npm 1.0.5)

### 1. `hooks/capture.ts` — 2 changes

**Change A: Handler signature + provider guard** (from upstream `main`)
The handler accepts `ctx` parameter and skips `exec-event`/`cron-event` providers:

```typescript
// BEFORE (npm 1.0.5):
return async (event: Record<string, unknown>) => {

// AFTER:
return async (
    event: Record<string, unknown>,
    ctx: Record<string, unknown>,
) => {
    log.info(
        `agent_end fired: provider="${ctx.messageProvider}" success=${event.success}`,
    )
    const provider = ctx.messageProvider
    if (provider === "exec-event" || provider === "cron-event") {
        return
    }
```

**Change B: SessionKey fallback** (our addition)
After the provider guard, add a sessionKey-based check for OpenClaw versions where `messageProvider` is undefined:

```typescript
    // Fallback for OpenClaw versions where messageProvider is undefined
    const cronSessionKey = getSessionKey()
    if (cronSessionKey && /^cron:|:cron:|:isolated:/.test(cronSessionKey)) {
        log.info("skipping capture for cron/isolated session: " + cronSessionKey)
        return
    }
```

**Change C: ENTITY_CONTEXT in addMemory call** (from upstream `main`)
Pass `ENTITY_CONTEXT` as 5th argument to `client.addMemory()`:

```typescript
// BEFORE:
await client.addMemory(
    content,
    { source: "openclaw", timestamp: new Date().toISOString() },
    customId,
)

// AFTER:
await client.addMemory(
    content,
    { source: "openclaw", timestamp: new Date().toISOString() },
    customId,
    undefined,
    ENTITY_CONTEXT,
)
```

Also update the import to include `ENTITY_CONTEXT`:
```typescript
import { buildDocumentId, ENTITY_CONTEXT } from "../memory.ts"
```

### 2. `memory.ts` — 1 addition

Add the `ENTITY_CONTEXT` export (from upstream `main`):

```typescript
export const ENTITY_CONTEXT =
    "Messages are tagged with [role: user] and [role: assistant]. Only create memories from what the user actually said — their preferences, decisions, and important personal details. Agent (assistant) responses are just context, not facts to remember. Only remember things that will be useful later. Ignore noise like greetings or status updates."
```

### 3. `client.ts` — 1 change

Add `entityContext` parameter to `addMemory()` (from upstream `main`):

```typescript
// BEFORE:
async addMemory(
    content: string,
    metadata?: Record<string, string | number | boolean>,
    customId?: string,
    containerTag?: string,
): Promise<{ id: string }>

// AFTER:
async addMemory(
    content: string,
    metadata?: Record<string, string | number | boolean>,
    customId?: string,
    containerTag?: string,
    entityContext?: string,
): Promise<{ id: string }>
```

And pass it through in the `client.add()` call:
```typescript
...(entityContext && { entityContext }),
```

### 4. All other files — NO CHANGES
`index.ts`, `config.ts`, `recall.ts`, tools, commands — all identical to upstream.

## Deployment

```bash
# Backup current npm install
mv ~/.openclaw/extensions/openclaw-supermemory ~/.openclaw/extensions/openclaw-supermemory.backup-npm

# Symlink fork
ln -s ~/.openclaw/workspace/clones-repos/openclaw-supermemory-worthy ~/.openclaw/extensions/openclaw-supermemory

# Restart (full stop/start to ensure plugin reload)
openclaw gateway stop && sleep 2 && openclaw gateway start
```

## Verification

```bash
# 1. Check plugin loaded (no errors)
tail -5 ~/.openclaw/logs/gateway.err.log | grep -i supermemory
# Expected: no new errors

# 2. Send a message in Telegram, then check log
grep 'agent_end fired' ~/.openclaw/logs/gateway.log | tail -3
# Expected: provider="telegram" success=true

# 3. Force a cron run
openclaw cron run agentmail-inbox-poll
sleep 15
grep 'agent_end fired\|skipping capture' ~/.openclaw/logs/gateway.log | tail -5
# Expected: provider="cron-event" → early return, OR
#           "skipping capture for cron/isolated session: agent:main:cron:..."
```

## Rollback

```bash
rm ~/.openclaw/extensions/openclaw-supermemory
mv ~/.openclaw/extensions/openclaw-supermemory.backup-npm ~/.openclaw/extensions/openclaw-supermemory
openclaw gateway stop && sleep 2 && openclaw gateway start
```

## Maintenance

To pull upstream changes:
```bash
cd ~/.openclaw/workspace/clones-repos/openclaw-supermemory-worthy
git remote add upstream https://github.com/supermemoryai/openclaw-supermemory.git  # once
git fetch upstream
git merge upstream/main
# Resolve any conflicts in our 3 changed files, commit, push
```

## Deployment Note (2026-03-01)

**Symlinks don't work.** OpenClaw's extension directory scan skips symlink entries.

Instead, add the fork path to `plugins.load.paths` in `~/.openclaw/openclaw.json`:

```json
"plugins": {
  "load": {
    "paths": ["/Users/vatsal/.openclaw/workspace/clones-repos/openclaw-supermemory-worthy"]
  }
}
```

Then restart: `openclaw gateway stop && sleep 2 && openclaw gateway start`

The symlink-based deployment steps in the original SDP are superseded by this approach.
The backup npm install remains at `~/.openclaw/extensions/openclaw-supermemory.backup-npm`.

### Verified Working (2026-03-01 16:05 GMT)
- Plugin loads cleanly (no parse errors)
- Real conversations: `provider="telegram"` → captured normally
- Cron jobs: `provider="undefined"` → sessionKey fallback → `skipping capture for cron/isolated session: agent:main:cron:...`
- Both guards confirmed working
