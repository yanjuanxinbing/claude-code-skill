# Claude Code CLI — verified flag surface (v2.1.283)

Verified on Windows / PowerShell 7 against `claude --version` = `2.1.283`, with the CLI
pointed at a third-party Anthropic-compatible backend. Read this when a flag in `SKILL.md`
is not enough, or when a command behaves unexpectedly.

## Headless basics

| Flag | Effect | Verified |
| --- | --- | --- |
| `-p` / `--print` | Non-interactive: run one task, print, exit. Skips the workspace trust dialog. | yes |
| `--output-format json` | Single JSON object on stdout. Contains `result`, `session_id`, `is_error`, `stop_reason`, `num_turns`, `total_cost_usd`, `permission_denials`, `subtype`, `api_error_status`. | yes |
| `--output-format stream-json` | Realtime event stream. Only needed for live progress; far noisier — prefer `json` plus harness backgrounding. | not needed |
| prompt via stdin | `(Get-Content $f -Raw) \| claude -p ...` reads the prompt from stdin. | yes |
| prompt as positional | `claude -p '<text>'` — fine for short single-line prompts. | yes |

The JSON object is large (`modelUsage`, token accounting, frame timings). Parse and
print only the fields you need; pasting the raw object wastes context.

## Permissions

| Flag | Effect | Verified |
| --- | --- | --- |
| `--permission-prompts none` | Nobody answers prompts in `-p` mode: anything requiring approval is denied. This is what makes the allowlist a ceiling. | yes |
| `--allowedTools "Read,Grep"` | Pre-approve these tools. Bash entries take patterns: `Bash(git diff*)`. | yes |
| `--disallowedTools` | Explicit deny list, evaluated after allow. | not exercised |
| `--permission-mode acceptEdits` | Auto-accept file edits; still requires the prompt channel for anything else, so pair with `--permission-prompts none`. | yes |
| `--tools "Bash,Edit,Read"` | Restrict the toolset itself. `--tools ""` = reasoning with no file access. | yes |
| `--add-dir <dir>` | Grant access to directories outside cwd. Repeatable. | yes |
| `--dangerously-skip-permissions` | Bypass all checks. Do not use for delegation. | avoided by design |

Verified denial behaviour: with `--allowedTools "Read,Grep,Glob" --permission-prompts none`,
a request to create a file produced `permission_denials` containing the `Write` call, no
file on disk, and a Claude Code message explaining the block.

## Sessions

| Flag | Effect | Verified |
| --- | --- | --- |
| `--session-id <uuid>` | Start with a chosen id (must be a UUID). Useful to pre-name a multi-step delegation. | not exercised |
| `-r` / `--resume [id]` | Resume a session by id, keeping its context — the cheap way to send a follow-up. | yes |
| `-c` / `--continue` | Resume the most recent conversation in the current directory. | not exercised |
| `--fork-session` | With resume: new session id, copied history. Use when the follow-up should not pollute the original. | not exercised |
| `--no-session-persistence` | `-p` only: do not write the session to disk. Use for throwaway runs. | not exercised |

Resume is worth the habit: a follow-up costs one turn instead of re-reading the same
files, and the answer stays consistent with the earlier run.

## Prompt shaping and limits

- `--system-prompt <text>` / `--append-system-prompt <text>` — override or extend the
  system prompt for this run.
- `--json-schema '<schema>'` — constrain the final result to a JSON Schema, so the
  `.result` is machine-checkable. Verified working; use it when you must post-process.
- `--max-budget-usd <n>` — `-p` only: hard spend ceiling for the run.
- `--effort low|medium|high|xhigh|max` — trade depth for latency/cost.
- `--model <alias>` — aliases (`opus`, `sonnet`, …) resolve through the configured backend;
  with a third-party `ANTHROPIC_BASE_URL` they map to whatever that backend serves.
- `--agents '<json>'` — define sub-agents for fan-out inside one run.
- `--add-dir` also carries CLAUDE.md discovery for those directories.

## Background and background-session traps

- `claude --bg '<task>'` starts an interactive session for `claude attach` / `logs` /
  `stop` / `rm`. Two hard limits in a non-TTY harness: it **rejects `-p`** ("`--bg` and
  `--print` conflict"), and it **requires a human-accepted workspace trust prompt**
  ("Workspace not trusted"), which print mode does not need.
- `claude agents` errors with "requires an interactive terminal" unless you pass
  `--json`.
- Conclusion: for long delegation runs, background the *command* (harness background
  execution) and keep `-p`. Do not reach for `--bg`.

## Diagnostic subcommands

`claude doctor` (health check, no trust prompt), `claude auth` (auth state),
`claude project purge [path]` (drop a project's transcripts/config), `claude mcp`,
`claude plugin`. These are for the user to run when configuration is broken — pass them
through rather than working around them.

## Windows traps

| Symptom | Cause | Fix |
| --- | --- | --- |
| CJK / em-dash becomes `?` | PowerShell native-output decoding | `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8; $OutputEncoding = [System.Text.Encoding]::UTF8` |
| `ConvertFrom-Json` fails on the output | stderr merged into stdout | redirect stderr separately (`2> err.txt`) |
| `claude` cannot be loaded / policy violation | npm `claude.ps1` shim blocked | call `claude.cmd` |
| Runs in the wrong directory | each tool call starts in the session workspace | `Set-Location <dir>` inside the same command |

`[claude-code:unrecognized_model] {"model":"..."}` on stderr is benign with a third-party
backend — the CLI warns that it does not recognise the model string and proceeds anyway.
It is not an error, and it does not belong in the user's error report.
