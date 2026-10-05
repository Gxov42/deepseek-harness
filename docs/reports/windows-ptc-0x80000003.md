# [Windows] PTC worker dies instantly with 0x80000003 under the ACL sandbox; nodeExecutable default should not be process.execPath

## Summary

On Windows, every `run_code` / PTC tool call made in a restricted sandbox mode (`workspace-write` or `read-only`) fails immediately with:

```
Error: code run failed (worker-exit): Node process exited before completing (2147483651)
```

`2147483651` = `0x80000003` = `STATUS_BREAKPOINT`. The worker produces **no stderr at all** — it never reaches application code.

**Root cause:** `NodePtcRuntime` defaults `nodeExecutable` to `process.execPath` (`dsh-ptc-runtime-node/lib/index.js:794`). In the packaged desktop app that path is the **Electron binary** (`DeepSeek Harness.exe`). The Windows ACL sandbox runner spawns it with a restricted primary token (`dsh-sandbox-local/lib/index.js:539-553`), and Chromium's startup CHECK fires before early startup completes — an Electron exe cannot boot under a restricted token, so it dies via `base::ImmediateCrash()` (`__debugbreak` → `0x80000003`).

Setting `ELECTRON_RUN_AS_NODE=1` does **not** help (tested; same crash), so this is not the "unsupported NODE_OPTIONS" path — the restricted token itself is rejected during early startup.

## Environment

- DeepSeek Harness packaged desktop app, Windows host
- Electron 44.0.0 / embedded Node 24.18.1 (`process.versions` inside the host)
- `@deepseek-ai/dsh-ptc-runtime-node@0.2.0-rc.2`
- Sandbox mode: `workspace-write` (also reproduces on `read-only`)

## Reproduction

1. Launch the packaged app on Windows.
2. Call any PTC tool (e.g. `run_code`) with default session permissions — the ACL runner wraps the spawn in a restricted token.
3. Worker exits instantly with `Node process exited before completing (2147483651)`, empty stderr.

## Evidence matrix

Spawn combinations against the real workspace / real host exe, SID from `workspaceWriteSid` / `tempWriteSid`:

| Spawn | Result |
|---|---|
| ACL runner + `workspace-write` + Electron exe | `0x80000003`, empty output |
| ACL runner + `read-only` + Electron exe | `0x80000003`, empty output |
| ACL runner + `workspace-write` + `ELECTRON_RUN_AS_NODE=1` | `0x80000003`, empty output |
| ACL runner + bundled `node.exe -e` (same token) | exit 0, `NODE-OK` |

The same restricted-token path runs the ACL runner itself under the bundled Node fine, and a plain nested process in `workspace-write` runs fine (enforcement reports `partial` on this host). The crash is specific to booting the **Electron** binary as the worker.

Key exclusions: not the temp directory, not a non-ASCII (Chinese) workspace path, not the inherited environment (an empty-env spawn still crashes), and not a missing bootstrap — the process dies before it can write anything.

## Suggested fix

Make `nodeExecutable` default to a real Node runtime instead of `process.execPath` when the host is a packaged Electron app — for example the same bundled Node the sandbox layer already uses for the ACL runner itself (`dsh-sandbox-local/lib/index.js:543` hardcodes `...\runtime\primary-runtime\dependencies\node\bin\node.exe`), or the runtime under `dsh-runtimes`.

Note that `bootstrapPath` (documented in the package README) then becomes **required**, because a bare `node.exe` cannot read `app.asar` virtual paths: the default bootstrap argument resolves `./process.js` through `fs.processPathFromHostPath`, which yields an asar path that only Electron's patched `fs` can open. Setting `nodeExecutable` **without** `bootstrapPath` swaps one failure for another:

```
Error: Cannot find module '...app.asar\dsh\node_modules\@deepseek-ai\dsh-ptc-runtime-node\lib\process.js'
MODULE_NOT_FOUND  (Node.js v24.21.0)
```

So the two options really ship as a pair.

## Workaround (works today, no source changes)

1. Extract the bootstrap outside the asar:
   - copy `app.asar\dsh\node_modules\@deepseek-ai\dsh-ptc-runtime-node\lib\process.js` to a real path (e.g. `$DSH_HOME\ptc-bootstrap\process.js`)
   - put a `{"type":"module"}` package.json next to it (the file is ESM; 3 built-in imports, fully self-contained)
2. Override the preset in `$DSH_HOME/profiles/<profile>/cordis.patch.yml`:

   ```yaml
   - id: ptc-runtime
     name: "@deepseek-ai/dsh-ptc-runtime-node"
     config:
       nodeExecutable: '<bundled-node>\node.exe'
       bootstrapPath: '<extracted>\process.js'
   ```

The worker must be spawned with `DSH_SUBPROCESS_CONTROL=pipe` and fd 7 wired to the control pipe (`process.js:13-23`); that contract is unchanged and works under plain Node.

**Verified:** with this override, `run_code` under default `workspace-write` permissions (no elevation) runs on bundled Node 24.21.0 and passes a smoke test.

## Why this matters

Without the workaround, Windows users in default sandbox modes cannot run any PTC tool at all; the only escape is elevating every call to `danger-full-access`, which defeats the sandbox.

---
*Filed as a discussion because Issues are disabled on this repository.*
