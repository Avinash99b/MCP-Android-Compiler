# MCP-Android-Compiler

Turn a **free, disposable GitHub Actions runner into a remote MCP shell** you can drive from
anywhere — built for compiling Android projects without owning a build box.

Feasibility was verified end-to-end before this repo existed: a public tunnel URL carried a real
MCP session (`initialize` → `tools/list` → `tools/call run_shell_command`), executed a command on
the remote host, and rejected an unauthenticated request with HTTP 401.

## How it works

```
 your MCP client  ──HTTPS──►  https://<random>.trycloudflare.com/mcp
                                        │
                        (outbound-only cloudflared tunnel from the runner)
                                        │
                        Remote-Shell-MCP  (127.0.0.1:8765, bearer token)
                                        │
                              GitHub Actions runner
                        4 vCPU / 16 GB RAM / 14 GB SSD + Android SDK/NDK
```

GitHub-hosted runners have **no inbound network endpoint at all**, so an outbound tunnel is
mandatory. The server is bound to localhost and protected by a per-run random token.

## Quickstart

1. **Actions → remote-shell-mcp → Run workflow** — set `duration_minutes` (max 350) and leave
   `notify_phone` on.
2. Within ~60–90 s your phone (if the `MACRODROID_WEBHOOK_URL` secret is set) and the job summary
   receive the tunnel URL plus a random token.
3. Point your MCP client at it.

**Native HTTP client** (Claude Desktop / VS Code / Cursor / Cline):

```json
{
  "mcpServers": {
    "actions-runner": {
      "type": "http",
      "url": "https://<random>.trycloudflare.com/mcp",
      "headers": { "Authorization": "Bearer <MCP_TOKEN>" }
    }
  }
}
```

**stdio-only client** — bridge it with `mcp-remote` (`--header-file` keeps the token off the
process list):

```json
{
  "mcpServers": {
    "actions-runner": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://<random>.trycloudflare.com/mcp",
               "--header", "Authorization:Bearer <MCP_TOKEN>"]
    }
  }
}
```

**Smoke test from a terminal:**

```bash
URL=https://<random>.trycloudflare.com
curl -sS -X POST "$URL/mcp" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"cli","version":"1"}}}'
# 200 + `event: message` = good · 401 = bad/missing token · 421 = Host allowlist mismatch
```

## Tools exposed

`run_shell_command` · `get_job` · `list_jobs` · `terminate_job` · `cleanup_jobs` ·
`get_diagnostics` · `upload_file` · `download_file` · `begin_upload` · `upload_chunk` ·
`finish_upload` · `abort_upload` · `file_sha256` · `git_push` · `screenshot`

Commands run as **background jobs**, so long builds survive client disconnects — you can start a
Gradle build, close the laptop, and poll `get_job` later.

## Android build tips

```bash
df -h .                                    # 14 GB SSD — check before every serious build
java -version                              # JDK 17 default; 8/11/21/25 also installed
echo "$ANDROID_HOME" "$ANDROID_NDK_LATEST_HOME"
./gradlew :app:assembleDebug --no-daemon    # --no-daemon is usually faster in a one-shot VM
```

Preinstalled on `ubuntu-latest`: Android cmdline-tools 12.0, build-tools 34–37, platforms
android-34 → android-37.2, NDK 27.3 / 29.0 / 30.0, Gradle 9.9.1, JDK 8/11/17/21/25, plus KVM
hardware acceleration for the emulator. Most builds need **no SDK download**.

Watch the disk: a cold `~/.gradle` plus intermediates can exhaust 14 GB. Keep the preinstalled SDK
(don't re-download), and consider `actions/cache` on `~/.gradle/caches` (10 GB per repo).

## Limits you should design around

| Limit | Value |
|---|---|
| Job wall-clock | **6 h hard cap** (workflow uses 355 min) |
| Runner | 4 vCPU / 16 GB RAM / 14 GB SSD, fresh VM per job |
| Persistence | **None** — disk, caches, working tree all destroyed at job end |
| Inbound networking | **None** — tunnel required |
| URL/token | New random pair on every run |
| Artifacts | 500 MB on GitHub Free |
| Cache | 10 GB per repository |

If you need a persistent box with a stable address, use **GitHub Codespaces** (personal free
tier: 120 core-hours + 15 GB-month, state survives stop/start) or a **self-hosted runner**.

## Compliance

GitHub's *Terms for Additional Products and Features* prohibit using GitHub-hosted runners for
"any other activity unrelated to the production, testing, deployment, or publication of the
software project associated with the repository where GitHub Actions are used."

Keep the Android sources you compile **in this repository** and the workload stays on the right
side of that line. Do not repurpose this workflow as a general-purpose cloud VM for unrelated
projects, and prefer short, attributable build sessions over hours-long idle shells.

## Security

This publishes a shell on the public internet. The workflow mitigates:

- random 48-hex-char token per run, masked via `::add-mask::`
- localhost-only bind (`127.0.0.1`)
- `--allowed-host <tunnel-host>` Host-header allowlist
- `permissions: {}` so the shell cannot use `GITHUB_TOKEN`
- `timeout-minutes: 355` self-destruct, single-run concurrency

Still: anyone who obtains **both** the URL and the token gets arbitrary command execution on the
runner with passwordless `sudo`. Treat both as secrets, never reuse a token, and delete the
workflow run logs/artifacts when you don't need them.

## Files

- `.github/workflows/remote-shell-mcp.yml` — the runner workflow
- Run it with **Actions → remote-shell-mcp → Run workflow**
