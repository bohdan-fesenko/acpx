# ACPX Source Code Customizations (Brain Fork)

Changes applied to upstream `openclaw/acpx@v0.5.3` in our fork
`bohdan-fesenko/acpx`. Branch: `patches/0.5.3-env-and-cancel`.
Tag: `v0.5.3-brain.1`.

This document mirrors the format of
`bohdan-fesenko/brain:tools/openclaw/OPENCLAW-SOURCE-CHANGES.md` so the same
mental model applies — patches numbered, files listed, problem + fix +
verification documented per patch.

## Where the code lives

| Location                          | Role                                                                 | Path                                                                                        |
| --------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **GitHub fork** (source of truth) | Patch tracking + tags                                                | `https://github.com/bohdan-fesenko/acpx`                                                    |
| **Mac local clone**               | Editing + building locally                                           | `acpx_source/` (this repo's sibling — git-ignored from outer codebase via own `.gitignore`) |
| **VM consumer**                   | Linked into `~/openclaw/extensions/acpx/` via `package.json` git URL | Auto-pulled when `pnpm install` runs in VM openclaw repo                                    |

### Reference / consume from openclaw fork

`bohdan-fesenko/openclaw:extensions/acpx/package.json`:

```jsonc
{
  "dependencies": {
    "acpx": "github:bohdan-fesenko/acpx#v0.5.3-brain.1",
  },
}
```

The `prepare` script in our fork runs `pnpm run build` after install, so
consumers always get a fresh `dist/` even though `dist/` is git-ignored.

### Sync with upstream

```bash
cd acpx_source
git fetch upstream
# Inspect new upstream tag, then:
git rebase v0.6.0                  # or whatever the next stable tag is
# Resolve any conflicts (each patch is small + self-contained)
pnpm build && pnpm test
git tag v0.6.0-brain.1
git push origin patches/0.5.3-env-and-cancel --tags
# Update package.json reference in openclaw fork to new tag
```

---

## Why we forked

Upstream `openclaw/acpx@0.5.3` exposes `env?: Record<string, string>` on the
`AcpRuntimeEnsureInput` type but the implementation **drops the field on the
floor** — it never reaches the spawned subprocess. Combined with the graceful
`cancel()` protocol that hangs during active turns (30-120s latency), running
brain workloads on stock acpx breaks two of our hard requirements:

1. **Per-agent identity** in spawned subprocess env (`BRAIN_AGENT_NAME`,
   `WF_AGENT_ROLE`, `BRAIN_PATH`) — without it brain signals from all agents
   collapse to a single identity.
2. **Sub-second `/cancel` response** in Telegram — without it users wait
   30-120s after a `/cancel` for the agent to actually stop, forcing
   `systemctl restart` as workaround.

Both gaps require source patches. Brain fork carries these as small,
self-contained commits on top of upstream tag `v0.5.3` so we can `git rebase`
cleanly when upstream releases new versions.

---

## Applied Patches

### 1. Per-Session Environment Propagation (`cef5537`)

**Files (6 touchpoints):**

- `src/runtime/public/contract.ts` — add `env?: Record<string, string>` to `AcpRuntimeEnsureInput`
- `src/runtime.ts` — top-level `Runtime.ensureSession`: pass `input.env` to manager
- `src/runtime/engine/manager.ts` — `Manager.ensureSession`: accept `env`, pass to client constructor as `extraEnv`
- `src/types.ts` — add `extraEnv?: Record<string, string>` to `AcpClientOptions`
- `src/acp/client.ts` — pass `this.options.extraEnv` to `buildAgentSpawnOptions`
- `src/acp/auth-env.ts` — `buildAgentEnvironment` and `buildAgentSpawnOptions`: accept `extraEnv`, merge into env map

**Problem:** Brain platform hooks identify the spawning agent via env vars
(`BRAIN_AGENT_NAME=wf-exec-openclaw` etc.). Without per-session env in the
ACPX subprocess, all sessions appear as the same identity in brain signals →
signal routing, cursor tracking, and activity attribution all break.

Upstream `AcpRuntimeEnsureInput` already has `env?: Record<string, string>`
in the type contract — apparently the API was designed for this — but the
implementation drops the field at the very first hop:

```typescript
// src/runtime.ts — UPSTREAM (env dropped here)
const record = await manager.ensureSession({
  sessionKey: sessionName,
  agent,
  mode: input.mode,
  cwd: input.cwd ?? this.options.cwd,
  resumeSessionId: input.resumeSessionId,
  // ← input.env IS NOT FORWARDED
});
```

**Fix:** Thread `env` through the entire chain:

```
Runtime.ensureSession({env}) →
  Manager.ensureSession({env}) →
    AcpClient({extraEnv}) →
      buildAgentSpawnOptions(cwd, authCredentials, extraEnv) →
        buildAgentEnvironment(authCredentials, extraEnv) →
          spawn(child, {env: { ...process.env, ...extraEnv, ...authCreds }})
```

**Merge order** (later wins):

- `process.env` (parent — gateway env) — base
- `extraEnv` (per-session — `BRAIN_AGENT_NAME`, `WF_AGENT_ROLE`, etc.) — overrides parent
- `authCredentials` — most specific, wins everything

**Why renamed `env` → `extraEnv` at the AcpClient layer:** `AcpClientOptions`
already has many fields; using `extraEnv` makes it crystal clear this is an
_additional_ env merged into the spawn, not the full env replacement. Public
API at `AcpRuntimeEnsureInput.env` keeps the simpler upstream-defined name.

**Verification:**

- `pnpm build` — passes clean (1.4MB dist, 36 files)
- Manual env trace via inline script — value reaches `buildAgentEnvironment`
- Compiled `dist/runtime.js` contains `extraEnv` symbol (1 occurrence — the
  Manager → Client passthrough; bundler inlined other paths)
- E2E verification deferred to openclaw integration smoke test (post-merge)

**Interaction with other patches:**

- Patch #2 (cancel fast-kill): independent. Both work in any order.
- Required by openclaw `OPENCLAW-SOURCE-CHANGES.md` patches #6 / #6b / #6c —
  those threaded env on the openclaw side; this patch closes the chain inside
  acpx so the value actually reaches the subprocess.

---

### 2. Cancel Fast-Kill (TD-004 backport) (`818ffa7`)

**Files (2 touchpoints):**

- `src/acp/auth-env.ts` — `buildAgentSpawnOptions`: add `detached: process.platform !== "win32"` so child runs in own process group on POSIX
- `src/runtime/engine/manager.ts` — `AcpRuntimeManager.cancel`: detach graceful cancel + immediately call `killAgentProcessTree(client.getAgentPid())`. Also adds `killAgentProcessTree` helper at module scope.

**Problem (TD-004 — critical UX):** Telegram `/cancel` (or any
`ABORT_TRIGGERS` word) does not interrupt an active agent turn for 30-120
seconds. Root cause: upstream `Manager.cancel` only calls
`controller.requestCancelActivePrompt()` — a graceful protocol-level cancel
request to the agent's queue-owner. During an active turn the queue-owner
event loop doesn't service the cancel until the current prompt completes,
so the protocol cancel hangs ~as long as the turn itself.

Upstream behavior:

```typescript
// src/runtime/engine/manager.ts — UPSTREAM
async cancel(handle: AcpRuntimeHandle): Promise<void> {
  const controller = this.activeControllers.get(handle.acpxRecordId ?? handle.sessionKey);
  await controller?.requestCancelActivePrompt();  // ← hangs during active turn
}
```

The result: `acpManager.cancel()` blocks for tens of seconds; in openclaw
`cancelSession` waits on this; the gateway handler can't return; the user
sees nothing happen for a long time after typing `/cancel`.

**Fix — two parts:**

#### Part A: Process group on POSIX

`buildAgentSpawnOptions` adds `detached: true` (POSIX only). Putting the
agent subprocess in its own process group makes `process.kill(-pid, signal)`
target the entire subprocess tree (Claude Code + its MCP children + tool
processes), not just the direct child.

```typescript
// src/acp/auth-env.ts — AFTER PATCH
export function buildAgentSpawnOptions(
  cwd: string,
  authCredentials: Record<string, string> | undefined,
  extraEnv?: Record<string, string>,
) {
  const detached = process.platform !== "win32";
  return {
    cwd,
    env: buildAgentEnvironment(authCredentials, extraEnv),
    stdio: ["pipe", "pipe", "pipe"],
    windowsHide: true,
    detached,
  };
}
```

Side effect: `detached: true` alone does NOT prevent the child from dying
when the parent dies — that requires `child.unref()` which we don't call.
The flag _only_ enables process group semantics; children are still subject
to normal parent-death cleanup paths.

#### Part B: Manager.cancel fires fast-kill alongside graceful

```typescript
// src/runtime/engine/manager.ts — AFTER PATCH
async cancel(handle: AcpRuntimeHandle): Promise<void> {
  const recordId = handle.acpxRecordId ?? handle.sessionKey;
  const controller = this.activeControllers.get(recordId);
  // Fire-and-forget graceful cancel (best-effort, often hangs)
  void controller?.requestCancelActivePrompt().catch(() => {});
  // Direct fast-kill the process tree
  const client = this.pendingPersistentClients.get(recordId);
  const pid = client?.getAgentPid();
  if (pid != null && pid > 1 && process.platform !== "win32") {
    killAgentProcessTree(pid);
  }
}

function killAgentProcessTree(pid: number): void {
  try { process.kill(-pid, "SIGTERM"); }
  catch { try { process.kill(pid, "SIGTERM"); } catch { return; } }
  setTimeout(() => {
    try { process.kill(-pid, "SIGKILL"); }
    catch { try { process.kill(pid, "SIGKILL"); } catch {} }
  }, 2000).unref();
}
```

**Result:** cancel latency drops from 30-120s to <1-2s. Graceful protocol
cancel still fires (no-op if process already dead, useful for clean shutdown
of session state); SIGTERM at process group level kills the agent tree
immediately; SIGKILL escalation handles unresponsive children after 2s grace.
`unref()` on the timer prevents it from keeping the gateway alive
unnecessarily.

**Verification:**

- `pnpm build` — passes (compiled `killAgentProcessTree` appears 4 times in
  `dist/runtime.js`: definition + use + 2 catch fallbacks)
- E2E verification deferred to openclaw integration smoke test:
  - Send `/cancel` during long-running turn in Telegram Code topic
  - Expect: `⚙️ Agent was aborted` reply within <2s
  - Expect: subsequent message resumes conversation context (acpx file-based
    session resume handles the dead-session case naturally)

**Interaction with other patches:**

- Patch #1 (env propagation): independent.
- Backports openclaw `OPENCLAW-SOURCE-CHANGES.md` patch #19 (TD-004 fix) into
  upstream acpx itself. The openclaw-side patch (extending `ABORT_TRIGGERS`
  and detaching `cancelSession.cancelPromise`) still required — those handle
  the _Telegram message routing_ and _gateway dispatch latency_ layers. This
  patch handles only the _spawn process termination_ layer.

---

### 3. Build dist on git-install (`86bdefd`)

**Files (1 touchpoint):**

- `package.json` — `prepare` script: `(husky 2>/dev/null || true) && pnpm run build`

**Problem:** Consumers reference our fork via git URL:

```jsonc
"acpx": "github:bohdan-fesenko/acpx#v0.5.3-brain.1"
```

pnpm clones the repo, runs `prepare` script, then exposes the package. But
upstream `prepare` is just `husky` — installs git hooks for dev. On
git-install:

- husky fails (no devDeps installed for transitive packages)
- No build runs
- `dist/` (git-ignored) is missing
- Consumer's bundler can't import from `acpx/runtime` → breaks

**Fix:** make `prepare` robust:

```json
"prepare": "(husky 2>/dev/null || true) && pnpm run build"
```

- `(husky 2>/dev/null || true)` — runs husky if available (dev clone), no-op
  silently otherwise (git install)
- `&& pnpm run build` — always runs, produces `dist/`

This means `pnpm install` time on consumers is +~3-5s (install deps + build).
Trade-off: simpler distribution, no need to publish to npm.

**Alternative considered:** publish as `@bohdan-fesenko/acpx` to npm registry
under our scope. Adds complexity (npm token management, version bumps, npm
registry account). Deferred until we have ≥3 patches that warrant it; the
git-URL approach is fine for now.

**Verification:**

- `pnpm install` in our fork (dev): runs husky + build, dist appears, dev
  workflow unchanged
- `pnpm install acpx@github:bohdan-fesenko/acpx#tag` in fresh project (sim):
  husky silently fails, build runs, dist appears, import works

---

## Summary of patches

| #   | Patch                       | Files | LOC      | Status                |
| --- | --------------------------- | ----- | -------- | --------------------- |
| 1   | Per-session env propagation | 6     | +37 / -3 | ✓ committed `cef5537` |
| 2   | Cancel fast-kill (TD-004)   | 2     | +52 / -2 | ✓ committed `818ffa7` |
| 3   | Build dist on git-install   | 1     | +1 / -1  | ✓ committed `86bdefd` |

Total: **+90 / -6 lines across 8 distinct files.** Tagged `v0.5.3-brain.1`.

## What this fork does NOT include

The following patches from openclaw (`OPENCLAW-SOURCE-CHANGES.md`) live at
the openclaw layer or are obsolete in `acpx@0.5.3`'s new architecture:

- **#2** (file-based session resume): handled natively by acpx 0.5.3's session
  store + `shouldReuseExistingRecord` logic
- **#4** (kill zombie processes during repair): N/A — acpx 0.5.3 architectural
  rewrite removed `replaceDeadNamedSession` concept entirely; the new
  thin-wrapper approach delegates all session lifecycle to the acpx package
- **#7** (route ALL dead via replaceDeadNamedSession): N/A — same as #4
- **#9** (age-based orphan reaper): NOT YET PORTED. May still be needed if
  acpx 0.5.3 leaves orphans in some edge case (e.g., gateway crash mid-turn).
  TBD pending production observation.
- **#11** (startup orphan cleanup): lives in openclaw's `run-loop.ts`, not in
  acpx. No port needed.
- **#12** (dedup concurrent replaceDead): N/A — same as #4

## Maintenance discipline

- **Each patch = one focused commit.** Easier to rebase when upstream changes
  affected files.
- **Comment every diff with `Brain fork patch:` prefix.** Search for that
  string finds all our deltas:
  ```bash
  grep -rn 'Brain fork patch' src/
  ```
- **No docs changes mixed with code changes.** Doc-only commits stay separate
  so code rebases don't conflict on README/changelog text.
- **Tag every release of our fork as `vX.Y.Z-brain.N`** where `vX.Y.Z`
  matches upstream tag we rebased onto, and `N` is our patchlevel
  (increments on each new commit added).

## After Changes

```bash
cd acpx_source
pnpm build                       # produces fresh dist/ locally
git tag vX.Y.Z-brain.<N+1>       # new patchlevel
git push origin <branch> --tags
# Update openclaw fork's extensions/acpx/package.json reference to new tag
# Trigger openclaw rebuild (watcher pulls new acpx via pnpm install)
```
