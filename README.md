# plan-review-harness

Independent review for AI-written plans and code. Each role runs as a **fresh, separate model session**, so the reviewer never sees the author's context.

```
planner -> critic -> [you approve] -> coder -> tests -> reviewer -> auditor -> [you merge]
```

- `bin/harness-run <role> <rundir> [input files...]` runs one role. It builds the prompt (role prompt + your principles + the input files), calls the first working backend in the role's chain, writes `<rundir>/<role>.md` and `<role>.meta` (which backend served, model family, cost, seconds), appends `log.jsonl`, and prints the last `VERDICT:` line.
- Backends: `claude -p` (Claude Code CLI), `opencode run`, or a direct OpenAI-compatible HTTP call. Each role has a **fallback chain**; on error or outage it moves to the next entry.
- **Independence rule**: `role.avoid=<other role>` skips any backend whose model family equals the family the other role was served by. The family is a plain string guess from the model name (`claude:opus` and `claude:claude-opus-5-5` count as different families); pin it explicitly with `@family`, e.g. `claude:opus@claude-opus`. If nothing is left it exits 1 (no silent fallback).
- **Plan gate** (`bin/plan-gate-hook`): a Claude Code `PreToolUse` hook on `ExitPlanMode`. It lints the plan for required sections, sends it to the critic, and bounces it back with findings on REVISE/REJECT (max 2 rounds, then you decide).
- **Secret scan**: before a prompt goes to a third-party (non-Claude) backend, it is checked for token patterns. A hit skips that backend.
- `bin/harness-report roles` summarises runs, cost and latency from `log.jsonl`.

More detail: [docs/internals.md](docs/internals.md).

## Requirements

- bash 4.4+, **GNU grep** (`grep -P`), `jq`, `curl`
- macOS ships bash 3.2 and BSD grep, so install the GNU tools and put them first on `PATH`: `brew install bash grep jq coreutils`, then e.g. `export PATH="$(brew --prefix)/opt/grep/libexec/gnubin:$(brew --prefix)/opt/coreutils/libexec/gnubin:$(brew --prefix)/bin:$PATH"`. Without GNU grep the secret scan cannot run, and `harness-run` exits 4 instead of running a scanned backend unchecked.
- [Claude Code CLI](https://docs.claude.com/en/docs/claude-code) (`claude`), logged in, for `claude:` backends
- optional: `opencode` (for `opencode:` backends), `python3` (for `harness-report`), `shellcheck` (to lint)

## Install

```bash
git clone https://github.com/alamin-mahamud/ai-harness && cd ai-harness
H=~/.claude/harness                      # any dir works; set HARNESS_HOME if you pick another
mkdir -p "$H" && cp -r bin prompts "$H"/ # or: ln -s "$PWD/bin" "$PWD/prompts" "$H"/
cp roles.conf.example          "$H/roles.conf"
cp model-purposes.conf.example "$H/model-purposes.conf"
# optional backends (placeholders inside, edit them, chmod 600):
cp cp.env.example "$H/cp.env"; cp gw.env.example "$H/gw.env"
# optional house principles (every *.md in the dir is injected into every role prompt):
mkdir -p ~/.claude/principles && cp examples/principles/*.md ~/.claude/principles/
```

Note: the sample principles declare `plan_sections`, which **turns on the plan-gate section lint**: every plan-mode plan is bounced until it has the headings Facts, Baseline, Confidence, Steps, Rollback and Acceptance. Edit that list in the sample (or skip copying it) if you do not want that.

The roles read and search the **current directory**: their tools resolve relative paths from it (and `--setting-sources project` loads that project's settings), so run `harness-run` from the repo under review.

Try it:

```bash
RUN="$H/runs/$(date +%Y%m%d-%H%M%S)-demo"; mkdir -p "$RUN"
echo "Goal: add a retry to the upload script. Constraint: no new deps." > "$RUN/task.md"
echo "Plan: wrap curl in a 3-try loop with backoff. Test: kill the server mid-upload." > "$RUN/planner.md"
"$H/bin/harness-run" critic "$RUN" "$RUN/task.md" "$RUN/planner.md"   # prints: VERDICT: APPROVE | REVISE | REJECT
cat "$RUN/critic.md" "$RUN/critic.meta"
```

Each run needs real inputs in files; the critic only sees what you pass plus what its tools can read.

## roles.conf

One `role=chain` line per role. Chain entries are separated by ` | ` and tried left to right.

```
planner=claude:claude-opus-5-5 | claude:opus | claude:sonnet
# tools allowed for claude/cp/gw backends:
planner.tools=Read,Grep,Glob,WebSearch,WebFetch
# run that nested claude with all hooks disabled:
planner.hooks=off
reviewer=claude:claude-opus-5-5 | claude:opus
# independence rule:
reviewer.avoid=coder
```

Comments go on their own lines. A trailing `  # comment` after a value is stripped, but keep them separate anyway.

| entry | meaning |
|---|---|
| `claude:<model>` | `claude -p` with your normal login (`sonnet`, `opus`, `haiku` or a full model id) |
| `cp:<model>` | `claude -p` through a cliproxy-style proxy you trust; needs `cp.env` (`HARNESS_CP_URL`, `HARNESS_CP_KEY_FILE`, optional `HARNESS_CP_CA`). `HARNESS_CP_PREFIX` (default empty) is prepended to the model id, e.g. `claude/` for a proxy that serves Claude as `claude/<model>`; the short names `sonnet`, `opus`, `haiku` map to `claude-sonnet-5-5`, `claude-opus-5-5`, `claude-haiku-5-5` |
| `gw:<alias>` | `claude -p` through an Anthropic-compatible gateway; needs `HARNESS_GW_URL`, `HARNESS_GW_KEY` |
| `oai:<alias>` | direct streamed OpenAI-compatible chat call, no tools; same `HARNESS_GW_*` vars |
| `opencode:<provider/model>` | `opencode run -m ...` |
| `...@family` | override the guessed model family (used by `role.avoid`), e.g. `oai:reviewer@glm` |

Keys per role: `role.tools`, `role.avoid`, `role.hooks=off`, `role.mcp=<name>` (reads `$HARNESS_HOME/mcp/<name>.json`; see `mcp/rag.json.example`). A missing variable or file makes that backend fail and the chain moves on. `HARNESS_OVERRIDE='claude:opus' harness-run ...` replaces the chain for one run.

Useful env vars: `HARNESS_HOME`, `HARNESS_CP_PREFIX`, `HARNESS_GATE_TIMEOUT`, `HARNESS_PRINCIPLES_DIR` (set it empty to run with no principles; harness-run warns), `HARNESS_PRINCIPLES_REQUIRED=on` (exit 3 instead of warning), `HARNESS_SECRET_SCAN=off`, `HARNESS_WORKFLOW=<label>`, `HARNESS_CLAUDE_FLAGS`, `HARNESS_PLAN_GATE=off`, `HARNESS_BASELINE_GATE=off`.

## Plan-gate hook for Claude Code plan mode

Add to `~/.claude/settings.json` (merge with your existing `hooks`):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          { "type": "command", "command": "$HOME/.claude/harness/bin/plan-gate-hook", "timeout": 600 }
        ]
      }
    ]
  }
}
```

Behaviour: if your principles define `plan_sections:` in their frontmatter (see `examples/principles/`), a plan without those headings is rejected before any critic call (a plan may say `Baseline: n/a - <reason>`). Otherwise the plan goes to the `critic` role. APPROVE passes; REVISE/REJECT exits 2 and Claude gets the findings to fold in. After 2 gated rounds the next `ExitPlanMode` passes and you decide. The hook wraps the critic call in `timeout` (`HARNESS_GATE_TIMEOUT`, default 540 s, needs coreutils `timeout`), kept below the 600 s hook timeout: if the critic is too slow or every backend fails, the plan **passes unreviewed** with a "critic unavailable" notice. If Claude Code kills the hook at its own timeout instead, treat the plan as unreviewed too. The critic can read files and search the web but cannot run commands, so put live evidence (command output, metrics) into the plan text.

## Security notes

- **Read-only roles run with hooks off** (`planner`, `critic`, `reviewer`, `auditor`, `chores`: `role.hooks=off`). Without it the nested `claude -p` fires your own Stop/SessionEnd hooks from the current project, which can commit or push. The `coder` role keeps hooks and has Edit/Write/Bash: run it only in a branch or worktree you can throw away.
- **Keys come from env or env files only.** `*.env` is git-ignored; `cp.env`/`gw.env` are sourced by `harness-run` (keep them `chmod 600`); the cliproxy key lives in a separate file named by `HARNESS_CP_KEY_FILE`. Nothing in this repo contains a credential: the `.example` files are placeholders.
- **Prompts bound for a third party get a secret scan.** The decision depends on the backend kind, never on the model name: `claude:` (your own login) and `cp:` (a proxy you declare trusted) are not scanned; `gw:`, `oai:` and `opencode:` always are. Checked: token shapes (AWS, GitHub, GitLab, Slack, npm, Google, Cloudflare, Hugging Face, LLM API keys, JWT, Bearer, private keys) plus your configured gateway/proxy key values. A hit skips that backend. It needs GNU grep; without it `harness-run` exits 4. `harness-run --scan <file...>` runs the scanner on its own.
- **`gw:` and `opencode:` backends have tools.** The scan covers the prompt only; files such as `.env` or keys that the model `Read`s are sent to that third party as well. Do not run those backends on a tree that holds secrets.
- Keys never appear in process arguments (curl gets its header on stdin; the nested `claude` gets its env inside a subshell). `gw:`/`cp:` runs drop inherited `ANTHROPIC_*` credentials so they cannot reach the gateway host, and every nested session loses `HARNESS_GW_KEY` / `HARNESS_CP_KEY_FILE`. A gateway or proxy URL must be `https://` (plain `http://` only for `localhost` / `127.0.0.1`).
- Critic output is advice, not proof. Keep tests, linters and a human approval step in the loop.

## Maintainers

Published as a snapshot of a private working copy; issues and PRs welcome, changes are ported upstream.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
