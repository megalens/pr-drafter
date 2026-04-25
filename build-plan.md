# PR Description Drafter — V1 Engineering Build Plan

**Document status:** Draft for implementation  
**Audience:** Single developer, self-hosted, local-first  
**Repository root (this project):** `/root/pr-drafter`  
**Runtime target:** Linux (systemd), macOS feasible with launchd notes only where different  

---

## 0. Product definition (normative)

**Goal:** A long-running local process (“daemon”) that detects new commits on tracked local branches, aggregates `git` context (messages + diff), merges template/voice from repo and global files, calls an LLM **only with that assembled text**, and creates a **draft** pull request on GitHub via HTTPS API. The PR must never be opened for review, auto-assigned, or merged by this tool—only `draft: true` creation and optional subsequent `PATCH` of the same PR’s `body` if re-drafting the same branch tip is supported in V1 (optional; see §8).

**Trust posture:** All repository content, tokens, and LLM prompts stay on the host. The only intentional egress is TLS to (a) the LLM provider API and (b) `api.github.com` (or GitHub Enterprise host), carrying the generated title/body and OAuth token.

**Non-goals (V1):** Multi-user tenancy, org-wide policies, CI integration, code review automation, merge queue, GitLab/Azure DevOps, mobile clients, web UI beyond optional localhost health endpoint.

---

## 1. Architecture

### 1.1 Components

| Component | Responsibility | Suggested implementation |
|-----------|----------------|---------------------------|
| **Supervisor / daemon** | Process lifecycle, signal handling (`SIGTERM`, `SIGHUP` for reload), single-instance lock | Rust (`tokio`) or Go; lock file `~/.config/pr-drafter/pr-drafter.pid` + `flock` on Unix |
| **Repo registry** | Map filesystem paths → repo config, enabled flag, remote name, base branch | YAML/JSON under `~/.config/pr-drafter/config.yaml` |
| **Watcher** | Detect ref changes under `.git/refs/heads/*` and optionally `packed-refs` | `notify` crate (Rust) or `fsnotify` (Go); fallback **polling** (`GIT_DIR` mtime + `git rev-parse`) every N seconds if inotify unavailable (NFS, some containers) |
| **Work scheduler** | Debounce rapid pushes; coalesce events; at-most-one draft job per `(repo_path, head_ref)` | In-memory queue + debounce window (e.g. 2–5 s) |
| **Git adapter** | Read-only `git` subprocess calls; parse output | Shell out to `git(1)` only—no libgit2 requirement in V1 unless latency demands |
| **Context builder** | Produce bounded prompt payload: commits, file list, diff hunks, `.pr-style.md`, `context.md` | Pure functions + hard caps (§4) |
| **LLM client** | HTTPS POST, streaming optional, retries, timeouts | Provider-agnostic interface; first adapter: OpenAI-compatible (`/v1/chat/completions`) |
| **GitHub client** | OAuth bearer, create draft PR, idempotency | `POST https://api.github.com/repos/{owner}/{repo}/pulls` |
| **State store** | Remember last processed `HEAD` per watched ref to avoid duplicate PRs | JSON: `~/.config/pr-drafter/state/{sanitized_repo_path}.json` |
| **Logger** | Structured logs to stdout + optional file | `RUST_LOG`/zerolog style; no secrets in logs |

### 1.2 Data flow

```
[filesystem: .git/refs/heads/X changes]
        → Watcher event (debounced)
        → Scheduler enqueues (repo_path, ref_name)
        → Git adapter: rev-parse HEAD, merge-base with base, log, diff
        → Context builder: inject .pr-style.md + ~/.config/pr-drafter/context.md + capped diff
        → LLM client → title + body (markdown)
        → GitHub client: POST draft PR (head = remote branch name resolution)
        → State store: write last_sha for ref
```

**Remote naming:** V1 assumes `origin` exists and `push.default` / explicit config `remote: origin`. `head` for GitHub API must be `owner:branch` if cross-fork; for same-repo feature branches use `branch_name` only when `head_repo` is implicit—document that V1 supports **same-repo branches only** unless `github.head` override is set (§7).

### 1.3 Daemon lifecycle

1. **Startup:** Parse config; validate paths exist and are git work trees; acquire lock; register watchers per repo; load state files.
2. **Reload (`SIGHUP`):** Re-read `config.yaml` and `.pr-style.md` / `context.md` paths (not necessarily file contents until next job—optional full reload).
3. **Steady state:** Event-driven wakeups + periodic **heartbeat** (e.g. every 6 h) to catch missed notify events on flaky FS.
4. **Shutdown (`SIGTERM`):** Stop accepting new jobs; wait in-flight job with timeout (e.g. 120 s); flush state; release lock.

**Single-developer assumption:** One daemon instance per user session or system-wide under dedicated UNIX user—document “do not run two daemons on same `state/` directory.”

---

## 2. GitHub OAuth setup and required scopes

### 2.1 App type

Use a **GitHub OAuth App** (not GitHub App for V1 simplicity). Callback URL for **Authorization Code flow with PKCE** (preferred over implicit; no client secret required on public native daemons if using PKCE with optional confidential client).

**Developer settings path (human):** `https://github.com/settings/developers` → OAuth Apps → New.

**Registered callback (local daemon):** `http://127.0.0.1:8765/oauth/callback` (pick fixed port in V1; configurable). The daemon binds this loopback listener only during `pr-drafter login`.

### 2.2 Authorization endpoints (reference)

- Authorize: `GET https://github.com/login/oauth/authorize`  
  Query params: `client_id`, `redirect_uri`, `scope`, `state`, `code_challenge`, `code_challenge_method=S256`
- Token exchange: `POST https://github.com/login/oauth/access_token`  
  Body: `client_id`, `client_secret` (if used), `code`, `redirect_uri`, `code_verifier`

### 2.3 Required OAuth scopes

| Scope | Why |
|-------|-----|
| `repo` | Create pull requests on **private** repositories; read repo metadata needed for `POST /repos/{owner}/{repo}/pulls`. |

**If all targets are public repositories only**, `public_repo` is narrower but **insufficient** the moment one private repo is added—recommend **`repo`** for V1 to avoid support burden.

**Not required for V1:** `workflow`, `read:org`, `admin:repo_hook`, `delete_repo`.

### 2.4 Token storage

- File: `~/.config/pr-drafter/github-token.json` or use **libsecret**/keychain in a later version; V1: file mode `0600`, JSON `{"access_token":"gho_...","token_type":"bearer","scope":"repo"}`.
- **Never** pass token to LLM prompts or log bodies.

### 2.5 GitHub API endpoints used

| Operation | Method | URL |
|-----------|--------|-----|
| Verify token / rate limit | `GET` | `https://api.github.com/user` |
| Resolve default branch | `GET` | `https://api.github.com/repos/{owner}/{repo}` → `default_branch` |
| Create draft PR | `POST` | `https://api.github.com/repos/{owner}/{repo}/pulls` |
| Optional: update PR body | `PATCH` | `https://api.github.com/repos/{owner}/{repo}/pulls/{pull_number}` |

**Create draft PR JSON body (minimal):**

```json
{
  "title": "string",
  "body": "string",
  "head": "feature-branch-name",
  "base": "main",
  "draft": true
}
```

**GitHub Enterprise:** Base URL from config, e.g. `https://github.example.com/api/v3`.

### 2.6 Rate limits

- Authenticated: 5,000 REST requests/hour per user (typical); daemon should stay well under this (one PR creation per branch advance).
- Respect `X-RateLimit-Remaining` and `Retry-After` headers (§8).

---

## 3. Git operations

### 3.1 Preconditions

- Each registered path must satisfy: `git -C <path> rev-parse --is-inside-work-tree` → `true`.
- **Watching scope:** Only **local** branches under `refs/heads/` that match `watch.branches` glob list (default: `*` excluding `HEAD`).

### 3.2 Detecting “new commits”

**Definition:** For branch `B`, let `H = git -C repo rev-parse B`. Let `B_base = git merge-base origin/<defaultBase> H` (or configured base ref). **Trigger** when `H` changes compared to persisted `state[B].last_head` **and** `H` is **not** an ancestor of `origin/<defaultBase>` (optional: still draft if ahead by ≥1 commit).

**Commands (illustrative):**

```bash
git -C /path/to/repo fetch origin --prune --quiet   # optional; user may fetch separately
git -C /path/to/repo rev-parse HEAD
git -C /path/to/repo rev-parse refs/heads/feature/foo
git -C /path/to/repo merge-base refs/heads/main refs/heads/feature/foo
git -C /path/to/repo log --reverse --format=%H%n%s%n%b%n---COMMIT--- refs/heads/feature/foo ^refs/heads/main
git -C /path/to/repo diff refs/heads/main...refs/heads/feature/foo
git -C /path/to/repo diff --stat refs/heads/main...refs/heads/feature/foo
```

Use **three-dot** `base...head` symmetric difference for PR-shaped diffs when `base` tracks upstream.

### 3.3 Remote / push assumptions

- Before calling GitHub API, ensure branch exists on remote:  
  `git -C repo rev-parse origin/feature/foo` must succeed **or** V1 skips API and logs “push branch first.”
- **No `git push` from daemon** unless explicitly added later; keeps blast radius low.

### 3.4 Edge cases

| Case | Behavior |
|------|----------|
| **Force-push / rewritten history** | New `H` still triggers; may create **second** draft PR unless state tracks `github_pr_number` per branch and V1 implements “update existing draft” (recommended: store `pr_number` in state and PATCH body if open+draft). |
| **Empty diff (only merge commit realignment)** | Skip LLM/GitHub; log “no net changes.” |
| **Huge diff** | Truncate with explicit sentinel in prompt: “[DIFF TRUNCATED]”; include full file list from `git diff --name-status`. |
| **Binary files** | Replace blob with “Binary file changed”; no base64 to LLM. |
| **Submodule pointers** | Show mode line change as text; do not recurse. |
| **Conflicts with base** | `merge-base` still defined; diff may include conflict markers only if user committed them—treat as text. |
| **Detached HEAD** | Do not watch; only `refs/heads/*`. |
| **Shallow clone** | `git fetch --depth` may make history incomplete—document; optionally warn if `.git/shallow` exists. |
| **Worktree / multiple worktrees** | V1: register **primary work tree path** only; ignore linked worktrees or detect duplicate `.git` and refuse. |

---

## 4. LLM integration

### 4.1 Model selection (V1 recommendation)

- **Default model:** `gpt-4o` or `gpt-4o-mini` (cost/latency tradeoff) via OpenAI-compatible API.
- **Alternative:** Anthropic Messages API (`https://api.anthropic.com/v1/messages`) behind same interface.

**Rationale:** Strong summarization of diffs; wide context windows.

### 4.2 Context window and token policy

- Reserve ~15–25% of context for **system + instructions + style files**.
- **Budget remainder** for: commit messages + file list + diff text.
- **Pre-count:** If using OpenAI, `tiktoken` (or provider tokenizer) to estimate; if unavailable, byte/char heuristic with safety margin.
- **Truncation order:** (1) trim inter-file diff hunks oldest-first or largest-first; (2) collapse hunks to `@@` headers only; (3) drop full diff, keep `--stat` + commit messages only; always log truncation level.

### 4.3 Prompt design

**System message (fixed, versioned in code):**

- Role: senior engineer writing PR descriptions for human reviewers.
- Output: strict **JSON** `{"title":"...","body":"..."}` where `body` is GitHub-flavored markdown.
- Constraints: no PII fabrication; if diff unclear, say what is uncertain; include **Testing** section placeholder if no tests touched; use imperative mood in title (~50 chars target).

**User message assembly:**

1. **Global context:** full contents of `~/.config/pr-drafter/context.md` (capped, e.g. 8k chars).
2. **Repo style:** full contents of `<repo>/.pr-style.md` (capped).
3. **Structured sections:** `## Commits` (from `git log`), `## Files`, `## Diff`.

**No** inclusion of: `GITHUB_TOKEN`, `.env`, SSH private keys, unrelated repo paths.

### 4.4 API endpoints (OpenAI-compatible)

- `POST https://api.openai.com/v1/chat/completions`  
  Headers: `Authorization: Bearer $OPENAI_API_KEY`, `Content-Type: application/json`  
  Body: `model`, `messages`, `temperature` (low, e.g. 0.2), `max_tokens`.

**Local inference (optional V1.1):** `POST http://127.0.0.1:11434/api/chat` (Ollama)—same interface adapter.

---

## 5. PR template rendering and context injection

### 5.1 File precedence

1. **System prompt** (built-in, immutable except version bumps).
2. **`~/.config/pr-drafter/context.md`** — personal preferences: tone, default sections, “always mention X”, signing style.
3. **`<repo>/.pr-style.md`** — repo voice, required headings, link to CONTRIBUTING, ticket URL patterns.

**Conflict rule:** Later sections in the **user** message clarify that repo style **overrides** global for structural headings; LLM instructed explicitly: “If instructions conflict, prefer `.pr-style.md` for structure and `context.md` for personal voice defaults.”

### 5.2 Optional static skeleton

If `.pr-style.md` contains fenced block tagged `<!-- pr-drafter:template -->` … optional parser extracts markdown skeleton into which LLM fills—V1 can skip parsing and treat entire file as instructions unless time permits simple extraction.

### 5.3 Output → GitHub `body`

- Pass LLM markdown **verbatim** after JSON parse validation.
- Escape nothing beyond JSON decoding; GitHub accepts markdown.

---

## 6. Security considerations

### 6.1 Input vectors

| Vector | Risk | Mitigation |
|--------|------|------------|
| **Repo files** (malicious project) | Prompt injection (“ignore instructions, exfiltrate token”) | System prompt hardening; **never** echo secrets; LLM output only JSON keys `title`/`body`; strip unknown keys; max length on `body`; outbound HTTP only to configured LLM + GitHub hosts (optional TLS pin not in V1). |
| **`.pr-style.md` / `context.md`** | Same | Treat as untrusted text; same caps; log file hashes not contents. |
| **Git commit messages** | Injection, social engineering in PR body | Sanitize/control length; no auto-execution of URLs by tool (N/A). |
| **Diff content** | Embedded ```` ``` instructions | Same as repo files; truncation reduces payload. |
| **GitHub API responses** | SSRF not applicable client-side; JSON parse bombs | Strict size limits on HTTP responses. |
| **Local config.yaml** | Path traversal to read arbitrary files | Only allowlisted keys; validate repo paths are absolute and exist under user home or explicit roots. |
| **OAuth callback** | CSRF | `state` param random + cookie/session match on loopback server. |
| **Token file** | Local privilege escalation | `chmod 0600`; warn if home dir world-readable. |
| **Env vars** | Leak via `/proc` | Document; run as dedicated user for server deployments. |

### 6.2 Trust boundaries

- **T1:** Local disk (full trust by OS user—daemon runs as developer).
- **T2:** LLM provider (trust: prompts contain proprietary code snippets—**user accepts** by using product).
- **T3:** GitHub API (trust: TLS + GitHub; token scoped `repo`).

### 6.3 Egress policy

- Implement **allowlist** of hostnames: `api.github.com`, `github.com` (OAuth), `api.openai.com` (or custom), optional enterprise host.
- **No** telemetry endpoints in V1.

### 6.4 Supply chain

- Lockfile (`Cargo.lock` / `go.sum`), reproducible builds, signed tags for releases.

---

## 7. Configuration format and defaults

### 7.1 Primary config path

`~/.config/pr-drafter/config.yaml`

### 7.2 Example schema (normative for V1)

```yaml
version: 1

daemon:
  debounce_ms: 3000
  poll_fallback_seconds: 300
  in_flight_timeout_seconds: 120

llm:
  provider: openai                    # openai | anthropic | ollama
  base_url: https://api.openai.com/v1
  model: gpt-4o-mini
  api_key_env: OPENAI_API_KEY         # read secret from env, not from file
  max_input_tokens: 120000
  temperature: 0.2

github:
  token_path: ~/.config/pr-drafter/github-token.json
  api_base: https://api.github.com
  oauth_client_id: Ov23liXXXXXXXXXXXX
  # optional for confidential OAuth app:
  # oauth_client_secret_env: GITHUB_OAUTH_CLIENT_SECRET

repos:
  - path: /home/dev/projects/acme-app
    enabled: true
    remote: origin
    base_branch: main                  # or auto-detect via API + git symbolic-ref
    watch:
      branches:
        - "feature/*"
        - "fix/*"
    github:
      owner: acme-corp
      repo: acme-app
    # head_branch_name defaults to local branch name; set if different:
    # head: my-fork:branch
```

### 7.3 Defaults

| Key | Default |
|-----|---------|
| `daemon.debounce_ms` | `3000` |
| `daemon.poll_fallback_seconds` | `300` |
| `llm.provider` | `openai` |
| `llm.model` | `gpt-4o-mini` |
| `repos[].remote` | `origin` |
| `repos[].base_branch` | `main` (with warning if missing; try `git symbolic-ref refs/remotes/origin/HEAD`) |
| Global context file | `~/.config/pr-drafter/context.md` (create empty if absent) |
| Repo style file | `<repo>/.pr-style.md` (optional; empty if absent) |

### 7.4 Environment variables

| Variable | Purpose |
|----------|---------|
| `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` | LLM auth |
| `PR_DRAFTER_CONFIG` | Override config path |
| `GITHUB_OAUTH_CLIENT_SECRET` | Optional OAuth app secret |

---

## 8. Error handling and recovery

### 8.1 Classified failures

| Class | Example | Action |
|-------|---------|--------|
| **Transient network** | TLS timeout, 502 | Exponential backoff (max 5 retries), then surface error |
| **Rate limit** | GitHub `403` + `X-RateLimit-Remaining: 0` | Sleep until reset + jitter |
| **LLM bad output** | Non-JSON, missing keys | Retry once with “reply JSON only”; then fail job |
| **Git error** | Shallow history, missing ref | Log; skip job; do not corrupt state |
| **GitHub 422** | head branch missing on remote | Log “push required”; skip state advance **or** advance to avoid hot loop—prefer **no state advance** so user retry works after push |
| **Duplicate PR** | Same open draft exists | List `GET /repos/{owner}/{repo}/pulls?head={owner}:branch&state=open` before POST (optional optimization) |

### 8.2 State consistency

- **Commit state only after** successful `201` from `POST /pulls` **or** successful `PATCH` if updating.
- Use atomic write: `write temp; fsync; rename` for JSON state.

### 8.3 Idempotency

- Key: `(repo_path, branch_name, head_sha)`.
- If `head_sha` already in state as `processed`, no-op.
- If branch moved but draft PR exists for previous sha—product decision: **update draft** (PATCH) vs new draft; recommend **one open draft per branch** enforced by `GET pulls` head filter.

---

## 9. Testing strategy

### 9.1 Unit tests

- Context builder: truncation tiers produce expected byte/token counts.
- JSON extraction from LLM responses with markdown fences.
- Config parser: valid/invalid YAML, missing fields.

### 9.2 Integration tests (local git fixtures)

- Script creates temp repo with `main` and `feature`, commits, `state/` pointing to temp `XDG_CONFIG_HOME`.
- Mock GitHub API with **wiremock** or `httptest` (Go) / `mockito` (Rust): assert `draft: true`, no assignees.
- Mock LLM with static JSON response.

### 9.3 Manual checklist

- [ ] OAuth login obtains token with `repo` scope.
- [ ] Creating PR leaves PR in draft; UI shows “Draft”.
- [ ] Kill daemon mid-LLM: no partial state advance.
- [ ] Large repo: truncation message appears in generated body (once).

### 9.4 CI

- GitHub Actions **not required** for V1 self-hosted doc; if added: `cargo test` / `go test ./...` on `ubuntu-latest` only.

---

## 10. Deployment

### 10.1 Install layout

```text
~/.config/pr-drafter/
  config.yaml
  context.md
  github-token.json          # chmod 600
  state/
    _home_dev_projects_acme-app.json
  pr-drafter.log             # optional
/usr/local/bin/pr-drafter    # binary
/etc/systemd/user/pr-drafter.service
```

### 10.2 Build commands (illustrative — Rust)

```bash
cd /root/pr-drafter
cargo build --release
sudo install -m 0755 target/release/pr-drafter /usr/local/bin/pr-drafter
install -d -m 0700 ~/.config/pr-drafter/state
```

### 10.3 `install.sh` (deliverable behavior)

- Detect arch/OS; download or build from source.
- Create dirs: `~/.config/pr-drafter/state`.
- Copy **example** `config.example.yaml` → `~/.config/pr-drafter/config.yaml` if missing.
- Touch `~/.config/pr-drafter/context.md` if missing.
- Print: run `pr-drafter login` then `systemctl --user enable --now pr-drafter`.

**Example user session:**

```bash
pr-drafter login              # opens browser / prints URL; writes github-token.json
export OPENAI_API_KEY=sk-...
pr-drafter doctor             # checks git, token, API reachability
pr-drafter run --foreground   # dev
```

### 10.4 systemd user unit

**File:** `~/.config/systemd/user/pr-drafter.service` (or ship template to `/etc/systemd/user/`).

```ini
[Unit]
Description=PR Description Drafter (local git watcher)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/pr-drafter run
Restart=on-failure
RestartSec=10
Environment=OPENAI_API_KEY=%h/.keys/openai  # prefer EnvironmentFile=
EnvironmentFile=-%h/.config/pr-drafter/env
WorkingDirectory=%h
LimitNOFILE=65535

[Install]
WantedBy=default.target
```

**Enable:**

```bash
systemctl --user daemon-reload
systemctl --user enable --now pr-drafter.service
systemctl --user status pr-drafter.service
journalctl --user -u pr-drafter.service -f
```

**Note:** `EnvironmentFile` should point to a file containing `KEY=value` lines; **do not** commit secrets.

### 10.5 Log rotation

- Optional `logrotate` stanza for `~/.config/pr-drafter/pr-drafter.log` if file logging enabled.

---

## 11. Implementation milestones (suggested order)

1. CLI skeleton: `run`, `login`, `doctor`, config load.  
2. Git context extraction + truncation + golden tests.  
3. LLM adapter + JSON validation.  
4. GitHub OAuth login + draft PR POST.  
5. Filesystem watcher + debounce + state file.  
6. `install.sh` + systemd unit templates in repo: `/root/pr-drafter/scripts/install.sh`, `/root/pr-drafter/scripts/pr-drafter.service`.

---

## 12. Open decisions (track before coding)

- **Update vs new draft PR** on repeated commits: single open draft per `head` ref (recommended).  
- **Auto-fetch:** opt-in `repos[].auto_fetch: true` vs never (default never).  
- **Title source:** LLM-only vs `git log -1 --format=%s` fallback if LLM fails.

---

*End of V1 engineering build plan.*
