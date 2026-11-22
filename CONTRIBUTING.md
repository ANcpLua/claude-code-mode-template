# Contributing

People and the `ccc` agent both change this repository. This file covers turning
on Code Mode, using the rules in every project, setting up `ccc`, and the rules for
changing the files. What Code Mode is and its trade-offs are in `README.md`.

## Set up Code Mode

Code Mode is experimental and off by default. Each person turns it on for
themselves: a project's settings can't turn it on.

Set the environment variable `CLAUDE_CODE_EXPERIMENTAL_CODE_MODE` to `1` before
Claude Code starts.

### Check the requirements

- Claude Code v2.1.323 or later; v2.1.330 or later is better (see `README.md`).
  Check with `claude --version`, update with `claude update`.
- The native install of Claude Code.
- A supported model: Opus 5.5, Sonnet 5.5, Haiku 5.5 or Fable 5.2.
- A Claude subscription or Console account, used directly.
- An interactive `claude` session in a terminal.

Read the **Cons** in `README.md` before you turn it on for every session. The
biggest one: Claude in Chrome, computer use and Remote Control stop working.

### Enable it

1. **For one session:** start Claude Code from a terminal like this:

   ```bash
   CLAUDE_CODE_EXPERIMENTAL_CODE_MODE=1 claude
   ```

2. **For every session:** add this to `~/.claude/settings.json`. Putting it in a
   project's settings has no effect.

   ```json
   { "env": { "CLAUDE_CODE_EXPERIMENTAL_CODE_MODE": "1" } }
   ```

   If a dotfiles manager owns `~/.claude/settings.json`, edit its source instead and
   apply it.

3. **Confirm:** on first launch it asks you to confirm. Choose **Yes, turn on Code
   Mode**. Claude Code then restarts and shows this line at the top:

   ```text
   Code Mode (experimental) is on · this session can be resumed only with Code Mode on
   ```

4. **See the code:** press `Ctrl+O` and expand a **Code** row to read the JavaScript
   behind the tool calls.

A session you already have open can't switch to Code Mode, and resuming it won't
turn it on. Start a new session.

### Turn it off

Remove the variable or set it to `0`. A `0` in either place wins, so with `1` in
your settings you can still start one session without Code Mode, for example for a
browser task:

```bash
CLAUDE_CODE_EXPERIMENTAL_CODE_MODE=0 claude
```

### If it doesn't start

The line `Code Mode (experimental) isn't available` at the top of the session gives
the reason, and `/status` shows it too. Look it up in the
[docs' troubleshooting table](https://code.claude.com/docs/en/code-mode#code-mode-doesnt-start).

## Use the rules in every project

Code Mode in `~/.claude/settings.json` is on in every project, but
`.claude/rules/code-mode.md` loads only in repositories made from this template.
To have the rules wherever Code Mode is on, put the file in `~/.claude/rules/`,
where Claude Code loads it in every project. The rules apply only while Code Mode
is on, so they stay quiet in other sessions.

- **Copy it** once: `cp .claude/rules/code-mode.md ~/.claude/rules/`. Updates from
  `ccc` don't reach the copy.
- **Or let your dotfiles manager fetch it**, so updates arrive. With chezmoi, add
  this to `.chezmoiexternal.toml` in the source directory and run `chezmoi apply`:

  ```toml
  [".claude/rules/code-mode.md"]
      type = "file"
      url = "https://raw.githubusercontent.com/ANcpLua/claude-code-mode-template/main/.claude/rules/code-mode.md"
      refreshPeriod = "24h"
  ```

Then repositories made from the template load the rules twice. Skip their copies
with `claudeMdExcludes` in `~/.claude/settings.json`, using a pattern that matches
where you keep repositories, for example:

```json
{ "claudeMdExcludes": ["**/Projects/**/.claude/rules/code-mode.md"] }
```

## Set up ccc

`ccc` (`.github/workflows/ccc.yaml`) runs once a day and on demand. Without its
token it does nothing and leaves a notice in the run.

1. In your own terminal, create a long-lived token for your Claude subscription:

   ```bash
   claude setup-token
   ```

2. Save it as the repository secret `CLAUDE_CODE_OAUTH_TOKEN`. This command asks
   for the value, so it stays out of your shell history:

   ```bash
   gh secret set CLAUDE_CODE_OAUTH_TOKEN
   ```

3. Let workflows open pull requests: in the repository's **Settings → Actions →
   General**, turn on **Allow GitHub Actions to create and approve pull requests**.

4. Run it once to check: `gh workflow run ccc`, then `gh run watch`.

The agent runs `claude -p` in CI, where Code Mode isn't available. That's expected:
it reviews text and edits files, and needs no batched tool calls.

## Keeping the template current

The ccc agent and people follow the same rules.

### Which file holds what

| File | Holds | Doesn't hold |
| --- | --- | --- |
| `README.md` | Facts about Code Mode, trade-offs, features that work well with it, version notes, the checked version and date | Rules for Claude, setup steps |
| `.claude/rules/code-mode.md` | Rules for how Claude works in Code Mode, in any project: what to do and when | Facts kept only for reference, setup steps, anything about one project or one machine |
| `CLAUDE.md` | This repository's own facts | Code Mode rules |
| `CONTRIBUTING.md` | Setup, `ccc`, and these rules | Trade-offs, rules for Claude |

- Put each fact in one file. Claude loads the rules file and `CLAUDE.md` in every
  session whether it needs them or not, so a fact repeated there costs context every
  time. Point to the other file instead.
- The rules file starts by saying it applies only when the `REPL` tool exists. Keep
  that line first: on machines that load it in every project, it keeps the rules
  quiet while Code Mode is off.
- Rules are concrete enough to check: a command, a path, a tool name, a limit. Keep
  the rules file under 200 lines; `ccc` refuses to merge over 500. Check with
  `wc -l .claude/rules/code-mode.md`.
- When a release makes something untrue, remove or correct it in the file that
  holds it. Don't add a note beside it.
- Update the checked version and date at the top of `README.md` whenever you check
  the template against a release or the docs.
- Write for both readers: plain sentences, no filler, no marketing.

### Before you merge a change by hand

Run `/code-review` on the change. From v2.1.328 it reports bugs and `CLAUDE.md`
violations, so it also catches a change that breaks the project instructions.
