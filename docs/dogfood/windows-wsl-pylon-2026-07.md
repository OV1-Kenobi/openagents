# Windows / WSL2 Pylon Dogfood Guide (2026-07)

**Author:** OV1-Kenobi (Chris) — the only Windows operator on the design sprint team
**Fork:** [OV1-Kenobi/openagents](https://github.com/OV1-Kenobi/openagents), branch `feat/windows-wsl-pylon-coverage`
**Purpose:** Capture a first-person, receipt-first walkthrough of running `apps/pylon` on Windows 11 host + WSL2 Ubuntu, so upstream owners can decide whether to lift the `windows_wsl_consumer_install_coverage_missing` blocker with real evidence attached.

> **Honest scope up front.** This document is **not** a coverage claim. Upstream's `apps/pylon/src/consumer-install-platform-support.ts` classifier deliberately holds WSL hosts `inScope: false` (see the [Why the bootstrap will say you're out of scope](#why-the-bootstrap-will-say-youre-out-of-scope) section). This guide walks a Windows operator through installing Pylon **inside** WSL2 Ubuntu, exercising the earn-loop paths, and capturing receipts. Nothing in this guide changes the classifier, the README audit, or the launch-scope contract. That change requires an owner sign-off separately from this dogfood.

---

## Table of contents

1. [Prerequisites](#prerequisites)
2. [WSL2 Ubuntu setup on Windows 11](#wsl2-ubuntu-setup-on-windows-11)
3. [Clone your fork inside WSL (not on `/mnt/c/`)](#clone-your-fork-inside-wsl-not-on-mntc)
4. [Install Bun, Node, Codex inside Ubuntu](#install-bun-node-codex-inside-ubuntu)
5. [Run the Pylon bootstrap and read the honest verdict](#run-the-pylon-bootstrap-and-read-the-honest-verdict)
6. [Why the bootstrap will say you're out of scope](#why-the-bootstrap-will-say-youre-out-of-scope)
7. [Exercise the Pylon paths that WORK today](#exercise-the-pylon-paths-that-work-today)
8. [Receipt capture checklist](#receipt-capture-checklist)
9. [Known upstream blockers to check against](#known-upstream-blockers-to-check-against)
10. [Rollback / clean uninstall](#rollback--clean-uninstall)
11. [What to do with the receipts](#what-to-do-with-the-receipts)

---

## Prerequisites

- Windows 11 (Pro or Home, build 22000+)
- Local admin on the Windows box (needed for WSL2 install, virtualization enable)
- 20+ GB free on the drive that will host the WSL2 VHDX (default `C:`)
- A working `git` on Windows is **not** required — we clone inside WSL
- GitHub credentials for `OV1-Kenobi/openagents` (PAT or `gh auth login` in WSL is fine)

> **Never work off `/mnt/c/…` in WSL.** The Windows-to-Linux file system bridge on WSL2 is orders of magnitude slower than the native ext4 volume, and file watchers/inotify events are unreliable. Every step below assumes your checkout lives inside the Ubuntu home directory (`~/`).

---

## WSL2 Ubuntu setup on Windows 11

All commands below run in **PowerShell (Admin)** on the Windows host. Windows-only, no `cmd.exe` shorthand.

### Verify before you install

```powershell
# Verify: is virtualization enabled?
Get-ComputerInfo -Property "HyperV*"

# Verify: is WSL already installed?
wsl --status
```

Expected `wsl --status` output when WSL is NOT installed yet: `The Windows Subsystem for Linux is not installed. Install it by running: wsl --install`.

### Install (destructive: enables virtualization features, requires reboot)

```powershell
# Installs WSL2 + Ubuntu (default distro)
wsl --install --distribution Ubuntu-24.04
```

Reboot when prompted. On next login, Ubuntu opens a terminal and asks for a UNIX username + password. Choose something you'll remember; this is your Linux user, not your Windows user.

### Verify after install

```powershell
wsl --status
wsl -l -v
```

Expected: `NAME` = `Ubuntu-24.04`, `STATE` = `Running`, `VERSION` = `2`.

### Rollback (if you want to remove WSL entirely)

```powershell
wsl --unregister Ubuntu-24.04
wsl --uninstall
# Optional: disable the feature entirely
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart
```

`wsl --unregister` **permanently deletes** the Ubuntu VHDX and everything inside it, including any Pylon config, wallet material, or dogfood receipts you captured. Copy anything you care about to Windows first (`\\wsl$\Ubuntu-24.04\home\<you>\...` in Explorer).

---

## Clone your fork inside WSL (not on `/mnt/c/`)

Open a WSL Ubuntu shell (Start menu → "Ubuntu 24.04") or run `wsl` from PowerShell. Every command from here down runs inside Ubuntu.

### Verify before

```bash
pwd                       # should be /home/<you>
uname -a                  # should include "microsoft" or "WSL" → confirms WSL2 kernel
cat /proc/version         # same signal, more specific
echo "$WSL_DISTRO_NAME"   # should print "Ubuntu-24.04"
```

The `WSL_DISTRO_NAME` env var, `/proc/version` "microsoft" marker, and `uname -a` all match the signals [`apps/pylon/src/wsl-host-detect.ts`](../../apps/pylon/src/wsl-host-detect.ts) checks. Capturing them here is the first receipt.

### Clone

```bash
# Shallow clone your fork — INSTALL.md's rule applies to WSL too
cd ~
git clone --depth 1 --branch feat/windows-wsl-pylon-coverage \
  https://github.com/OV1-Kenobi/openagents.git
cd openagents
```

### Verify after

```bash
git status
git branch --show-current                 # feat/windows-wsl-pylon-coverage
git log --oneline -5
du -sh .git                                # ~40 MB for a shallow clone
```

### Rollback

```bash
cd ~
rm -rf ~/openagents
```

---

## Install Bun, Node, Codex inside Ubuntu

### Verify before

```bash
which bun 2>/dev/null || echo "bun not installed"
which node 2>/dev/null || echo "node not installed"
which codex 2>/dev/null || echo "codex not installed"
```

### Install Node 20 (via nvm — the least-destructive path)

```bash
# nvm is user-scoped, does not need sudo, easy to roll back
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install 20
nvm use 20
node --version              # expect v20.x
```

### Install Bun 1.3+

```bash
curl -fsSL https://bun.sh/install | bash
# Reload shell to pick up ~/.bashrc PATH change
exec bash
bun --version               # expect >= 1.3.0
```

### Install Codex CLI

Follow the same guidance as [repo-root INSTALL.md](../../INSTALL.md): install `@openai/codex` and log in **only if you do not already have a live Codex session on this machine**. Under WSL, each distro has its own `~/.codex` — installing Codex inside Ubuntu does **not** touch a Windows-side Codex install.

```bash
npm install -g @openai/codex
# Only run codex login if you have never logged in inside this WSL distro:
codex login
```

### Verify after

```bash
node --version
bun --version
codex --version
```

### Rollback (uninstall Bun + nvm + Codex in Ubuntu)

```bash
# Codex
npm uninstall -g @openai/codex
# Bun
rm -rf ~/.bun ~/.bunfig
# nvm (destroys all node versions installed via nvm)
rm -rf ~/.nvm
# Clean shell RC entries
sed -i '/NVM_DIR/d;/bun/d' ~/.bashrc
```

---

## Run the Pylon bootstrap and read the honest verdict

### Install workspace deps

```bash
cd ~/openagents
bun install               # MUST run at repo root, not in apps/pylon
```

Expected: Bun resolves the workspace, writes `bun.lockb`. On WSL2 native ext4 this is fast (~30 s for a cold cache); on `/mnt/c/` it can take 10+ minutes and may fail — another reason to stay in `~/`.

### Run the Pylon bootstrap directly (dry / summary only)

The bootstrap exposes a summary object without actually starting any long-running node. Reading it is the fastest, safest way to confirm the classifier's verdict on your machine.

```bash
cd ~/openagents/apps/pylon
bun run --silent -- node -e "
  import('./src/bootstrap.js').then(m => {
    const summary = m.createBootstrapSummary(
      { registerOpenAgents: false, setupMdkWallet: false, resourceMode: 'idle' },
    );
    console.log(JSON.stringify(summary.platform, null, 2));
  });
"
```

**Expected output on WSL2 Ubuntu:**

```json
{
  "current": "linux",
  "supported": true,
  "supportedTargets": ["darwin-arm64", "linux-x64", "linux-arm64"],
  "wsl": true,
  "inScope": false
}
```

That `inScope: false` on a `linux` host with `wsl: true` **is the honest signal upstream engineered.** It is Pylon telling you: "I can run on the WSL kernel because Node/Bun see this as Linux, but the launch scope does not cover this host, so I will not claim earning coverage." Capture this JSON verbatim — it is your first receipt.

### Try the actual `npx @openagentsinc/pylon` path (optional, more evidence)

```bash
# From ANY directory (npx does not need the checkout)
cd ~
npx @openagentsinc/pylon --help 2>&1 | tee ~/pylon-help.log
```

Then attempt a bootstrap that DOES register / write config, in a scratch home so you can nuke it later:

```bash
export PYLON_HOME="$HOME/pylon-dogfood-2026-07"
mkdir -p "$PYLON_HOME"
npx @openagentsinc/pylon bootstrap --resource-mode=idle 2>&1 | tee ~/pylon-bootstrap.log
```

**Expected behavior:** Pylon will run, print its bootstrap summary, and — if the current build honors the classifier — refuse to advance to any earning path with a message referencing `blocker.product_promises.windows_wsl_consumer_install_coverage_missing`. If it advances anyway, that is itself a receipt worth filing.

---

## Why the bootstrap will say you're out of scope

Reading the code, three modules interlock to make WSL an explicit out-of-scope host:

1. **[`apps/pylon/src/wsl-host-detect.ts`](../../apps/pylon/src/wsl-host-detect.ts)** — a dependency-free leaf that returns `true` whenever any of `WSL_DISTRO_NAME`, `WSL_INTEROP`, `WSLENV` env vars are set, or `/proc/version` contains "microsoft"/"wsl". It is deliberately a leaf so `bootstrap.ts` and the classifier can both consume it without a cycle.

2. **[`apps/pylon/src/bootstrap.ts`](../../apps/pylon/src/bootstrap.ts) (line ~219)** — `createBootstrapSummary` computes `wsl = currentPlatform === "linux" && detectWslHost(env)` and then `inScope: supported && !wsl`. So a genuine Ubuntu box on bare metal is `inScope: true`; the same Ubuntu userland under WSL is `inScope: false`.

3. **[`apps/pylon/src/consumer-install-platform-support.ts`](../../apps/pylon/src/consumer-install-platform-support.ts)** — the classifier + `verifyConsumerInstallPlatformClaim` + `auditReadmePlatformCopy` trio. `verifyConsumerInstallPlatformClaim` hard-rejects any claim asserting `wslInScope: true` or `windowsInScope: true` with reason `windows-wsl-not-in-launch-scope`. `auditReadmePlatformCopy` regex-scans `apps/pylon/README.md` for coverage phrases and fails CI if any appear.

**None of that changes in this dogfood.** We do not touch the classifier, the verifier, `apps/pylon/README.md`, or the README audit test. We only run Pylon on a WSL host, capture what happens, and file evidence.

---

## Exercise the Pylon paths that WORK today

Even with `inScope: false`, several Pylon subsystems are exercisable under WSL2 Ubuntu and produce useful receipts:

### 1. `bun test` the WSL-related suites (safe, read-only, expected to PASS)

```bash
cd ~/openagents
bun test apps/pylon/src/wsl-host-detect
bun test apps/pylon/src/consumer-install-platform-support
bun test apps/pylon/tests/consumer-install-readme-copy-guard
```

Expected: all three suites pass. This is the receipt that **your fork's tests still hold the honesty-guard line** — critical evidence that this dogfood didn't regress the launch scope contract.

### 2. Boot the Pylon socket layer, watch for the leak from [issue #4349](https://github.com/OpenAgentsInc/openagents/issues/4349)

```bash
cd ~/openagents/apps/pylon
# Start pylon in a scratch home and log stderr/stdout
PYLON_HOME="$HOME/pylon-dogfood-2026-07" \
  bun run start 2>&1 | tee ~/pylon-run.log &
PYLON_PID=$!

# Let it run 60 s
sleep 60

# Snapshot socket count (WSL2 has a real /proc)
ss -tunp | grep -c pylon || echo 0
lsof -p "$PYLON_PID" 2>/dev/null | wc -l

# Kill
kill "$PYLON_PID"; wait "$PYLON_PID" 2>/dev/null
```

**Capture:** `pylon-run.log` (redact any wallet material before sharing), the socket count over time, and the exit behavior. If sockets grow unboundedly, that is direct evidence for #4349 with a Windows/WSL reproduction attached — genuinely useful even though the underlying bug is not WSL-specific.

### 3. Repo-context indexing via Khala Code (see [companion doc](../../clients/khala-code-desktop/docs/repo-context-window.md))

Khala Code's editor + repo-level context window feature is host-agnostic and does not depend on the Pylon platform classifier. Set it up per the companion doc and index your local `~/openagents/` checkout. This is the piece the user specifically called out and is unblocked on WSL2 today.

---

## Receipt capture checklist

For each of the receipts below, save the raw output to `~/pylon-dogfood-receipts/` inside WSL. **Do not put anything on `/mnt/c/`** and **do not commit receipts to git** — they may include user identifiers or wallet-adjacent data.

- [ ] `env.txt` — output of `env | grep -E "^WSL|^PATH="` (redact PATH if it exposes usernames)
- [ ] `uname.txt` — `uname -a && cat /proc/version`
- [ ] `bootstrap-platform.json` — the JSON from the `createBootstrapSummary` snippet above
- [ ] `pylon-help.log` — `npx @openagentsinc/pylon --help` output
- [ ] `pylon-bootstrap.log` — full run of `npx @openagentsinc/pylon bootstrap`
- [ ] `bun-test-wsl-host-detect.log` — `bun test` output for the WSL detect suite
- [ ] `bun-test-consumer-install.log` — `bun test` output for the classifier suite
- [ ] `bun-test-readme-copy-guard.log` — `bun test` output for the README audit test
- [ ] `pylon-run.log` — the 60-second run capture (redacted)
- [ ] `socket-count-over-time.txt` — `ss` / `lsof` snapshots at t=10s, 30s, 60s

### What must NEVER appear in a receipt

- Codex API keys, GitHub PATs, or any `Bearer`/`Authorization:` header
- Wallet mnemonics, xprivs, or Nostr private keys (`nsec1...`)
- Absolute paths that reveal your Windows username more than once (redact `/home/<you>/` → `/home/dogfooder/`)
- Machine identifiers (`/etc/machine-id`, MAC addresses)

---

## Known upstream blockers to check against

While running, you are gathering evidence against these open questions:

| Upstream ref | What to watch for on WSL2 |
|---|---|
| [`blocker.product_promises.windows_wsl_consumer_install_coverage_missing`](https://github.com/OpenAgentsInc/openagents/issues?q=windows_wsl) | Does `platform.inScope === false` propagate all the way to any UI/log that a real end user would see? Capture the exact user-visible message. |
| [Issue #5527 (EPIC, closed)](https://github.com/OpenAgentsInc/openagents/issues/5527) | The green-gate that references self-serve + Windows/WSL + Spark autostart. Your receipts feed the "dereferenceable receipt" requirement. |
| [Issue #4349 (pylon socket leak)](https://github.com/OpenAgentsInc/openagents/issues/4349) | File-descriptor / socket growth over time on WSL2. Reproduces the bug OR clears WSL as a factor. |
| [Issue #4355 (WSL install instructions, closed)](https://github.com/OpenAgentsInc/openagents/issues/4355) | This dogfood is the follow-through. Compare against the closed branch `codex/issue-4355-wsl-install-instructions` if it exists. |
| [Issue #8282 (Khala Sync EPIC, open)](https://github.com/OpenAgentsInc/openagents/issues/8282) | If you touch Khala Sync on WSL, note any host-specific behavior. |

---

## Rollback / clean uninstall

Order matters — clean up inside WSL first, then WSL itself, so you don't leave orphaned wallet material on the VHDX.

```bash
# 1. Kill any running pylon
pkill -f "@openagentsinc/pylon" 2>/dev/null || true
pkill -f "pylon" 2>/dev/null || true

# 2. Nuke the Pylon dogfood home (contains config, cache, releases)
rm -rf "$HOME/pylon-dogfood-2026-07"
unset PYLON_HOME

# 3. Nuke the receipts folder ONLY after you've copied anything you want to keep
#    to Windows via \\wsl$\Ubuntu-24.04\home\<you>\pylon-dogfood-receipts\
rm -rf "$HOME/pylon-dogfood-receipts"

# 4. Remove the fork checkout
rm -rf "$HOME/openagents"

# 5. (Optional) Uninstall Codex / Bun / nvm as in the earlier rollback section
```

Then, from PowerShell (Admin) on Windows, if you want the whole WSL2 Ubuntu gone:

```powershell
wsl --terminate Ubuntu-24.04
wsl --unregister Ubuntu-24.04
```

---

## What to do with the receipts

1. **Do not attach receipts to a public PR against upstream** without owner review. Some artifacts (socket dumps, run logs) may contain identifiers you missed.
2. **File a targeted upstream issue** using the strict-bug form: <https://github.com/OpenAgentsInc/openagents/issues/new?template=strict-bug.yml>. Title: `Pylon WSL2 Ubuntu dogfood receipts — <YYYY-MM-DD>`. Body: link back to this doc, list the receipts you captured, and — critically — attach only redacted excerpts, not full logs.
3. **Reference [issue #5527](https://github.com/OpenAgentsInc/openagents/issues/5527) and the `windows_wsl_consumer_install_coverage_missing` blocker** so the receipt lands in the right green-gate conversation.
4. **Do not open a PR that flips `wslInScope: true`** or edits `apps/pylon/README.md` from this dogfood. That is a launch-scope design change requiring the classifier owner's sign-off. Propose it in the issue thread and wait for a green light before writing code.

---

## Provenance

- Fork: [OV1-Kenobi/openagents@feat/windows-wsl-pylon-coverage](https://github.com/OV1-Kenobi/openagents/tree/feat/windows-wsl-pylon-coverage)
- Fork sync: fast-forwarded to upstream `main` at commit `44ced78` on 2026-07-05
- Related docs in this commit:
  - [`clients/khala-code-desktop/docs/repo-context-window.md`](../../clients/khala-code-desktop/docs/repo-context-window.md)
  - [`INSTALL.md`](../../INSTALL.md) — "Windows Users" pointer added in this branch
- No changes to `apps/pylon/README.md`, `apps/pylon/src/consumer-install-platform-support.ts`, or the honesty-guard tests. Ship as evidence, not as a coverage claim.
