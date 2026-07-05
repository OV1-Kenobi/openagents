# Khala Code Desktop — Repo-Level Context Window

**Audience:** operators who want Khala Code Desktop's editor pane wired to a specific local checkout — including on Windows 11 running WSL2 Ubuntu — so the LLM harness has repo-level file/directory context for coding work.

**Companion doc:** [`docs/dogfood/windows-wsl-pylon-2026-07.md`](../../../docs/dogfood/windows-wsl-pylon-2026-07.md) — the Windows/WSL Pylon dogfood walkthrough this repo-context setup pairs with.

---

## Table of contents

1. [What "repo-level context window" means here](#what-repo-level-context-window-means-here)
2. [The one env var that controls it](#the-one-env-var-that-controls-it)
3. [Where the context comes from (the editor file service)](#where-the-context-comes-from-the-editor-file-service)
4. [Setup on macOS / native Linux](#setup-on-macos--native-linux)
5. [Setup on Windows 11 + WSL2 Ubuntu](#setup-on-windows-11--wsl2-ubuntu)
6. [Verify the context window is live](#verify-the-context-window-is-live)
7. [Safe defaults, limits, and what NOT to point it at](#safe-defaults-limits-and-what-not-to-point-it-at)
8. [Multi-repo / worktree usage](#multi-repo--worktree-usage)
9. [Troubleshooting](#troubleshooting)
10. [Rollback](#rollback)

---

## What "repo-level context window" means here

Khala Code Desktop's editor pane (introduced in the mid-2026 editor foundation commits — read-only file tree, hotbar entry, provider-neutral file service) exposes a live view of a repository to:

- the desktop shell's editor UI (file tree, open files, tab bar), and
- the underlying Codex harness's tool calls (directory reads, file reads, workspace metadata) when the LLM asks for repo context.

Under the hood this is served by [`src/bun/editor-file-service.ts`](../src/bun/editor-file-service.ts) — a provider-neutral file service that answers `providerList`, `workspaceRead`, `directoryRead`, and `fileRead` RPCs. Its default provider is `local-workspace`, rooted at whatever path the app resolves as the working directory at launch.

The "repo-level context window" is therefore just: **make sure Khala Code Desktop resolves its `workingDirectory` to the root of your OpenAgents fork checkout, not to a random home directory.**

---

## The one env var that controls it

```
KHALA_CODE_DESKTOP_WORKSPACE
```

The Khala chat runtime resolves the workspace root in this order (see [`src/bun/khala-chat-runtime.ts`](../src/bun/khala-chat-runtime.ts):

1. Explicit `workingDirectory` passed by the caller (used by tests / headless smokes).
2. `process.env.KHALA_CODE_DESKTOP_WORKSPACE`.
3. `process.cwd()` at process start.

Rule of thumb: if you launch `bun run dev:khala-code-desktop` from your repo root, `process.cwd()` already resolves to the right thing and you can skip the env var. If you launch from anywhere else — a launcher shortcut, a systemd unit, a background service — set the env var explicitly.

---

## Where the context comes from (the editor file service)

`editor-file-service.ts` treats `workingDirectory` as an **allowlist root**. All directory and file reads:

- resolve requested paths relative to the root,
- reject any path that escapes the root (`..`, absolute paths outside the tree, symlinks pointing out),
- cap file reads at `KHALA_CODE_EDITOR_DEFAULT_MAX_FILE_BYTES` unless the caller overrides,
- return typed errors (`out-of-root`, `not-found`, `too-large`, etc.) rather than raw filesystem errors.

Because the root is enforced, pointing the workspace at a broad path (e.g. `~/` or `C:\Users\<you>`) is both wasteful (huge tree walks) and a data-exposure risk (every dotfile becomes reachable to the harness). Always point at the repo root.

---

## Setup on macOS / native Linux

### Verify before

```sh
cd ~/path/to/your/openagents           # your fork checkout
pwd                                    # confirm this is the repo root (should contain package.json + apps/pylon)
git rev-parse --show-toplevel          # same path
```

### Configure

Add to your shell RC (`~/.zshrc` or `~/.bashrc`):

```sh
export KHALA_CODE_DESKTOP_WORKSPACE="$HOME/path/to/your/openagents"
```

Reload the shell (`exec $SHELL -l`) or `source` the RC file.

### Launch

```sh
cd "$KHALA_CODE_DESKTOP_WORKSPACE"
bun install                            # only needed once, or after dep changes
bun run dev:khala-code-desktop
```

### Verify after

Open the editor pane from the hotbar. The file tree should show the repo root's top-level directories (`apps/`, `clients/`, `packages/`, `docs/`, …). If it shows `$HOME` contents instead, the env var did not propagate — see [Troubleshooting](#troubleshooting).

---

## Setup on Windows 11 + WSL2 Ubuntu

> **Read this first.** Khala Code Desktop today is Electrobun-based and targets macOS as the primary desktop shell (see [`clients/khala-code-desktop/README.md`](../README.md) — "macOS is the primary target"). On a Windows box, running the **desktop shell** natively is not the supported path. What IS supported and useful for you is running Khala Code's Bun-side services + editor file service inside WSL2 so:
>
> - the file service indexes your WSL2-side checkout at native ext4 speed,
> - the chat runtime and editor RPCs run against that checkout,
> - anything you display later (via a browser preview, headless smoke, or the eventual Windows shell) reads the same context.
>
> If a Windows desktop shell arrives later, this env var still applies — but for now, treat this as WSL-side setup for the file/context layer.

### Verify before (in WSL2 Ubuntu)

```bash
cd ~/openagents                         # your fork checkout inside WSL, per the dogfood doc
pwd                                     # should be /home/<you>/openagents
git rev-parse --show-toplevel           # same path
uname -a                                # should include "microsoft" or "WSL"
```

If your checkout is on `/mnt/c/...`, **stop and re-clone into `~/`**. See the dogfood doc's [Clone your fork inside WSL](../../../docs/dogfood/windows-wsl-pylon-2026-07.md#clone-your-fork-inside-wsl-not-on-mntc) section.

### Configure

Add to `~/.bashrc`:

```bash
export KHALA_CODE_DESKTOP_WORKSPACE="$HOME/openagents"
```

Reload:

```bash
exec bash
echo "$KHALA_CODE_DESKTOP_WORKSPACE"    # should print /home/<you>/openagents
```

### Launch the Bun-side pieces (WSL)

```bash
cd "$KHALA_CODE_DESKTOP_WORKSPACE"
bun install                             # first time, or after dep changes
```

Then pick ONE of:

- **Headless smokes** (works today, produces repo-context receipts):
  ```bash
  bun test clients/khala-code-desktop/tests/editor-file-service
  bun test clients/khala-code-desktop/tests/editor-panel
  ```
  Both suites exercise the file service against the current checkout. Expected: all tests pass. Save the log — this is a receipt that repo-context indexing works on WSL2.

- **Desktop shell attempt** (may fail on Windows/WSL — capture the failure as evidence):
  ```bash
  bun run dev:khala-code-desktop 2>&1 | tee ~/khala-code-desktop-launch.log
  ```
  Electrobun is Mac-first; if this errors, that IS the receipt for the "no Windows desktop shell yet" gap. Do NOT try to make it work with hacks — file an issue instead.

### Verify after

```bash
# The file service should treat your checkout as the workspace root
grep -E "^export KHALA_CODE_DESKTOP_WORKSPACE" ~/.bashrc
echo "$KHALA_CODE_DESKTOP_WORKSPACE"
ls "$KHALA_CODE_DESKTOP_WORKSPACE/apps/pylon" >/dev/null && echo "workspace root looks correct"
```

---

## Verify the context window is live

Regardless of platform, once Khala Code Desktop (or its Bun-side file service) is running against `KHALA_CODE_DESKTOP_WORKSPACE`, you can prove the context window is wired by asking the harness a question that requires actually reading the repo:

- "List the top-level directories under the workspace root."
- "Read the first 40 lines of `apps/pylon/src/wsl-host-detect.ts`."
- "How many `.md` files are under `docs/dogfood/`?"

If it answers correctly and cites the paths, the file service is live and the LLM has repo-level context. If it hallucinates paths that don't exist in your checkout, the workspace root is wrong or the file service isn't wired — see [Troubleshooting](#troubleshooting).

The editor pane's file tree is the visual complement: it lists real entries from the workspace root and refuses paths that escape the root. If the tree is empty or shows an unexpected location, same diagnosis.

---

## Safe defaults, limits, and what NOT to point it at

**Do:**

- Point at a repo root — the checkout of your fork.
- Use one workspace per Khala Code Desktop launch. Multi-workspace switching is not supported today.
- Keep the checkout on the native filesystem (macOS APFS, Linux ext4, or WSL2 ext4 — **never** `/mnt/c/…`).

**Do NOT:**

- Point at `$HOME` or `C:\Users\<you>`. The tree walk is huge and every dotfile becomes readable to the harness.
- Point at `/`, `/etc`, `/var`, or any path containing wallet material, Codex home (`~/.codex`), or SSH keys (`~/.ssh`).
- Point at a symlink into a mounted network share or `/mnt/c/…`. WSL2's DrvFS is slow and unreliable for file watchers.
- Rely on the file service to hide dotfiles — it does not. If a repo has an `.env` or a `.credentials` file at its root, it is readable through the workspace.

**Byte limit:** individual file reads are capped at `KHALA_CODE_EDITOR_DEFAULT_MAX_FILE_BYTES` (currently a few hundred KB, defined in [`src/shared/editor.ts`](../src/shared/editor.ts)). Callers can pass a lower cap; they cannot raise it above the default. Large lockfiles and generated assets are effectively out of scope by design.

---

## Multi-repo / worktree usage

The workspace is a single root. If you have several fork branches you dogfood in parallel (e.g. via `git worktree add`), keep each in its own directory and launch a separate Khala Code Desktop instance per worktree:

```bash
# In one WSL terminal — branch A
cd ~/openagents-feat-windows-wsl-pylon-coverage
KHALA_CODE_DESKTOP_WORKSPACE="$PWD" bun run dev:khala-code-desktop

# In another WSL terminal — branch B
cd ~/openagents-feat-autopilot-project-management
KHALA_CODE_DESKTOP_WORKSPACE="$PWD" bun run dev:khala-code-desktop
```

Each process sees only its own root. There is no built-in "switch workspace" affordance; relaunching against a different env var is the supported path today.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Editor tree shows `$HOME` contents, not the repo | `KHALA_CODE_DESKTOP_WORKSPACE` not set OR launched from wrong `cwd` | Set the env var explicitly, then `exec bash`, then relaunch |
| Editor tree is empty | Workspace root points at a non-existent path | `echo "$KHALA_CODE_DESKTOP_WORKSPACE"` and `ls` it |
| File reads return `out-of-root` errors | Path escapes the workspace (symlink out, `..` traversal) | Move the target inside the workspace, or rewrite the path relative to the root |
| File reads return `too-large` | File exceeds the byte cap | This is by design. Read a slice or split the file. Do not raise the cap without an ADR |
| `bun run dev:khala-code-desktop` errors on Windows/WSL | Electrobun's macOS-first shell not building on this host | Expected today. Capture the log, use the headless smokes instead, and file an issue if you want Windows shell support prioritized |
| File watcher misses changes | Checkout is on `/mnt/c/…` | Re-clone into WSL2 ext4 (`~/openagents`). See dogfood doc |
| Tree shows `.env` / secrets | Workspace root is too broad | Point at the repo root, not `$HOME`. Add unwanted files to `.gitignore` — the file service does not filter beyond the root |

---

## Rollback

The setup is env-var-driven and non-destructive. To unwind:

```bash
# 1. Remove the export from your RC file
sed -i '/^export KHALA_CODE_DESKTOP_WORKSPACE=/d' ~/.bashrc
# (or ~/.zshrc on macOS)

# 2. Unset for the current shell
unset KHALA_CODE_DESKTOP_WORKSPACE

# 3. Kill any running Khala Code Desktop process
pkill -f khala-code-desktop 2>/dev/null || true

# 4. (Optional) Remove any cached editor state — safe to delete, will be re-created
#    Location depends on Electrobun's config path; see clients/khala-code-desktop/README.md
```

Nothing writes into your repo checkout, so there is no in-tree state to clean up.

---

## Provenance

- Branch: [`feat/windows-wsl-pylon-coverage`](https://github.com/OV1-Kenobi/openagents/tree/feat/windows-wsl-pylon-coverage)
- Companion: [`docs/dogfood/windows-wsl-pylon-2026-07.md`](../../../docs/dogfood/windows-wsl-pylon-2026-07.md)
- Source of truth for the file service: [`clients/khala-code-desktop/src/bun/editor-file-service.ts`](../src/bun/editor-file-service.ts)
- Env var contract: [`clients/khala-code-desktop/src/bun/khala-chat-runtime.ts`](../src/bun/khala-chat-runtime.ts) (search for `KHALA_CODE_DESKTOP_WORKSPACE`)
- No changes to `apps/pylon/`. No changes to the honesty-guard suite. This is documentation only.
