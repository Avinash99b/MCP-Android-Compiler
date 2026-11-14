# MCP-Android-Compiler

Turn a **free, disposable GitHub Actions runner into a remote MCP shell** you can drive from
anywhere — built for compiling Android projects without owning a build box.

Transport is **LinkPulse** (self-hosted tunnel, `my_proxy` client) with a **fixed subdomain**, and
the MCP server runs with **`--no-auth`** by design.

Feasibility was verified end-to-end before this repo existed: a public tunnel URL carried a real MCP
session (`initialize` → `tools/list` → `tools/call run_shell_command`) and executed a command on the
remote host. The LinkPulse path used here was also verified separately — env-var auth, tunnel
registration, live public fetch, and teardown.

## Required secrets

Set these in **Settings → Secrets and variables → Actions**. Nothing secret lives in this repo.

| Secret | Required | Value |
|---|---|---|
| `LINKPULSE_TOKEN` | **yes** | The shared secret from `~/.config/linkpulse/authtoken.json` (`"token"` field) |
| `LINKPULSE_SERVER` | no | Control server `host:port`. Default `<linkpulse-host>:9000` |
| `MACRODROID_WEBHOOK_URL` | no | MacroDroid trigger URL; receives the public URL when the runner is ready |

```bash
# set them (paste the token when prompted - it is never echoed)
gh secret set LINKPULSE_TOKEN  --repo Avinash99b/MCP-Android-Compiler
gh secret set LINKPULSE_SERVER --repo Avinash99b/MCP-Android-Compiler --body '<linkpulse-host>:9000'
gh secret set MACRODROID_WEBHOOK_URL --repo Avinash99b/MCP-Android-Compiler
```

You do **not** need `LINKPULSE_SERVER` — it defaults in the workflow.

## How it works

```
 your MCP client  ──HTTPS──►  https://<subdomain>.<linkpulse-host>/mcp
                                          │
                        LinkPulse public edge (your own server)
                                          │
                        multiplexed, authenticated control connection (:9000)
                                          │
                        my_proxy client  (on the runner, outbound only)
                                          │
                        Remote-Shell-MCP  (127.0.0.1:8765, --no-auth)
                                          │
                              GitHub Actions runner
                    4 vCPU / 16 GB RAM / 14 GB SSD + Android SDK/NDK
```

GitHub-hosted runners have **no inbound network endpoint**, so an outbound tunnel is mandatory.
Unlike a cloudflared quick tunnel, LinkPulse gives you a **stable hostname**, so your MCP client
config does not change between runs.

## Quickstart

1. **Actions → remote-shell-mcp → Run workflow**
   - `duration_minutes` — how long to keep the runner (max 350; GitHub kills hosted jobs at 360)
   - `subdomain` — default `mcp-android-compiler` → `https://mcp-android-compiler.<your-domain>`
   - `notify_phone` — push the URL to your phone
2. The workflow verifies the **public** endpoint with a real MCP `initialize` before declaring
   success, then idles.
3. Point your MCP client at it.

```json
{
  "mcpServers": {
    "actions-runner": {
      "type": "http",
      "url": "https://mcp-android-compiler.<linkpulse-host>/mcp"
    }
  }
}
```

No auth header — `--no-auth` is intentional.

Smoke test:

```bash
curl -sS -X POST https://mcp-android-compiler.<linkpulse-host>/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"cli","version":"1"}}}'
# 200 + `event: message` = live
# 404 "No tunnel registered for this host" = the runner isn't up (or holds a different subdomain)
# 421 = Host allowlist mismatch (the workflow sets --allowed-host automatically)
```

## Tools exposed

`run_shell_command` · `get_job` · `list_jobs` · `terminate_job` · `cleanup_jobs` ·
`get_diagnostics` · `upload_file` · `download_file` · `begin_upload` · `upload_chunk` ·
`finish_upload` · `abort_upload` · `file_sha256` · `git_push` · `screenshot`

Commands run as **background jobs**, so a long Gradle build survives client disconnects — start it,
close the laptop, poll `get_job` later.

> `git_push` will not work as written: the workflow sets `permissions: {}`. Add `contents: write`
> to the workflow if you want the runner to push.

## Subdomain notes

- The subdomain is a **lease held by one client at a time**. Two runners (or a runner plus your
  laptop) requesting the same name collide; the second gets
  `Tunnel <id> failed: subdomain '...' is already in use`.
- LinkPulse **does not exit** when a subdomain is refused — it logs the error and stays alive. That
  is why this workflow verifies the *public URL* instead of trusting the process.
- The workflow retries 3× with a 10 s gap (stale leases from killed jobs can linger), then fails
  with an actionable `::error::` telling you to pick another subdomain.
- `concurrency: remote-shell-mcp` keeps exactly one runner per repo so the lease is never
  self-contended.

## Android build tips

```bash
df -h .                                     # 14 GB SSD — check before every serious build
java -version                               # JDK 17 default; 8/11/21/25 also installed
echo "$ANDROID_HOME" "$ANDROID_NDK_LATEST_HOME"
./gradlew :app:assembleDebug --no-daemon    # --no-daemon is usually faster in a one-shot VM
```

Preinstalled on `ubuntu-latest`: Android cmdline-tools 12.0, build-tools 34–37, platforms
android-34 → android-37.2, NDK 27.3 / 29.0 / 30.0, Gradle 9.9.1, JDK 8/11/17/21/25, plus KVM
hardware acceleration for the emulator. Most builds need **no SDK download**.

A cold `~/.gradle` plus intermediates can exhaust 14 GB. Keep the preinstalled SDK, and consider
`actions/cache` on `~/.gradle/caches` (10 GB per repo).

## Limits

| Limit | Value |
|---|---|
| Job wall-clock | **6 h hard cap** (workflow uses 355 min) |
| Runner | 4 vCPU / 16 GB RAM / 14 GB SSD, fresh VM per job |
| Persistence | **None** — disk, caches, working tree destroyed at job end |
| Inbound networking | **None** — tunnel required |
| Artifacts | 500 MB on GitHub Free |
| Cache | 10 GB per repository |

Need persistence? Use **GitHub Codespaces** (personal free tier: 120 core-hours + 15 GB-month,
state survives stop/start) or a **self-hosted runner** (5-day jobs).

## Compliance

GitHub's *Terms for Additional Products and Features* prohibit using GitHub-hosted runners for
"any other activity unrelated to the production, testing, deployment, or publication of the software
project associated with the repository where GitHub Actions are used."

Keep the Android sources you compile **in this repository** and the workload stays on the right side
of that line.

## Security

The endpoint is **unauthenticated on a fixed, guessable hostname** — that is a deliberate choice for
convenience, not an oversight. Consequences to accept:

- Anyone who learns the URL gets a shell on the runner, with passwordless `sudo`.
- The URL is stable across runs, so it does not need to be *discovered* — only guessed/known.
- Mitigations that remain: `permissions: {}` (the shell holds no GitHub token), loopback-only bind,
  `--allowed-host` Host-header allowlist, 355-minute self-destruct, one-run concurrency.
- Stop the runner when you are done (**Actions → Cancel workflow**); the URL 404s as soon as the
  client disconnects.

## Files

- `.github/workflows/remote-shell-mcp.yml` — the runner workflow
- Run it with **Actions → remote-shell-mcp → Run workflow**
