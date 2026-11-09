# RESULTS: DUBJUG sandbox on the governed MCP gateway

Verification of the migration described in `docs/mcp-gateway-architecture.md`,
run 5 Nov 2026 on macOS with `sbx` v0.51.0, Docker Desktop, Codex workload
`sbx-kit-codex:0.162.1`. Evidence files live in the lab workspace
(`~/dev/dubjug/gateway-lab/out/`, indexed in `gateway-lab/findings.md`); the
file names below refer to that directory.

## Before / after

| | Before (`dubjug-atlassian-mcp-kit:1.3.1`) | After (gateway) |
| --- | --- | --- |
| `~/.codex/config.toml` | `[mcp_servers.mcp-gateway]` **and** `[mcp_servers.atlassian]` → `mcp.atlassian.com/v2/mcp` | `[mcp_servers.mcp-gateway]` only |
| Credential in sandbox | `ATLASSIAN_AUTH_HEADER=proxy-managed` placeholder; proxy injects the real header | None. No `ATLASSIAN_*` variable at all |
| Network policy | Allow rule for `mcp.atlassian.com` | No rule for either MCP host |
| `curl https://mcp.atlassian.com` from sandbox | 400 (reached the server) | 403 (blocked by the proxy) |
| Jira tools | `mcp__atlassian__*` (direct) | `mcp__mcp_gateway__*` (gateway) |
| Governance point | None: direct connection bypasses MCP policy | Every registration and tool call crosses the host gateway |

## Verification matrix (brief §8)

| # | Check | Result | Evidence |
| --- | --- | --- | --- |
| 1 | Before: direct path present | **Pass.** `[mcp_servers.atlassian]` present, `curl` 400 = reachable, `ATLASSIAN_AUTH_HEADER` set | `step4-checks.txt` (attempt 1, reused 1.3.1 sandbox) |
| 2 | Gateway variables present | **Pass.** `MCP_GATEWAY_URL`, `MCP_SENTINEL_TOKEN_NAME` set; no `ATLASSIAN_AUTH_HEADER` | `step4-checks.txt` 21:46Z |
| 3 | Workload hook wrote the gateway | **Pass.** `[mcp_servers.mcp-gateway]`, `url = http://mcp-gateway.docker.internal/mcp`, `Bearer proxy-managed`; `codex mcp list` shows only `mcp-gateway` | `step2b-sandbox.txt`, `step4-checks.txt` |
| 4 | After: no direct Atlassian config | **Pass.** `grep -iE 'atlassian\|notion' config.toml` → no match; no env var | `step4-checks.txt` |
| 5 | Servers registered and authorized | **Pass.** atlassian `ready` (custom headers, `${auth}: (set)`); notion `ready`, OAuth authorized, granted `default` | `step5-host-state.txt` |
| 6 | Servers attached | **Pass.** One gateway session lists 106 tools: 26 Atlassian (bare names) + 77 `notion-*` + `notion-revoke-auth` + `mcp-exec`, `code-mode` | `step5-gateway-tools.txt` |
| 7 | Read tools work | **Pass.** JQL `project = EM` → EM-1, EM-4, EM-6; `notion-get-self` → workspace "John Feeney's Space"; `notion-search` broad query → 7 results | `step5-codex-notion-jira.txt`, `step5-codex-notion-proof.txt` |
| 8 | Direct path blocked | **Pass.** `curl` to both hosts → 403 from the proxy | `step4-checks.txt` |
| 9 | Zero-touch | **Pass.** Fresh `sbx env rm` + `sbx env run` from the committed file: one plan approval, postCreate registered/loaded Atlassian and printed "Notion already authorized" (consent itself proven on the earlier create), all before Codex started; first session answered a Jira JQL (EM-7/6/4/1) and a Notion search ("Welcome to Notion!") through `mcp-gateway` | `step6-final-create.txt`; consent path: `step4-plan.txt`, `step4-run.txt` |
| 10 | Repeat run silent | **Pass with the setting, fail without.** As shipped, any plan declaring host commands prompts every run even when unchanged. With `sbx settings set env.rememberHostCommands true` **and** the plan matching the applied state, a second `sbx env run` is fully silent (verified). Caveat: a pending kit change ("applies on the next create") keeps the plan in a changed state, so it prompts or needs `--force` until the sandbox is recreated | `step5-repeat-run.txt`, `step6-final-create.txt` |
| 10b | Machine-level file | **Pass (plan-level).** `mcp:` block moved to `~/.sbxenv.yaml`, project block removed: the merged plan lists both files and still registers notion; project `lifecycle` commands merge in | `step5-machine-level-plan.txt` |
| 10c | Path B declarative | **Pass.** postCreate registers and loads Atlassian; developer action limited to one `sbx secret set` (which works *before* registration, no restart) | `step2a-register.txt`, `step4-run.txt` |
| 11 | Mode | **Static.** No `mcp-find` / `mcp-add` / `mcp-config-set` in the session | `step5-gateway-tools.txt` |
| 12 | Context delivered | **Pass.** Codex answers site, gateway route for Jira and Notion, and the ticket rule from the kits; both marker blocks exactly once in `AGENTS.md`; site arg expanded | `step4-checks.txt`, pasted TUI |
| 13 | Ticket rule honoured | **Pass.** Asked for a small source edit, Codex ran `searchJiraIssuesUsingJql` on EM, found no match, `createJiraIssue` → **EM-7**, cited the key, then edited. (Edit reverted afterwards; EM-7 remains in Jira as evidence) | `step5-matrix13-ticket-rule.txt` |
| 14 | Live attach | **Resolved with a caveat.** `sbx mcp load` before the agent starts is the working pattern. The gateway advertises `tools/listChanged`, but Codex 0.162 does not refresh a running session: tools granted mid-session (e.g. Notion authorized after start) appear only in a new session | findings 5.1 |
| 15–18 | Policy rows | **Skipped.** `sbx policy ls` → "Managed by docker": this account is governed by the Docker org, where demo policy must not be written. Needs a separate demo org where John is admin. Groundwork recorded: every Atlassian and Notion tool carries `readOnlyHint`/`destructiveHint`, so the read-only Notion posture in the draft Cedar policy is workable; `identityURL` must equal the exact registered URL | `step1-host.txt`, `step2c-gateway-tools.txt` |

## Answers to the brief's open questions (§9)

1. **Atlassian `/v2/mcp` OAuth?** Not tested: path B (header auth) was chosen
   before OAuth was attempted, because the scoped-token tool set was already
   proven under 1.3.1. `/v2/mcp` works with header auth; `/v1/mcp/authv2` never needed.
2. **OAuth vs API-token tool set?** For the API token, the gateway relays
   exactly the direct list: 26 tools including `searchJiraIssuesUsingJql` and
   `discover`/`execute*`. (OAuth set not measured, see 1.)
3. **Tool name shape?** Bare upstream names (`mcp.useServerNamePrefixes`
   defaults to `false`). Codex qualifies as `mcp__mcp_gateway__<tool>`
   (hyphens → underscores) and logs `mcp-gateway/<tool>`. No collisions:
   Notion's names all start `notion-`.
4. **Does `sbx env run` run OAuth consent during apply?** No. `mcp.servers`
   registers unauthorized; the gateway then hides all the server's tools and
   offers only `<server>-authorize`. Consent is run by the postCreate
   `sbx mcp auth notion` command instead.
5. **Static or dynamic?** Static, both with `--static-mcp` and with
   `mcp.servers` + `sbx mcp load`.
6. **`mcp.servers` vs header registration of the same name?** Untested;
   avoided by keeping `atlassian` out of `mcp.servers`.
7. **Does mixin `agent-context@1` merge with the workload's AGENTS.md?** The
   AGENTS.md append hooks and `agent-context@1` coexist: marker blocks appear
   exactly once, the workload's context is intact, and Codex also opens the
   mixin side file in the TUI.
8. **Local or hosted gateway?** Local (`"decision": "local"`).
9. **Secret before registration?** Yes. `sbx secret set mcp:atlassian:auth`
   works with no registration present; the later `sbx mcp add … ${auth}` binds
   immediately, no restart. Documented order: set secret, then first run.
10. **`~/.sbxenv.yaml` merge?** Yes: the plan lists both files and registers
    the machine-level `mcp.servers` entry (plan-level test; not applied).
11. **`sbx mcp ls --json` shape?** Pretty-printed array of objects with
    top-level `"name"`; guard is `grep -Eq '"name": *"atlassian"'`.
12. **Two OAuth consents in one plan?** Not applicable in the final design:
    only Notion uses OAuth, and consent runs as one postCreate command.
13. **Do Notion tools declare `readOnly`?** Yes. All 77 carry `readOnlyHint`
    and `destructiveHint`, so the read-only Notion policy works as drafted.

## What contradicted the original brief or target design

- **Repeat-run approval cannot be designed away in the file.** Moving every
  host command to `postCreate` (nothing in `initialize`) still prompts on each
  `sbx env run`, because approval is tied to host commands being *declared*,
  not to whether any would run. The brief's path-A expectation "a plan with
  only `url` servers applies silently" does not survive adding any `lifecycle`
  block. Remedy: `env.rememberHostCommands true` (per machine, MDM-able), or
  `--force` / `--skip-host-commands` per run.
- **`mcp.servers` does not run OAuth consent** (brief Q4 hoped it might), and
  agent-initiated `notion-authorize` is a dead end in the Codex TUI: the
  wrapped sign-in link corrupts on copy, and a token obtained mid-session is
  invisible to that session anyway (Codex ignores `tools/listChanged`).
  Sign-in therefore happens on the host at create (postCreate
  `sbx mcp auth notion`), and the Notion kit rule (1.0.1) forbids
  `notion-authorize` and points the user at the host command plus a new
  session.
- **`sbx env run` reuses an existing sandbox** without applying kit changes or
  postCreate ("waits for the next create"). Anyone holding a pre-gateway
  sandbox must `sbx env rm` once. Documented in the README.
- **Atlassian ended on path B** (scoped API token in a header) rather than the
  brief's path-A-first preference, by decision: known-good tool set and the
  token story the demo wants.

## sbx / Codex observations worth filing

1. **Codex 0.162 ignores `tools/listChanged`** from the gateway: a running
   session never picks up tools granted after it started (new server loaded,
   or OAuth completed). Clean workaround: new session. Repro: start Codex with
   Notion unauthorized, run `sbx mcp auth notion` on the host, ask the same
   session for Notion tools (absent); a new `codex exec` sees all 77.
2. **Approval on every `sbx env run` even when the plan is a no-op** (see
   above). Arguably working as designed, but the message "nothing in this plan
   has changed" invites a gentler default for postCreate-only plans.
3. **Minor:** the Codex TUI line-wraps long URLs with indented continuation
   lines, so copied OAuth links carry spaces; `pbpaste | tr -d ' \n' | pbcopy`
   repairs them.
