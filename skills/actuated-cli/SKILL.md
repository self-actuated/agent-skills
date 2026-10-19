---
name: actuated-cli
description: Diagnose and repair actuated CI runner dispatch failures — jobs stuck queued, VMs dying, hosts unreachable. Covers the full diagnostic workflow: GitHub status, queue inspection, runner capacity, agent/VM logs, upgrades, and repair. Use when actuated jobs are not dispatching, runners are unreachable, or VMs fail immediately after launch.
---

# Actuated CLI diagnostic and repair agent

`actuated-cli` is the staff CLI for the actuated self-hosted GitHub Actions runner
platform. It talks to the actuated controller API to inspect queues, ping hosts,
fetch logs, upgrade agents, and force-repair stuck jobs.

Binary location: `~/.arkade/bin/actuated-cli` (arkade install, on the operator's workstation)
Default token: `~/.actuated/PAT` (set via `actuated-cli auth`)

## Golden rules

- Always use `--staff` for cross-customer visibility. Without it you only see
  your own org.
- Prefer table output for quick checks. Use `--json` only when you need to
  extract specific fields (status, uptime, imageRef, reachable, labels).
- **Never dump `--json` to context raw.** It produces 50-160 lines. Always pipe
  through `python3 -c` or `jq` to extract the fields you need.
- `agent-logs` and `logs` can produce hundreds of lines. Always use `--age` to
  bound the window (default 15m). For VM logs, use `--id` to target a specific
  VM. **`--age` and `--id` are mutually exclusive** — you can't use both.
- When a host is unreachable, `agent-logs` returns HTTP 500 with a nested error
  (503, 404, or connection refused). This confirms the host is down — don't
  keep retrying, move on.
- **OWNER comes from the runners table.** Each host has a `customer` field
  (e.g. `your-org`, `another-org`). Use this as `--owner` for agent-logs,
  logs, and upgrade commands on that host.
- After `upgrade --force` (Linux hosts), the host cycles: `running` → `starting`
  → `running`. During `starting` it won't accept jobs. Wait 1-2 minutes, then
  verify with `runners --staff` that status is `running` again. `base
  update-runner` on macOS marks the host draining immediately: it rejects new
  jobs, waits for in-flight VMs to finish, updates the base, then resumes
  acceptance (up to ~25min total). Update macOS hosts sequentially when
  continuous capacity is needed.
- After any repair, always verify: `jobs --staff` (queue cleared?) and
  `runners --staff` (hosts back to "running"?).
- Run independent checks in parallel: GitHub status, jobs, and runners can all
  run at the same turn.
- **stderr = failure, empty stdout = valid result.** If a command prints
  nothing to stdout but returns exit code 0, that's success (e.g., empty
  queue). Errors always go to stderr.

## The diagnostic workflow (follow in order)

Run Steps 1–3 in parallel — they are independent.

### Step 1 — Rule out GitHub-side issues

```bash
curl -s https://www.githubstatus.com/api/v2/status.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(d['status']['indicator'], d['status']['description'])
"
```

Expected: `none All Systems Operational`

If not, check specific components:
```bash
curl -s https://www.githubstatus.com/api/v2/components.json | python3 -c "
import json, sys
for c in json.load(sys.stdin).get('components', []):
    if c['name'] in ('Webhooks', 'API Requests', 'Actions'):
        print(f\"{c['name']}: {c['status']}\")
"
```

If any of these are degraded, that's the root cause. Stop and wait.

### Step 2 — Check queued jobs

```bash
# Quick table view (safe for context, 3-15 lines)
actuated-cli jobs --staff
```

If table shows queued jobs, get details:
```bash
actuated-cli jobs --staff --json | python3 -c "
import json, sys
jobs = json.load(sys.stdin)
print(f\"Queued: {len(jobs)}\")
for j in jobs:
    print(f\"  {j['owner']}/{j['repo']}: {j['job_name']} ({j['status']}) — labels: {j['labels']}\")
"
```

Note: `jobs --json` returns `[]` when queue is empty. This is normal, not an error.

### Step 3 — Check host capacity

```bash
# Quick table view (safe for context, 5-20 lines)
actuated-cli runners --staff
```

For structured data, parse selectively:
```bash
actuated-cli runners --staff --json | python3 -c "
import json, sys
for h in json.load(sys.stdin):
    status = h.get('status', 'unreachable')
    reachable = h.get('reachable', False)
    vms = f\"{h.get('vms',0)}/{h.get('maxVms',0)}\"
    uptime = f\"{h.get('uptimeNano',0)/1e9/60:.0f}m\" if reachable else '—'
    arch = h.get('arch') or '?'
    print(f\"  {h['name']:15s} {status:10s} {vms:6s} {uptime:8s} arch={arch:5s}\")
"
```

For VM image info (needed if diagnosing runner version issues):
```bash
actuated-cli runners --staff --json --images | python3 -c "
import json, sys
for h in json.load(sys.stdin):
    if h.get('reachable'):
        print(f\"{h['name']}: image={h.get('imageRef','unknown')}\")
"
```

What to look for:
- `unreachable` hosts — agent is down, tunnel is dead, or machine is off
- `0/20` VMs across all hosts while jobs are queued — dispatch is failing silently
- `draining` or `starting` status — host is in transition, won't accept jobs
- Architecture mismatch: jobs need `amd64` but only `arm64` hosts are up

### Decision point — what did you find?

| Situation | Next step |
|-----------|-----------|
| GitHub degraded | Stop — wait for GitHub to recover |
| All hosts unreachable | Check agent-logs (will fail 500), then SSH to hosts |
| Hosts running, 0 VMs, jobs queued | Go to Step 4 |
| Jobs queued, no matching arch/labels | Host configuration issue |
| Queue empty, all hosts running | System is healthy — no issue |

### Step 4 — Check agent logs on running hosts

The `--owner` value comes from the `customer` column in the runners table
(or `customer` field in JSON). Run for **each running host**:

```bash
# Safe for context (default 15m window, 0-30 lines)
actuated-cli agent-logs --staff --owner OWNER --age 15m HOSTNAME
```

**Warning:** `--age 24h` produces hundreds of lines of ACME certificate
renewal noise. Stick to `--age 15m` or `--age 1h` unless you specifically
need historical data. If you do need wider history, grep for key patterns:
`Launching`, `Exited`, `VM stopped`, `Upgrading`, `Agent ready`.

**Important:** Agent logs only tell you if VMs are *launching* on the host.
They do NOT tell you if the runner inside connected successfully or is
healthy. A VM can launch perfectly (agent sees "Started") but the runner
inside can fail immediately (e.g. deprecated version). You need Step 5 to
see what happens inside the VM.

Key patterns:
- `Launching with: XGB RAM Y vCPUs` — VM is being dispatched
- `VM stopped after: N seconds` — VM died (**if N < 30s, go to Step 5**)
- `Upgrading agent` / `Upgraded agent, restarting` — upgrade in progress
- `Agent ready to start accepting jobs` — agent is healthy
- `-- No entries --` — no recent activity on this host

If you see VMs dying in under 30 seconds, the problem is inside the VM.
Go to Step 5.

If all running hosts show `-- No entries --` and the queue is empty, the
system is healthy. No action needed.

### Step 5 — Check VM logs (the smoking gun)

**This is where you find out if the runner connected and is healthy.**
Agent logs can only tell you VMs launched. VM logs tell you whether the
runner registered, connected to GitHub, and picked up a job.

```bash
# All VMs on a host (use --age to bound, can be 50-200 lines)
actuated-cli logs --staff --owner OWNER --age 30m HOSTNAME

# Specific VM by hostname (--age and --id are mutually exclusive)
actuated-cli logs --staff --owner OWNER --id VMHOSTNAME HOSTNAME
```

Key patterns in VM logs:
- `Current runner version: 'X.Y.Z'` — note the version
- `Runner version vX.Y.Z is deprecated` — **ROOT CAUSE: runner too old**
- `Running job: job_name` — job is executing (good)
- `Job completed with result: Succeeded` — job finished (good)
- `Job completed with result: Failed` — job failed inside the runner
- `An error occurred:` — look at the error message

**Image SHA change:** The agent logs show snapshot SHA256 values. A change
in this value (e.g. `a1b2c3d4` → `e5f60718`) indicates a fresh image pull,
which usually means a new runner version is inside. Compare before/after
failed VMs to see if the image changed.

**Most common root cause:** deprecated runner version. The VM image
`ghcr.io/your-org/actuated-ubuntu22.04:x86_64-latest` may have an old
runner that GitHub rejects. The agent's cached image needs a fresh pull.

### Step 6 — Check controller logs (dispatch decisions)

**This is the single most valuable dispatch diagnostic.** Controller logs show
scheduler decisions, circuit breaker trips, and dispatch outcomes. Run this
when agent/VM logs don't reveal the cause.

```bash
# Controller logs (use --age to bound, can be 50-200 lines)
actuated-cli controller logs --staff --age 30m

# Recent controller logs (default 15m window, 0-30 lines)
actuated-cli controller logs --staff --age 15m
```

Key patterns:
- `Dispatched X to: HOST` — successful dispatch (good)
- `Dropped message for X/Y, no suitable hosts available` — no matching hosts
- `Delaying message, max concurrency reached` — at customer limit
- `Circuit breaker tripped for HOST` — agent unreachable, host skipped
- `Reaping VM X on HOST` — VM cleanup (expected)
- `Error getting capacity from HOST` — agent health check failed

If controller logs show "no suitable hosts" but runners are reachable, it's
likely a label or architecture mismatch. Check host labels vs job labels.

## Finding & debugging an out-of-date GitHub runner

**Runner ≠ agent.** On Linux hosts, the GitHub Actions *runner* is embedded
in the VM **image** (`ghcr.io/your-org/actuated-ubuntu22.04:*`), not in the
actuated agent binary itself. On macOS hosts the runner lives in the APFS
base bundle (see the macOS section below) — either way, `agent-logs`
(although the agent's systemd service / LaunchAgent) prints agent-level
activity; the agent SHA shown in `runners` says nothing about the runner
version inside the image. To inspect the runner you must look inside a VM
(Step 5). **On Linux, `upgrade --force` forces a re-pull of the OCI image
(the runner rootfs)** — the fix for a host serving a stale/degraded runner.
**On macOS, `~/.actuated/bin/agent base update-runner` is the equivalent.**
Always verify the runner version in VM logs after the upgrade rather than
assuming it moved.

Find the stale host (there is no direct "runner version" query):

**Linux hosts:**

1. Enumerate image refs per host using the **Step 3** `--json --images` snippet:
   `actuated-cli runners --staff --json --images` → read each reachable host's
   `imageRef`.
2. For each unique image, open one recent VM's log and read the runner version:
   ```bash
   actuated-cli logs --staff --owner OWNER --id <vm-hostname> HOST
   # look for: Current runner version: 'X.Y.Z'
   ```
3. Compare versions against what GitHub currently accepts (the newest
   `actions/runner` releases — a runner several releases behind is at risk of
   the "Runner version vX.Y.Z is deprecated … cannot receive messages" error).

**The stale-cache trap (Linux):** a host keeps serving a *cached* old image
digest even after a fixed image is rebuilt and re-published. Watch the agent
log snapshot SHA256 for each launch (`Preparing snapshot via sha256:<digest>`).
If the digest stays constant across days (e.g. stuck at `a1b2c3d4…`) the host
is pinned to the old image and still serves the old runner. Rebuilding/publishing
the image alone does not fix that host — run `upgrade --force` on it to force
the OCI re-pull. **Always re-verify** via that VM's log that the runner version
actually moved after the fix (the snapshot SHA should also change).

**macOS hosts:** identify the stale runner the same way (read one recent VM's
log for `Current runner version: 'X.Y.Z'`), then fix on the host:

```bash
~/.actuated/bin/agent base update-runner --url http://127.0.0.1:8080
# wait for: Runner update complete, accepting jobs   (in ~/.actuated/logs/agent-daemon.log)
```

Then trigger a fresh VM and re-read `Current runner version` from its log to
confirm the fix took.

## macOS hosts: two update paths, neither is `upgrade --force`

`actuated-cli upgrade --force` speaks the Linux agent's protocol. The macOS
agent (repo: `openfaasltd/actuated-macos`, runs under launchd as
`com.openfaas.actuated-agent`, GUI login session required) does not accept
it. Two distinct parts keep a macOS host current, and each has its own
command run **on the host over ssh** — they hit the agent's local API
directly, not the controller:

### Part 1 — agent binary self-update

Two ways to get the new binary running. Either:

```bash
# Option A: swap + explicit restart (kickstart does NOT drain; only use when 0 VMs)
# Prerequisite: ssh as the macOS user who installed the agent — the LaunchAgent
# lives in that user's GUI domain, so $(id -u) must be that user's uid.
~/.actuated/bin/agent upgrade    # pulls ghcr.io/openfaasltd/actuated-agent-macos:latest, swaps binary in place
launchctl kickstart -k "gui/$(id -u)/com.openfaas.actuated-agent"   # restart to run it
```

`agent upgrade` only replaces the on-disk binary — the running process
keeps the old version until restart.

Or ask the running agent to drain, swap and exit itself — launchd
KeepAlive restarts it. This goes through the agent's `POST /api/upgrade`
endpoint (bearer-token authed; drain VMs, swap, exit) and is the safe
choice when jobs may be running:

```bash
TOKEN=$(cat ~/.actuated/TOKEN)
curl --fail-with-body --show-error --silent -X POST \
  -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8080/api/upgrade
# note the VERSION (SHA) from `actuated-cli runners --staff` first; after
# the launchd restart, confirm the SHA changed before considering it done
```

Either way, verify afterwards: `actuated-cli runners --staff` shows the
new `VERSION` (SHA) and the agent log banner shows `Version: vX.Y.Z
SHA: ...`.

### Part 2 — base-image runner + guest-agent update

The GitHub Actions runner and the guest-agent live inside the sealed base
bundle (the `--base` arg in the launchd plist, e.g. `base-xcode.bundle`),
which is NOT refreshed by Part 1. Update it with:

```bash
~/.actuated/bin/agent base update-runner --url http://127.0.0.1:8080
```

This asks the running agent to: drain jobs, resolve the latest
`actions/runner` release, clone the base APFS disk, install the runner +
embedded guest-agent into the clone, then promote it on clean detach.
The original base is untouched on any failure. Progress is in
`~/.actuated/logs/agent-daemon.log` (the command itself is quiet); the
log line `Runner update complete, accepting jobs` means done. Takes
1-25min depending on download. A launchd scheduled variant can do this
daily (`--schedule-runner-updates` at install time; manage via
`agent service status runner-update`).

### macOS memory reporting

The agent's FREE RAM = `vm_stat` free + inactive + speculative pages — a
conservative floor, not Linux-style `MemAvailable`. Low FREE RAM on an
idle macOS host usually means app/GUI state or iCloud sync daemons, not
a bug.

## Repair actions (Linux) and re-queueing (both platforms)

### Force upgrade a host (pulls fresh VM image)

```bash
actuated-cli upgrade --staff --owner OWNER --force HOSTNAME
```

This restarts the agent and pulls the latest VM image. The host will cycle:
`running` → `starting` → `running`. During `starting` it won't accept jobs.
The `--force` flag ensures a fresh pull even if the agent thinks it's up to date.

Wait 1-2 minutes, then verify the host is back:
```bash
actuated-cli runners --staff --json | python3 -c "
import json, sys
for h in json.load(sys.stdin):
    if h['name'] == 'HOSTNAME':
        print(f\"Status: {h.get('status')}, Uptime: {h.get('uptimeNano',0)/1e9/60:.0f}m\")
"
```

Only proceed to `repair` once status is `running` again.

### Upgrade all hosts at once

```bash
# Upgrade all hosts for a customer
actuated-cli upgrade --staff --owner OWNER --all --force
```

### Repair stuck jobs (re-queue with fresh VMs)

```bash
actuated-cli repair --staff OWNER
```

This takes a **customer OWNER** (e.g. `your-org`), not a hostname. It forces
the controller to re-queue stuck jobs and launch fresh VMs.

**CRITICAL ordering:** Only run `repair` AFTER fixing the root cause. If the
runner version is deprecated: on Linux run `upgrade --force` first; on macOS
run `~/.actuated/bin/agent base update-runner` on the host and wait for
`Runner update complete, accepting jobs` in the agent log. Wait for the host
to return to `running`, THEN run `repair`. Running repair before fixing the
cause just creates more failing VMs.

Output: `Repair requested for OWNER, status: 202` / `Requeued VMs: N`
(Note: this is the expected output format not guaranteed to be exact -- confirm against the live output.)

After repair, wait for the ETA countdowns to complete, then verify:
```bash
actuated-cli jobs --staff   # should show empty queue
```

### Restart an unresponsive agent

```bash
# Graceful restart (drains running VMs first)
actuated-cli restart --staff --owner OWNER HOSTNAME

# Force reboot (use when agent is completely unresponsive)
actuated-cli restart --staff --owner OWNER --reboot HOSTNAME
```

### Disable a host (decommission)

```bash
actuated-cli disable --staff --owner OWNER HOSTNAME
```

Can only be re-enabled via SSH: `systemctl enable actuated`

## Output size guide

| Command | Typical lines | Context safe? |
|---------|--------------|---------------|
| `jobs --staff` (table) | 3-15 | Yes |
| `jobs --staff --json` | 1-50+ | Parse with python3/jq |
| `runners --staff` (table) | 5-20 | Yes |
| `runners --staff --json` | 50-150 | Parse with python3/jq |
| `runners --staff --json --images` | 60-160 | Parse with python3/jq |
| `agent-logs --age 15m` | 0-30 | Yes (bounded) |
| `agent-logs --age 1h` | 0-200 | Use cautiously |
| `logs --age 30m` (all VMs) | 0-500+ | Use `--id` to target |
| `logs --id VM` | 0-100 | Yes (single VM, no --age) |
| `version` | 12 | Yes |
| `metering --id VM` | 20-50 | Yes |
| `increases OWNER --days 30` | 10-30 | Yes (--json broken, use table) |

## JSON parsing patterns

Always pipe JSON output through a filter. Never dump raw. Reusable snippets
already shown in the workflow above: **host summary** and **VM images** (Step 3),
**queued-jobs summary** (Step 2). The one extra pattern you'll want:

```bash
# Unreachable hosts only
actuated-cli runners --staff --json | python3 -c "
import json, sys
for h in json.load(sys.stdin):
    if not h.get('reachable'):
        print(f\"UNREACHABLE: {h['name']} ({h['customer']})\")
"
```

## Decision tree (symptom → action)

Follow this after the diagnostic workflow:

1. **GitHub status not operational?**
   - Wait for GitHub to recover. No local action will help.

2. **All hosts unreachable?**
   - Linux: SSH to the machines, check `systemctl status actuated`
     - Restart: `actuated-cli restart --staff --owner OWNER HOSTNAME`
     - If still unresponsive: `actuated-cli restart --staff --owner OWNER --reboot HOSTNAME`
   - macOS: SSH as the user who installed the agent (its LaunchAgent lives in
     that user's GUI domain), then restart with
     `launchctl kickstart -k "gui/$(id -u)/com.openfaas.actuated-agent"`
     (GUI login session must exist)

3. **Jobs queued, hosts running, 0 VMs, VMs dying in <30s?**
   - Check VM logs → look for `deprecated runner version`
   - Linux host: `upgrade --force` on the host, wait for `running`, then
     `repair --staff OWNER`
   - macOS host: run `~/.actuated/bin/agent base update-runner` on the host
     (see the macOS section above), then `repair --staff OWNER`

4. **Jobs queued, hosts running, 0 VMs, no VM launches in agent logs?**
   - Check controller logs for "no suitable hosts" or "customer not found"
   - May be a configuration issue (labels, customer config, NATS)

5. **Jobs queued, host in `starting` status?**
   - Wait 1-2 minutes, then re-check

6. **Jobs queued, no matching architecture?**
   - Jobs need `amd64` but only `arm64` hosts are up (or vice versa)
   - Bring up a matching host or the jobs can't run

7. **Jobs in_progress but not completing?**
   - This is often a GitHub API inconsistency — the job may have finished
     but GitHub hasn't sent the completion webhook
   - Check the job URL in GitHub directly

## Common failure modes

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Jobs queued, 0 VMs, host reachable | Deprecated runner in cached VM image | Linux: `upgrade --force` + `repair` · macOS: `agent base update-runner` on host + `repair` |
| All hosts unreachable | Agents down, tunnels dead | SSH, `systemctl restart actuated` (macOS: `launchctl kickstart -k` the service) |
| Host `draining` | Being decommissioned | Wait or `repair` to force |
| Host `starting` | Agent just restarted | Wait 1-2 minutes |
| VMs die in 7s | Runner version deprecated by GitHub | Linux: `upgrade --force` to pull fresh image · macOS: `agent base update-runner` |
| Runner registers, no jobs | Runner can't connect to GitHub | Check GitHub status, network |
| No matching architecture | Jobs need amd64, only arm64 up | Bring up matching host |
| All hosts at 20/20 VMs | Concurrency limit reached | Wait for jobs to complete |

## Quick reference

| Need | Command |
|------|---------|
| Check queue | `actuated-cli jobs --staff` |
| Check hosts | `actuated-cli runners --staff` |
| Agent logs | `actuated-cli agent-logs --staff --owner OWNER --age 15m HOST` |
| VM logs | `actuated-cli logs --staff --owner OWNER --age 30m HOST` |
| Specific VM logs | `actuated-cli logs --staff --owner OWNER --id VMHOST HOST` |
| Force upgrade (Linux) | `actuated-cli upgrade --staff --owner OWNER --force HOST` |
| Repair queue | `actuated-cli repair --staff OWNER` |
| Restart agent | `actuated-cli restart --staff --owner OWNER HOST` |
| Reboot host | `actuated-cli restart --staff --owner OWNER --reboot HOST` |
| VM image info | `actuated-cli runners --staff --json --images` |
| Host capacity JSON | `actuated-cli runners --staff --json` |
| Queue JSON | `actuated-cli jobs --staff --json` |
| Metering | `actuated-cli metering --staff --owner OWNER --id VMHOST HOST` |
| Job trends | `actuated-cli increases --staff OWNER --days 30` |
| SSH into host | `actuated-cli ssh connect HOST` |