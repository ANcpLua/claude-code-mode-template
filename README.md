# Claude Code Mode template

A starting point for repositories you work in with Claude Code's **Code Mode**.
People and Claude both read this file. It holds the facts and the reasons.
`.claude/rules/code-mode.md` holds Claude's working rules, and `CONTRIBUTING.md`
holds setup and upkeep. Each fact lives in one file.

Checked against Claude Code **v2.1.330** and the
[Code Mode docs](https://code.claude.com/docs/en/code-mode) on **2026-11-21**. The
`ccc` workflow keeps this current.

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
- **Background runs.** In auto and `bypassPermissions` mode, code can run for up to
  two hours while you keep talking to Claude.
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
  can't start a background copy.
- **Resume lock-in.** A Code Mode session resumes only with Code Mode on. If you turn
  it off later, those sessions can't be reopened.
- **Narrow eligibility.** Claude Code v2.1.323 or later, native install, with Opus
  5.5, Sonnet 5.5, Haiku 5.5 or Fable 5.2, on a Claude subscription or Console
  account used directly. Not through Bedrock, Google Cloud, Foundry or a gateway;
  not under HIPAA or zero data retention.
- **Settings that block it.** An `apiKeyHelper`, a custom `ANTHROPIC_BASE_URL`,
  `DISABLE_TELEMETRY`, and some plugins and mods. OpenTelemetry events and metrics
  work from v2.1.329; earlier versions refuse any OpenTelemetry export.
- **Less visible by default.** You see the tool calls, not the code, unless you open
  it. Values found along the way are used inside the code without Claude reading
  each one first.
- **Weaker auto-mode review.** The auto-mode classifier reviews each program, not
  each call, and doesn't see values computed while it runs, such as a path taken
  from search output. Keep rules or hooks for calls you must control.
- **No pause between steps in `bypassPermissions`.** A whole program of edits and
  commands runs start to finish without a prompt.
- **Sandbox covers shell only.** The JavaScript and its file, web and MCP calls run
  outside the sandbox. Full isolation needs a container or VM.
- **Ten-minute limit.** Code you wait on stops after ten minutes, unless it can run
  in the background.
- **Hook edge cases.** A `PreToolUse` hook that times out refuses the call with
  `the hook did not finish`. From v2.1.328, `UserPromptSubmit` hooks no longer
  receive code output, so checks there move to a `PostToolUse` hook.

## Works well with Code Mode

| Feature | Since | Why it helps |
| --- | --- | --- |
| `/tasks` | v2.1.329 | Lists background code, subagents, scheduled tasks, `/loop` wakeups and forks in one place; `x` stops one after asking. |
| `/status` | v2.1.327 | Lists hook matchers that never match, so a misspelled `REPL` matcher shows up. |
| `"onFailure": "block"` on hooks | v2.1.328 (`mcp_tool`), v2.1.329 (`command`, `http`) | A guarding hook that fails stops the call instead of letting it run. |
| easel, the built-in chart mod | v2.1.330 | Code can gather the numbers and easel draws the chart; it works in Code Mode from this version. |
| OpenTelemetry events and metrics | v2.1.329 | Monitoring no longer has to be off. |

## Use v2.1.330 or later

v2.1.330 fixes Code Mode problems you would otherwise hit:

- A long pause after code that read hundreds of files, in projects with path-scoped
  rules or Read deny rules
- Code Mode ignoring an MCP tool's request to end the turn in auto and bypass modes
- A file a script had read not being restored after `Esc` and `/compact`
- `/context` undercounting system tools before Claude's first reply

## When to use it

| Good fit | Poor fit |
| --- | --- |
| The same change across many files | Work in a browser or on the desktop |
| Searches whose results need filtering | Sessions in VS Code, the Desktop app or the cloud |
| Several independent commands | Teams on Bedrock, Google Cloud, Foundry or a gateway |
| Find, edit and test in one chain | Organizations with HIPAA or zero data retention |

## Guardrails worth adding

In `~/.claude/settings.json`:

- Approve each program before it runs: `"permissions": { "ask": ["REPL"] }`
- Approve a specific command even in auto or bypass mode, for example
  `"permissions": { "ask": ["Bash(git push *)"] }`
- Keep commands in the sandbox when code asks to leave it: turn on strict sandbox
  mode.

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
