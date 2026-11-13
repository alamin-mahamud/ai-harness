# Harness internals
Flow: planner -> critic -> [you approve] -> coder -> tests (hard gate) -> reviewer -> auditor -> [you merge]
- roles.conf      role -> fallback chain of backends (the ONLY place models are named); edit to experiment
- prompts/<role>.md  role prompts (model-agnostic)
- bin/harness-run <role> <rundir> [inputs]
- runs/<ts>/      handoff files: task, planner, critic, coder, reviewer (+ .meta/.err)
- log.jsonl       served model, family, cost, secs per stage
Backends: claude:<m> | cp:<m> (cliproxy-style proxy, cp.env) | gw:<alias> (HARNESS_GW_URL/HARNESS_GW_KEY env) | oai:<alias> (OpenAI-compatible chat call) | opencode:<prov/model>
Independence: role.avoid=<role> skips any backend whose family equals the avoided role's served family (the family is a string guess from the model name; pin it with `@family`, e.g. `claude:opus@claude-opus`);
if nothing is left it exits 1 (no silent fallback). Family = claude-<model> for claude, else leading letters of model name, or @family override.
Escalate one run: HARNESS_OVERRIDE='claude:opus' harness-run planner ...
Nested-run flags: HARNESS_CLAUDE_FLAGS (default skips user hooks/skills/MCP for speed).

## MCP, workflow label, harness-report
- `role.mcp=<name>` in roles.conf: claude:/gw: backends get `~/.claude/harness/mcp/<name>.json` (strict-mcp-config stays on, so only that server) and `mcp__<name>` is auto-allowed. The example config ships with the mcp lines commented out. oai: roles have no tools.
- `HARNESS_WORKFLOW=<label>` (default `dev`) is recorded as `workflow` on every log.jsonl line.
- `harness-report roles [days]`: per-role runs/cost/latency from log.jsonl. `models` and `spend` are optional and need a LiteLLM proxy on Kubernetes (see the header of bin/harness-report; model-purposes.conf documents why each alias exists).

## Principles
Rules every role follows live as `*.md` files in one directory (`HARNESS_PRINCIPLES_DIR`, default `~/.claude/principles`; see `examples/principles/`). Do not copy them into prompts; link.
- `bin/harness-principles text|sections`: the one reader. `harness-run` prepends `text` to every role prompt; `plan-gate-hook` lints with `sections`.
- New principle = add a file to the directory.
- Opt-out: HARNESS_PLAN_GATE=off, HARNESS_BASELINE_GATE=off.

## Auditor role and secret scan
- Role `auditor` (roles.conf, prompts/auditor.md): final read-only missed-work + security pass for larger changes. Independent of context, not of family (Claude on both sides).
- The planner and critic prompts ask for an industry-reference comparison (>=2 approaches); the coder prompt forbids sub-agents and polling and caps tool output.
- Secret scan: before a prompt goes to a `gw:`, `oai:` or `opencode:` backend it is checked against prefix-anchored token patterns (AWS, GitHub, GitLab, Slack, npm, Google, Cloudflare, Hugging Face, LLM API keys, JWT, Bearer, private keys) and against the configured gateway/proxy key values. The decision depends on the backend kind, never on the model name: `claude:` and `cp:` are not scanned, the others always are. A hit skips that backend and the chain falls through; only the pattern NAME is logged (`skipped:["secret:<name>"]`). Needs GNU grep -P (exit 4 without it). Off: `HARNESS_SECRET_SCAN=off`. Replay: `harness-run --scan <file...>`. The scan covers the prompt only: `gw:` and `opencode:` backends have tools, so files they Read leave the machine too.
- `harness-run` warns on stderr and sets `principles:false` in .meta/log.jsonl when the principles text is empty.
- Patterns are adapted from the MIT-licensed gitleaks rules and vendor token-format docs; a docker-run gitleaks was judged too slow per harness call, so the scan is a small prefix-anchored PCRE list with no dependency besides GNU grep.
- `HARNESS_PRINCIPLES_REQUIRED=on` makes harness-run exit 3 when the principles text is empty (default: warn only, so a broken principles directory does not take the critic/plan-gate down).

## Role isolation and plan gate
- `role.hooks=off` (read-only roles): nested `claude -p` gets `--settings '{"disableAllHooks":true}'`. Without it a critic run from a project checkout fires that project's Stop/SessionEnd hooks, which can commit or push. The coder keeps hooks (lint, guard).
- plan-gate-hook: at most 2 gated critic rounds per plan (the counter is per session and resets whenever the gate passes). The status line names the served model and flags a fallback when it differs from the head of `critic=`. The critic call is wrapped in `timeout` (HARNESS_GATE_TIMEOUT, default 540 s). Run dir: runs/plan-<sid>/.
