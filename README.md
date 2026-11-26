# Claude Code Mode template

A starting point for repositories you work in with Claude Code's **Code Mode**.
People and Claude both read this file. It holds the facts and the reasons.
`.claude/rules/code-mode.md` holds Claude's working rules, and `CONTRIBUTING.md`
holds setup and upkeep. Each fact lives in one file.

Checked against Claude Code **v2.1.332** and the
[Code Mode docs](https://code.claude.com/docs/en/code-mode) on **2026-11-26**. The
`ccc` workflow keeps this current. The trade-offs below describe that version. An
older one has more limits: the docs' requirements list them.

## What Code Mode is

Without Code Mode, Claude asks for one tool call at a time, such as Bash, Read or
Edit, and reads each result before it decides what to do next. With Code Mode,
Claude writes a short JavaScript program that makes the calls, and Claude Code runs
it in the session. One program can run commands in parallel, feed one result into
the next call, and return only the part of the output Claude needs to read.

You still see the tool calls, such as each edit's diff, but not the program. Press
`Ctrl+O` and expand a **Code** row to read it.

Code Mode is **experimental**, off by default, and may change or be removed in any
release. A repository can't turn it on: each person opts in for themselves, as
`CONTRIBUTING.md` describes. Where Code Mode isn't on, including in CI, Claude works
the usual way.

## What's in this repository

| Path | Read by | What it does |
| --- | --- | --- |
| `README.md` | People and Claude | What Code Mode is, its trade-offs, and what works well with it. |
| `.claude/rules/code-mode.md` | Claude, every session | How to work in Code Mode: when to batch calls in code, how to write and report that code, time limits and safety. Applies only while Code Mode is on, so you can also use it in every project on your machine (see `CONTRIBUTING.md`). |
| `CLAUDE.md` | Claude, every session | Your project's own facts: build and test commands, conventions, pitfalls. |
| `CONTRIBUTING.md` | People and the ccc agent | Turning Code Mode on and off, using the rules everywhere, setting up `ccc`, and which file holds what. |
| `.github/workflows/ccc.yaml` | GitHub Actions | **ccc**, Claude Code changelog aware: reviews each new Claude Code release for Code Mode. |
| `.github/workflows/claude.yml` | GitHub Actions | Answers a comment, review or issue that mentions `@claude`. |
| `.github/ccc/` | ccc | The Claude Code version and the docs page hash the template was last checked against. |
| `.gitignore` | Git | Keeps personal `CLAUDE.local.md` files out of the repository. |

## Pros

- **Fewer round trips.** One program can find a file, edit it and run the tests.
  Without Code Mode each of those is a separate step.
- **Less to read.** Code filters and trims output before Claude sees it, so a long
  search or log costs a few lines of context, not thousands.
- **Parallel work.** Independent commands run at the same time.
- **Fits repetitive work.** The same change across many files is one program, not
  dozens of edits.
- **Background runs.** In a session started in auto or `bypassPermissions` mode,
  code can run for up to two hours while you keep talking to Claude.
- **The same per-call guardrails.** Every tool call inside the code still goes
  through your `deny` and `ask` rules and `PreToolUse` hooks under its usual name,
  and shell commands still run in your sandbox.
- **You can inspect it.** `Ctrl+O` shows the code. A `PreToolUse` hook that matches
  `REPL`, the tool that runs the code, can check code before it runs, and a
  `PostToolUse` hook can check or redact what it returns.
- **Opt-in by design.** A repository you clone can't switch it on for you.

## Cons

- **Experimental.** It can change or disappear in any release.
- **Lost tools.** Claude in Chrome, computer use and Remote Control don't work in
  Code Mode sessions.
- **Terminal only.** Not in the VS Code panel, the Desktop app or cloud sessions. A
  session can't move to the Desktop app or become a background session, and `/fork`
  can't start a background copy. Agent-team teammates run in Code Mode in the
  default in-process display mode and start without it in split panes.
- **Resume lock-in.** A Code Mode session resumes only with Code Mode on. If you turn
  it off later, those sessions can't be reopened. A session that has run on a
  Claude subscription or a Console account can't be resumed on a cloud provider.
  A resumed session keeps no variables from its earlier code.
- **Narrow eligibility.** The native install, with Opus 5.5, Sonnet 5.5, Haiku 5.5
  or Fable 5.2, on a Claude subscription, a Console account or a cloud provider:
  Amazon Bedrock, Google Cloud's Agent Platform, Microsoft Foundry or Claude
  Platform on AWS. With an Enterprise Admin Console sign-in, only Agent Platform or
  Bedrock without the Mantle endpoint. Not through a gateway or with Claude for
  Government; not under HIPAA or zero data retention.
- **Settings that block it.** An `apiKeyHelper` or `ANTHROPIC_AUTH_TOKEN`, a custom
  `ANTHROPIC_BASE_URL`, OpenTelemetry tracing, and launch options that take the
  `REPL` tool away: `--restricted`, a `--tools` list without it, or a `deny` rule
  on it. OpenTelemetry events and metrics don't block it, and neither does
  `DISABLE_TELEMETRY` or another setting that turns feature-flag fetching off.
- **Less visible by default.** You see the tool calls, not the code, unless you open
  it. Values found along the way are used inside the code without Claude reading
  each one first.
- **Weaker auto-mode review.** The auto-mode classifier reviews each program, not
  each call, and doesn't see values computed while it runs, such as a path taken
  from search output. Keep rules or hooks for calls you must control.
- **Countdown prompts in auto mode.** Once you approve code yourself, after
  repeated classifier blocks or through an `ask` rule or a `PreToolUse` hook on
  `REPL`, its tool calls can prompt with a two-minute countdown. Unanswered, the
  call is refused, and so are later ones until you send Claude a message.
- **No pause between steps in `bypassPermissions`.** A whole program of edits and
  commands runs start to finish. Only your `ask` rules and the actions no mode
  auto-approves still prompt.
- **Sandbox covers shell only.** The JavaScript and its file, web and MCP calls run
  outside the sandbox. Full isolation needs a container or VM.
- **Ten-minute limit.** Code you wait on stops after ten minutes. Code runs in the
  background only in a session started in auto or `bypassPermissions` mode and
  still in it. A session begun in a mode that asks, such as Manual or
  `acceptEdits`, keeps waiting when you resume it in another mode: start a new
  session in auto mode instead.
- **Hook edge cases.** A `PreToolUse` hook that times out refuses the call with
  `the hook did not finish`. A command can reach the hook with a `cd` in front,
  such as `cd 'app' && git push`, so a hook that allows or blocks shell commands
  has to check each part of a command. `UserPromptSubmit` hooks don't receive code
  output: check it in a `PostToolUse` hook on `REPL`. A hook that redacts there must
  also leave `uncutOutputNote`, the path to the uncut text, out of the copy it
  returns.

## Works well with Code Mode

| Feature | Since | Why it helps |
| --- | --- | --- |
| `/tasks` | v2.1.329 | Lists background code, subagents, scheduled tasks, `/loop` wakeups and forks in one place; `x` stops one after asking. |
| `/status` | v2.1.327 | Lists hook matchers that never match, so a misspelled `REPL` matcher shows up. |
| `"onFailure": "block"` on hooks | v2.1.328 (`mcp_tool`), v2.1.329 (`command`, `http`) | A guarding hook that fails stops the call instead of letting it run. |
| `/permissions` | v2.1.332 | Refuses a rule that would be skipped at the next start and shows the reason, so a mistyped `ask` or `deny` rule doesn't fail without a word. |
| easel, the built-in chart mod | v2.1.330 | Code can gather the numbers and easel draws the chart; it works in Code Mode from this version. |
| OpenTelemetry events and metrics | v2.1.329 | Monitoring no longer has to be off. Each tool call the code makes sends its `tool_decision` and `tool_result` events under the tool's usual name; their `tool_use_id` starts with `repl_` or `toolu_cc_`. |

## Use v2.1.332 or later

v2.1.332 fixes Code Mode problems you would otherwise hit:

- A tool call whose input has a list of over 4,096 items being skipped without a
  word, and a freeze on a very large typed array in a tool's input, which is now
  refused
- Subagents failing with "Agent stalled" and losing their command when code ran for
  more than ten minutes
- Claude not being told that a slow command moved to the background and left the
  code with empty output
- Waits of minutes on a command that had already been stopped, and on resuming a
  paused workflow while other background work ran
- A message typed during a command going unanswered when you pressed `Esc` before
  Claude replied; it now starts a new turn, which a second `Esc` stops
- A typed slash command's inline shell commands not running after background code's
  output arrived
- Code Mode's tool search cutting long MCP tool descriptions short, which could lead
  to wrong calls

It also tells Claude at a resume that its earlier variables are gone, and Claude now
confirms before it discards worktree changes.

Earlier releases fixed more. The
[changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) lists
them under each version.

## When to use it

| Good fit | Poor fit |
| --- | --- |
| The same change across many files | Work in a browser or on the desktop |
| Searches whose results need filtering | Sessions in VS Code, the Desktop app or the cloud |
| Several independent commands | Teams behind a gateway or on Claude for Government |
| Find, edit and test in one chain | Organizations with HIPAA or zero data retention |

## Guardrails worth adding

In `~/.claude/settings.json`:

- Approve each program before it runs: `"permissions": { "ask": ["REPL"] }`. In
  auto mode this brings the countdown prompts described under **Cons**.
- Approve a specific command even in auto or bypass mode, for example
  `"permissions": { "ask": ["Bash(git push *)"] }`
- Keep commands in the sandbox when code asks to leave it: turn on strict sandbox
  mode.
- If you export OpenTelemetry events, leave `OTEL_LOG_TOOL_DETAILS` unset unless
  your backend may receive values that code reads and passes on, such as a token
  from a file.

An organization can block Code Mode for everyone by setting
`CLAUDE_CODE_EXPERIMENTAL_CODE_MODE` to `0` in managed settings.

## How this stays current

Once a day, `ccc` checks the
[Claude Code changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
for releases newer than the version above. For each one it opens a pull request,
and a Claude agent reviews the release. If nothing affects Code Mode, the agent
closes the pull request and says why. If a release changes Code Mode or adds
something that works well with it, the agent updates the rules and docs and the
pull request is merged. Setup is in `CONTRIBUTING.md`.
