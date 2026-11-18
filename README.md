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
| `SSH_PRIVATE_KEY` | no | **Unencrypted** OpenSSH/PEM private key. Written to the runner's `~/.ssh` so this workflow *and* the interactive shell authenticate to GitHub over SSH (`git@github.com:`). Needs no `contents: write` token scope. |

```bash
# both are required; the token is never echoed
gh secret set LINKPULSE_TOKEN  --repo Avinash99b/MCP-Android-Compiler
gh secret set LINKPULSE_SERVER --repo Avinash99b/MCP-Android-Compiler   # paste host:port
gh secret set MACRODROID_WEBHOOK_URL --repo Avinash99b/MCP-Android-Compiler   # optional
gh secret set SSH_PRIVATE_KEY --repo Avinash99b/MCP-Android-Compiler < ~/.ssh/mcp_runner   # optional
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
                    4 vCPU / 16 GB RAM / 145 GB disk + Android SDK/NDK
```

GitHub-hosted runners have **no inbound network endpoint**, so an outbound tunnel is mandatory.
Unlike a cloudflared quick tunnel, LinkPulse gives you a **stable hostname**, so your MCP client
config does not change between runs.

## Quickstart

1. **Actions → remote-shell-mcp → Run workflow**
   - `duration_minutes` — how long to keep the runner (max 350; GitHub kills hosted jobs at 360)
   - `subdomain` — default `mcp-android-compiler` → `https://mcp-android-compiler.<your-linkpulse-domain>`
   - `notify_phone` — push the URL to your phone
   - `git_ssh_rewrite` — default `true`; rewrite `https://github.com/` remotes to `git@github.com:`
     so every git command in the shell goes over SSH. Ignored when no key is installed.
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

> **`git_push` and `git clone` now work without any token scope** when `SSH_PRIVATE_KEY` is set:
> the runner authenticates over SSH, and `permissions: {}` still means the shell never holds a
> GitHub *token*. If you would rather push with a token instead, add `contents: write` to the
> workflow and the `GITHUB_TOKEN` becomes usable - but then the no-auth shell holds that too.

## GitHub credentials inside the shell (SSH)

Set `SSH_PRIVATE_KEY` and the workflow step **Install GitHub SSH key (optional)** runs before
anything else touches the network:

1. normalises the secret (single-line values with literal `\n`, CRLF, or stray quotes all work);
2. names the file after the key type it detects — `~/.ssh/id_ed25519`, `id_rsa` or `id_ecdsa`, mode `600`;
3. pins `github.com` host keys from `https://api.github.com/meta` and checks the well-known ED25519
   fingerprint (`SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU`), falling back to `ssh-keyscan`
   with a warning if the fetch fails — never blind trust-on-first-use;
4. writes `~/.ssh/config` with `IdentitiesOnly yes` and absolute paths, because OpenSSH resolves `~`
   from the passwd database and not from `$HOME`;
5. proves the key with a real `ssh -T git@github.com` — pinned to *that* key via
   `-F /dev/null -o IdentitiesOnly=yes -i <key>`, so no ambient key can vouch for it;
6. sets `git config --global user.name/user.email` to the authenticated account, so `git commit`
   works in the shell without "Author identity unknown".

The result is reported in the job summary: authenticated account, key type, fingerprint, host-key
source, and whether the rewrite is active.

Make a dedicated key (don't ship your laptop's primary identity to an unauthenticated runner):

```bash
ssh-keygen -t ed25519 -C "mcp-android-compiler" -f ~/.ssh/mcp_runner -N ''
# add ~/.ssh/mcp_runner.pub  -> GitHub Settings > SSH keys
# or, safer: repo Settings > Deploy keys (tick "Allow write access")
gh secret set SSH_PRIVATE_KEY < ~/.ssh/mcp_runner
```

Then, from inside the shell:

```bash
git clone git@github.com:Avinash99b/SomeRepo.git   # works as-is via ~/.ssh/config
cd SomeRepo && git commit -am "..." && git push     # commits get the right identity
```

Notes and failure modes:

- **Never fatal.** No key, a `.pub` pasted by mistake, a corrupt blob or a rejected key all log a
  warning/error annotation and continue over HTTPS. The summary says `authenticated: false` with the reason.
- **The step clears errexit deliberately** (`set +e`). GitHub runs `run:` scripts as
  `bash --noprofile --norc -eo pipefail`, and `-e` inherited from the command line is *not*
  cancelled by a later `set -u`. The first version of this step omitted that line and failed the
  job at `AUTH="$(ssh -T git@github.com)"` — because **`ssh -T` exits 1 even when it
  authenticates successfully** (GitHub returns non-zero since it grants no shell). If you edit
  this step, test it as `bash -eo pipefail`, not plain `bash`.
- **Passphrase-protected keys are refused by design** — a hosted runner has nowhere to type the
  passphrase. The step says so explicitly instead of hanging. Strip it with `ssh-keygen -p`.
- A **deploy key** is detected from the `Hi OWNER/REPO!` banner; because it only grants access to one
  repository, the https→ssh rewrite is forced **off** automatically so unrelated clones keep working.
- The workflow's own clone of `Remote-Shell-MCP` tries SSH first (proving the key end-to-end) and
  falls back to anonymous HTTPS with `GIT_CONFIG_GLOBAL=/dev/null`, which neutralises the rewrite so
  the fallback cannot be re-routed back onto the broken SSH path.

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
df -h .                                     # 145 GB, ~88 GB free — plenty of room
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

A cold `~/.gradle` plus intermediates fits easily in ~88 GB of free space. Keep the preinstalled SDK, and consider
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
- **If `SSH_PRIVATE_KEY` is set, the shell also holds a GitHub SSH identity.** Anyone reaching the
  URL can `git push` as that key. That is why the key should be dedicated (deploy key or a
  throwaway account), narrowly scoped, and rotated after use — the token-less `permissions: {}`
  guarantee does not cover it.
- Stop the runner when you are done (**Actions → Cancel workflow**); the URL 404s as soon as the
  client disconnects.

## Files

- `.github/workflows/remote-shell-mcp.yml` — the runner workflow
- Run it with **Actions → remote-shell-mcp → Run workflow**
