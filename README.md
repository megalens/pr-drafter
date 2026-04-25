# PR Description Drafter

A self-hosted git daemon that watches local branches, generates PR descriptions from diffs via LLM, and creates draft pull requests on GitHub.

**Status:** Build plan complete, security-audited. Implementation pending.

## What's inside

`build-plan.md` is a 493-line V1 engineering build plan covering architecture, GitHub OAuth (PKCE), git operations, LLM integration (OpenAI-compatible), template system, security considerations, configuration, error handling, testing strategy, and systemd deployment.

**This plan was security-audited two ways** to compare single-model vs multi-engine AI code review.

## Case Study: Single Model vs Multi-Engine Review

We reviewed `build-plan.md` two ways:

### Single model (Claude Sonnet 4.6)

One prompt, one model. Found **23 vulnerabilities:**

| Severity | Count |
|----------|-------|
| Critical | 3 |
| High | 7 |
| Medium | 8 |
| Low | 5 |

Caught the core issues: command injection via git subprocess args, prompt injection from repo files, token exfiltration through LLM payloads, path traversal in state store, TOCTOU races, OAuth CSRF, SSRF via configurable endpoints.

### MegaLens MCP (3 engines + GPT 5.4 judge)

Same plan, one MCP tool call. Found **45 vulnerabilities:**

| Severity | Count |
|----------|-------|
| Critical | 7 |
| High | 10 |
| Medium | 26 |
| Low | 2 |

MegaLens used 3 independent engines (Grok 4.1 Fast, DeepSeek V3.2, Gemini 3.1 Pro) debating in 2 rounds, with GPT 5.4 as final judge.

### The delta: 22 additional findings

Issues the single model missed but multi-engine debate caught:

- **Git config helper execution** — repository `.gitconfig` can define custom diff/merge tools, credential helpers, and aliases that execute arbitrary code when the daemon runs `git -C <repo> diff`. All 3 engines missed this; the GPT 5.4 judge caught it independently (Critical)
- **OAuth callback port hijacking** — fixed localhost port `8765` lets any local process race to bind first and intercept the authorization code. Judge-only finding (High)
- **Git transport protocol abuse** — `git fetch` can use SSH/Git protocols that trigger remote code execution via ProxyCommand or external transport helpers. Judge-only finding (High)
- **Stored XSS via PR markdown** — LLM output passes verbatim to GitHub PR body; crafted markdown with `<img>` tags or JavaScript URIs becomes a stored XSS vector for anyone viewing the PR. Judge-only finding (Medium)
- **YAML deserialization RCE** — config.yaml parsed without safe-load specification; unsafe YAML constructors (`!!python/object`) enable code execution (Critical)
- **Git subprocess resource exhaustion** — no timeout or output size limit on `git diff`/`git log` subprocesses; a repo with 10GB history causes OOM or indefinite hang (Critical)
- **Force-push state corruption** — SHA-based state tracking breaks on history rewrites; daemon creates duplicate PRs or enters hot loop (Critical)
- **SIGHUP config injection** — reload accepts new repo paths without re-validation; attacker who writes to config.yaml gets arbitrary path watching (High)
- **Symlink attacks on state files** — state directory paths derived from repo paths without symlink resolution; symlink to `/etc/passwd` causes write to system files (High)
- **TOCTOU in GitHub PR creation** — concurrent jobs can both pass the "check existing draft" query and both create duplicate PRs (Medium)
- **Branch-to-remote identity mismatch** — underspecified fork/remote mapping can create PRs targeting wrong repository (Medium)
- **Localhost health endpoint abuse** — browser-driven CSRF to local health/setup endpoint not bounded by origin checks (Medium)
- **LLM API cost runaway** — no per-hour or per-day cost cap; rapid branch updates trigger unbounded API spend (Medium)
- **No audit logging** for security-relevant events like token usage, OAuth flows, failed auth attempts (Medium)
- **Token reuse across all repos** — single `repo`-scoped token grants access to all private repos; no per-repo scoping (Medium)

### Why multi-engine matters

The 7 judge-originated findings are the most striking: issues that **all three debater engines independently missed**, but the GPT 5.4 judge caught by reasoning about the intersection of their analyses. The git config helper execution finding (Critical) is a real-world attack vector that single-model reviews consistently miss because it requires understanding Git's internal extension points, not just the daemon's code.

When engines disagree, that's signal. Grok found YAML deserialization RCE that DeepSeek and Gemini missed. DeepSeek found git subprocess resource exhaustion that the others missed. The debate surfaces these blind spots.

### Performance

| Metric | Value |
|--------|-------|
| Engines | Grok 4.1 Fast + DeepSeek V3.2 + Gemini 3.1 Pro |
| Judge | GPT 5.4 |
| Rounds | 2 (structured debate) |
| Total time | 419s (7.0 min) |
| Total cost | $0.22 (BYOK via OpenRouter) |
| Tier | Standard (3 engines + judge) |

## Review outputs

- [`build-plan.md`](build-plan.md) — Full V1 engineering build plan
- [`results/step2-cursor-solo-review.txt`](results/step2-cursor-solo-review.txt) — Single-model review (23 findings)
- [`results/step3-megalens-audit.txt`](results/step3-megalens-audit.txt) — MegaLens multi-engine audit (45 findings)

## The tool itself

PR Description Drafter is a real tool designed for real use:

1. **Watches** local git branches via inotify/polling
2. **Extracts** commit messages + diffs + repo context
3. **Generates** PR title and body via LLM (OpenAI-compatible)
4. **Creates** draft PRs on GitHub via OAuth (PKCE, `repo` scope)
5. **Runs** as a systemd user service

See `build-plan.md` for the full architecture, security model, and implementation milestones.

## Try it yourself

1. Clone this repo
2. Install MegaLens MCP in your IDE ([setup guide](https://megalens.ai/extensions/mcp))
3. Run `megalens_debate` with `skill: "security_audit"` on `build-plan.md`
4. Compare your IDE's solo findings vs MegaLens multi-engine results

## License

MIT
