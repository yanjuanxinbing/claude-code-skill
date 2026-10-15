# Delegation recipes (Windows PowerShell)

Copy-paste starting points. Each recipe keeps the two rules that matter: stderr stays out
of the JSON stream, and output decoding is UTF-8.

Common prelude — set encoding, then define the target and the prompt file:

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8

$Dir  = 'C:\path\to\repo'
$Task = Join-Path $env:TEMP 'cc-task.md'
Set-Content -Path $Task -Encoding UTF8 -Value @'
Goal: <what Claude Code must produce>
Working directory: <the repo> — inspect it before answering.
Done when: <verifiable condition, e.g. "the failing test passes">
Report back: a summary under 15 lines, plus file:line references for each finding.
Do not commit or push anything.
'@
```

Use a here-string (`@'` … `'@`) for the prompt: literal text, no PowerShell interpolation,
so `$` and backticks inside the prompt survive intact.

## 1. Read-only investigation (default)

```powershell
$out = Join-Path $env:TEMP 'cc-out.json'
$err = Join-Path $env:TEMP 'cc-err.txt'
Push-Location $Dir
(Get-Content $Task -Raw) | & claude -p --output-format json `
  --permission-prompts none `
  --allowedTools "Read,Grep,Glob,WebFetch,WebSearch,Bash(git status*),Bash(git diff*),Bash(git log*)" `
  1> $out 2> $err
Pop-Location

$j = Get-Content $out -Raw -Encoding UTF8 | ConvertFrom-Json
"is_error=$($j.is_error) cost=$($j.total_cost_usd) turns=$($j.num_turns) session=$($j.session_id)"
"denials: $(@($j.permission_denials).Count)"
$j.result
```

Empty `.result` with `is_error=$true` → read `$err`.

## 2. May-edit implementation

```powershell
$out = Join-Path $env:TEMP 'cc-out.json'
$err = Join-Path $env:TEMP 'cc-err.txt'
Push-Location $Dir
(Get-Content $Task -Raw) | & claude -p --output-format json `
  --permission-mode acceptEdits --permission-prompts none `
  --allowedTools "Read,Write,Edit,Glob,Grep,Bash" `
  --max-budget-usd 5 `
  1> $out 2> $err
Pop-Location
```

Then check what actually changed on disk yourself:

```powershell
Push-Location $Dir; git status --short; git diff --stat; Pop-Location
```

Run the project's own test/build command before telling the user it works.

## 3. Follow-up on the same session

```powershell
Push-Location $Dir
'Now also cover the error path in the same file.' | & claude -p -r '<session_id from step 1>' `
  --output-format json --permission-mode acceptEdits --permission-prompts none `
  --allowedTools "Read,Write,Edit,Glob,Grep,Bash" 1> $out 2> $err
Pop-Location
```

Resuming keeps Claude Code's context, so it does not re-read the whole repo. Use
`--fork-session` instead of `-r` when the follow-up should branch off cleanly.

## 4. Long task — background the command, keep `-p`

Same command as mode 1 or 2, run through the `Bash` tool with
`run_in_background: true`, then poll with `task_output`. Do not use `claude --bg`:
it rejects `-p`, needs a human-accepted workspace trust prompt, and `claude agents`
needs a TTY.

## 5. Machine-readable answer

```powershell
$schema = '{"type":"object","properties":{"findings":{"type":"array","items":{"type":"object","properties":{"file":{"type":"string"},"line":{"type":"integer"},"issue":{"type":"string"}},"required":["file","line","issue"]}},"summary":{"type":"string"}},"required":["findings","summary"]}'
Push-Location $Dir
(Get-Content $Task -Raw) | & claude -p --output-format json --json-schema $schema `
  --permission-prompts none --allowedTools "Read,Grep,Glob" 1> $out 2> $err
Pop-Location

$j = Get-Content $out -Raw -Encoding UTF8 | ConvertFrom-Json
($j.result | ConvertFrom-Json).findings | Format-Table file, line, issue
```

## 6. Extra directories

```powershell
--add-dir 'C:\path\to\sibling' --add-dir 'C:\path\to\fixtures'
```

Repeatable; grants both read and write scope there, so name only the directories the task
genuinely needs.

## 7. Verify a read-only delegation without touching the target

Snapshot before the run, so "nothing changed" is evidence rather than an assumption:

```powershell
$Dir = 'C:\path\to\target'
$before = Get-ChildItem $Dir -Recurse -File |
          ForEach-Object { [pscustomobject]@{ Path = $_.FullName; Hash = (Get-FileHash $_.FullName -Algorithm SHA256).Hash } }
# ... run the delegation (recipe 1) ...
$after = Get-ChildItem $Dir -Recurse -File |
         ForEach-Object { [pscustomobject]@{ Path = $_.FullName; Hash = (Get-FileHash $_.FullName -Algorithm SHA256).Hash } }
Compare-Object $before $after -Property Path, Hash
```

Empty output means untouched. Remember that `permission_denials` being empty only says
Claude Code never *tried* to write — this comparison is the actual proof.

To reproduce behaviour claims, run them on a copy outside the target:

```powershell
$Verify = Join-Path $env:TEMP ("cc-verify-" + (Get-Date -Format 'HHmmss'))
New-Item -ItemType Directory -Force -Path $Verify | Out-Null
Copy-Item "$Dir\*" $Verify -Recurse
Push-Location $Verify
# run the scenarios Claude Code described, one per branch
Pop-Location
```

Report what you measured, including where it contradicts `.result`.
