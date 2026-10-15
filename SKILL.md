---
name: claude-code
description: |
  Drive the installed Claude Code CLI (`claude`) as a second agent and integrate its
  result: Mavis keeps the conversation and hands one bounded subtask to an agent with
  its own Read/Edit/Bash tool loop, project CLAUDE.md conventions, skills and
  multi-turn session memory. Use when the user says "用 Claude Code / claude 跑一下",
  "让 claude 改一下 / 看看这个 PR / review 一下", asks for a second-opinion review, or
  asks to operate Claude Code directly. Do NOT use for a plain one-shot LLM call (use
  `llm-call`), for work Mavis can do directly with its own tools, or when the `claude`
  CLI is not installed.
descriptions:
  zh-Hans: "调用本机 Claude Code CLI（claude）作为第二个智能体执行一个边界清晰的任务，并把结果整合回来。用户说'用 Claude Code / claude 跑一下''让 claude 改一下''让 claude review''用 claude 看看这个 PR'或要求直接操作 Claude Code 时使用。普通单次大模型调用用 llm-call；Mavis 自己能直接做完的活不要委派；未安装 claude CLI 时不要用。"
displayNames:
  zh-Hans: "Claude Code 委派"
---

# Claude Code Delegation

Hand one bounded subtask to the `claude` CLI and own the integration. Mavis keeps the
conversation with the user; Claude Code runs a self-contained task with its own tool
loop and returns a final message that lands in `.result`.

The CLI on this machine is usually pointed at a third-party backend
(`ANTHROPIC_BASE_URL` in `~/.claude/settings.json`), so model names look foreign and
cost per run is nonzero. That is expected; see Failure handling.

## Inputs to collect

- **Target dir** — Claude Code only sees its cwd plus `--add-dir` entries. Each `Bash`
  call starts in the session workspace, so `cd` must happen inside the same command.
- **Mode** — read-only or may-edit. Infer from the request; ask only when the user
  could plausibly mean either (e.g. "让 claude 处理一下这个 bug").
- **Done condition** — the artifact or answer you will verify afterwards (a diff, a
  findings list, a passing test run). Without it Claude Code optimizes for its own
  summary instead of yours.

## Preflight

Run once per session before the first delegation:

```powershell
(Get-Command claude -ErrorAction SilentlyContinue).Source; claude --version
```

If `claude` is missing, stop and report it, offering
`npm i -g @anthropic-ai/claude-code` — do not install it on your own initiative, and do
not fall back to `llm-call` without saying so.

Never read, print, or echo `~/.claude/settings.json`: it holds a live
`ANTHROPIC_AUTH_TOKEN`. If auth fails, send the user to `claude auth status` or
`claude doctor` and let them handle credentials.

## Choose the mode

| Need | Flags |
| --- | --- |
| Investigate / review / analyze (default) | `--allowedTools "Read,Grep,Glob,WebFetch,WebSearch,Bash(git status*),Bash(git diff*),Bash(git log*)" --permission-prompts none` |
| Implement / fix / run tests | `--permission-mode acceptEdits --permission-prompts none --allowedTools "Read,Write,Edit,Glob,Grep,Bash"` |
| Pure reasoning, no file access | `--tools ""` |

Do not pass `--dangerously-skip-permissions` or `--allow-dangerously-skip-permissions`.
Those are the user's own guardrails; delegation must not switch them off. If the task
genuinely needs full bypass, explain why and let the user run that command.

`--permission-prompts none` is the load-bearing flag: in `-p` mode nobody can answer a
permission prompt, so anything outside the allowlist is denied instead of hanging. That
turns the allowlist into a real ceiling — verified: a read-only run asked to write a
file reports the block in `.permission_denials` and leaves the disk untouched.

Narrow the allowlist to what the task actually needs before running: the default row
grants `WebFetch`, `WebSearch` and three git `Bash` patterns, which are dead weight for a
single-file review and widen the ceiling for nothing. Add back only what the task needs.

Read-only mode cannot execute code — only the git patterns above. Claude Code will then
say it could not confirm behaviour empirically and fall back to static analysis. That is
a mode limitation, not a failure: if the task needs Claude Code to actually *run*
something, use the may-edit row or allow the specific command, and say which you chose.

## Procedure

1. **Write the task prompt to a file** (e.g. `$env:TEMP\cc-task-<n>.md`) and pipe it in:
   `(Get-Content $f -Raw) | claude -p ...`. A file keeps long or multi-line text exact —
   inline strings get mangled by PowerShell quoting and encoding, and the user can read
   what was delegated. The prompt should state the goal, the working directory, the done
   condition, and "reply with a summary under N lines" (that text is what you get back).

2. **Run it in the target dir in the same call**: `Set-Location <dir>` (or `Push-Location`
   … `Pop-Location`) before invoking `claude`. Add `--add-dir <path>` when the task needs
   files outside cwd.

3. **Capture structured output**: `--output-format json` with stdout to a file and stderr
   to a separate file. Do not merge them — a backend model-name warning on stderr breaks
   `ConvertFrom-Json`.

4. **Parse the fields you need**, not the whole blob:
   `.result` (the answer), `.is_error`, `.session_id`, `.num_turns`, `.total_cost_usd`,
   `.permission_denials`.

5. **Verify the claim before reporting it** — see Verifying what comes back.

6. **Follow-ups**: reuse `.session_id` with `-r <session_id>` to keep its context instead
   of re-sending files; use `--fork-session` when branching; `--no-session-persistence`
   for throwaway one-shots.

Add `--max-budget-usd <n>` when the user cares about spend on a large task, and
`--effort low|medium|high` to trade depth for speed.

For long or structured output, put it in the prompt instead: ask Claude Code to write its
findings to a file under `$env:TEMP` and return only a short pointer, then read that file.
That keeps `.result` small enough to stay untruncated.

### Verifying what comes back

Claude Code is a second model, not a source of truth. It gets file references and
severity rankings right far more often than it gets side-effect claims right, so verify
the **observable** claims — what it says a command does, what a file ends up containing,
whether a run succeeds — rather than counting findings.

When the target must not be modified, verify on a copy instead of in place:

1. Copy the relevant files to a scratch dir outside the target (`$env:TEMP\cc-verify-*`).
2. Run the commands there, per scenario Claude Code described.
3. Report what you measured, and correct it plainly where it disagrees.

One disagreement per run is normal; an unverified disagreement-free report is the
suspicious one.

`permission_denials` empty means Claude Code never *tried* to exceed the allowlist — it
is not proof that write protection worked. To show the user nothing changed, record
timestamps or `Get-FileHash` before and after the run.

### Long-running tasks

Run the same command with the `Bash` tool's `run_in_background: true`, then poll with
`task_output`. Do **not** use `claude --bg` for delegation: it starts an interactive
session, rejects `-p`, requires a human-accepted workspace trust prompt, and
`claude agents` needs a TTY — none of which exist in this environment.

## Windows (win32) platform notes

Full copy-paste commands: `references/recipes.md`. Verified flag surface and traps:
`references/cli-flags.md`.

- **Encoding**: set both before capturing output, otherwise CJK and punctuation come back
  as mojibake through the pipeline:
  `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8; $OutputEncoding = [System.Text.Encoding]::UTF8`
  The mojibake can appear only in the console echo while the saved file is valid — read the
  saved file with `Get-Content -Encoding UTF8` or the Read tool instead of trusting what
  scrolls past.
- **Executable shim**: `claude` resolves to `claude.ps1` (npm shim). If the execution
  policy blocks it, call `claude.cmd` instead — same CLI.
- **stderr is not empty**: a `[claude-code:unrecognized_model]` line is normal with a
  third-party backend. Keep it out of the JSON stream.
- **PowerShell 7** is assumed (`ConvertFrom-Json`, `1>` redirection, UTF-8 defaults). On
  Windows PowerShell 5.1 add `-Encoding UTF8` to every read/write and expect stricter
  encoding defaults.
- **Temp artifacts**: the prompt file, stdout JSON and stderr log belong in `$env:TEMP`
  (`cc-task*.md`, `cc-out*.json`, `cc-err*.txt`), never in the user's repo. Name them so
  a follow-up run can find them; they are disposable, so say where they are rather than
  leaving them to be discovered.

## Output contract

Report to the user, in their language:

1. What was delegated, at which path, and in which mode (read-only vs may-edit).
2. The substance of `.result`. It is normally Markdown with fenced code blocks — keep the
   content, drop the wrapper, and reorganise it to the user's question rather than pasting
   the run verbatim.
3. Files changed, confirmed by your own check — `git status` / `git diff` inside a repo,
   or `Get-FileHash` / timestamps when the target is not a repo — not by Claude Code's word.
4. `session_id` (so a follow-up is cheap) and `total_cost_usd`.
5. What you verified yourself, what you could not, and any `.permission_denials`.

State plainly when a task was delegated and came back partially done; do not present an
unverified run as a finished change.

## Failure handling

- `is_error = true` — read `.subtype` and `.api_error_status`. Retry once; if it repeats,
  report the status and stop rather than looping.
- `.permission_denials` non-empty — the task needed a tool outside the allowlist. Report
  which tool and on what path, then widen deliberately; for a read-only task, widening to
  write access is a scope change the user should hear about.
- Missing or wrong `result` — read the stderr file; an empty `.result` with
  `stop_reason` other than `end_turn` is an upstream failure, not an empty answer.
- Repeated timeouts — split the task into smaller delegated units, or do it yourself.
- Auth or trust errors — hand them to the user with `claude auth status` / `claude doctor`;
  credentials and trust dialogs are theirs, not yours to bypass.

## Examples

**Input**: "用 claude 帮我 review 一下这个 PR 的改动"

Read-only mode in the repo, prompt asks for findings as `file:line — problem`, allowlist
without write tools. Parse `.result`, then open the three highest-severity files yourself
to confirm the claims are real before listing them for the user.

**Input**: "让 claude 把登录那个测试修好"

May-edit mode with `acceptEdits` in the repo. Claude Code edits and may run the test, but
its "tests pass" statement is not evidence — run the suite yourself, and report the real
result plus the diff.
