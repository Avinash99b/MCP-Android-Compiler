# MCP-Android-Compiler

Turn a **free, disposable GitHub Actions runner into a remote MCP shell** you can drive from
anywhere — built for compiling Android projects without owning a build box.

Transport is **LinkPulse** (self-hosted tunnel, `my_proxy` client) with a **fixed subdomain**, and
the MCP server runs with **`--no-auth`** by design.

Feasibility was verified end-to-end before this repo existed: a public tunnel URL carried a real MCP
session (`initialize` → `tools/list` → `tools/call run_shell_command`) and executed a command on the
remote host. The LinkPulse path used here was also verified separately — env-var auth, tunnel
registration, live public fetch, and teardown.

On top of that, **this workflow has already run for real**: job `132484064233`
(run `43607121046`) cloned the server, started it on the runner and logged
`local initialize -> HTTP 200`. Both tunnel backends have therefore been exercised on an actual
GitHub-hosted runner — the LinkPulse version additionally verifies the *public* URL before reporting
success.

## Required secrets

Set these in **Settings → Secrets and variables → Actions**. Nothing secret lives in this repo.

| Secret | Required | Value |
|---|---|---|
| `LINKPULSE_TOKEN` | **yes** | Shared secret of the LinkPulse control server |
| `LINKPULSE_SERVER` | **yes** | Control server as `host:port`, e.g. `tunnel.example.com:9000`. The public domain is derived from the host part, so no hostname is committed here. |
| `MACRODROID_WEBHOOK_URL` | no | MacroDroid trigger URL; receives the public URL when the runner is ready |

```bash
# both are required; the token is never echoed
gh secret set LINKPULSE_TOKEN  --repo Avinash99b/MCP-Android-Compiler
gh secret set LINKPULSE_SERVER --repo Avinash99b/MCP-Android-Compiler   # paste host:port
gh secret set MACRODROID_WEBHOOK_URL --repo Avinash99b/MCP-Android-Compiler   # optional
```

No hostname, URL, token or key is stored in this repository — only the `subdomain` label. The public
URL is assembled at runtime from `LINKPULSE_SERVER` + the `subdomain` input.

## How it works

```
 your MCP client  ──HTTPS──►  https://<subdomain>.<your-linkpulse-domain>/mcp
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
   - `subdomain` — default `mcp-android-compiler` → `https://mcp-android-compiler.<your-linkpulse-domain>`
   - `notify_phone` — push the URL to your phone
2. The workflow verifies the **public** endpoint with a real MCP `initialize` before declaring
   success, then idles.
3. Point your MCP client at it.

```json
{
  "mcpServers": {
    "actions-runner": {
      "type": "http",
      "url": "https://mcp-android-compiler.<your-linkpulse-domain>/mcp"
    }
  }
}
```

No auth header — `--no-auth` is intentional.

Smoke test:

```bash
curl -sS -X POST https://mcp-android-compiler.<your-linkpulse-domain>/mcp \
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
java -version                               # careful: see the JDK warning below
echo "$ANDROID_HOME" "$ANDROID_NDK_LATEST_HOME"
export JAVA_HOME="$JAVA_HOME_17_X64"        # AGP is happiest on 17
./gradlew :app:assembleDebug --no-daemon    # --no-daemon is usually faster in a one-shot VM
```

**Toolchain actually on `ubuntu-latest` today** (verified against a real run on 2026-11-14, image
`ubuntu26 / 20261111.196.1` — `ubuntu-latest` is now **Ubuntu 26.04**, no longer 24.04):

| | |
|---|---|
| Android cmdline-tools | 20.0 |
| Build-tools | 37.0.0 / 36.0.0 / 36.1.0 / 35.0.0 / 35.0.1 / 34.0.0 |
| Platforms | android-34 → android-37.2 (incl. beta) |
| NDK | 27.3.13750724 (default) · 29.0.14206865 · 30.0.16248370 |
| Gradle | 9.9.1 |
| JDK | **11, 17, 21, 25 — 25 is the DEFAULT, and JDK 8 is gone** |
| CMake | 3.31.5 / 4.1.2 |

> ⚠️ **JDK warning.** The default `java` on `ubuntu-latest` is now **25**, which many Android
> Gradle Plugin versions do not support. Set `export JAVA_HOME="$JAVA_HOME_17_X64"` before building,
> or pin `runs-on: ubuntu-24.04` if you need the old JDK-8/17-default image.

A cold `~/.gradle` plus intermediates can exhaust 14 GB. Keep the preinstalled SDK, and consider
`actions/cache` on `~/.gradle/caches` (10 GB per repo).

### Measured on a live job (not from docs)

Verified by driving a running job over MCP on 2026-11-14 (run `43610118300`):

| | |
|---|---|
| OS | **Ubuntu 26.04.1 LTS** (`ubuntu-latest` is no longer 24.04) |
| CPU / RAM | 4 vCPU / ~15.6 GiB |
| Disk | **145 GB total, ~88 GB free** — the docs' "14 GB SSD" is outdated |
| `/dev/kvm` | **present** → Android emulator hardware acceleration works |
| Default JDK | **25** (11/17/21 also installed; JDK 8 is gone) |
| Gradle | 9.9.1 |
| `ANDROID_HOME` | `/usr/local/lib/android/sdk`, platforms through `android-37.2` |
| User | `runner`, passwordless `sudo` |

Because `/dev/kvm` is available, instrumented tests (`connectedAndroidTest`) are viable here — not just
`assembleDebug`.

## Limits

| Limit | Value |
|---|---|
| Job wall-clock | **6 h hard cap** (workflow uses 355 min) |
| Runner | 4 vCPU / ~16 GB RAM / **145 GB disk (~88 GB free, measured)**, fresh VM per job |
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
