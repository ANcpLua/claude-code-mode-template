# How Claude works in this repository

<!--
Maintainers: Claude loads this file in every session. It holds rules only; facts,
trade-offs and the checked Claude Code version live in README.md, setup in
CONTRIBUTING.md. Keep it under 200 lines (ccc refuses to merge over 500).
HTML comments like this one are stripped before Claude sees the file.
-->

You work in this repository in Code Mode. Instead of asking for one tool call,
reading the result and deciding the next, you write a short JavaScript program
that makes the calls, and Claude Code runs it in the session. This is the normal
way of working here. `README.md` has the facts and trade-offs when you need the why.

## Check the mode first

- Code Mode is on when you have a tool named `REPL`. That tool runs your code.
- If there is no `REPL` tool, Code Mode is off, as in CI or a session started
  without it. Work with direct tool calls and follow the rest of this file where it
  applies. In an interactive session, say once that Code Mode is off.
- Code Mode can't be turned on from inside a session or from project settings.
  Don't edit `.claude/settings.json` to enable it.

## When to write code, when to make one call

Write code when a task needs several related tool calls:

- The same change across many files
- A search whose results need filtering before you read them
- Independent commands that can run at the same time
- A chain where one result feeds the next call: find the file, edit it, test it

Make a single, separate call when:

- One read or one command answers the question
- The action is destructive or reaches outside the machine: deleting files,
  `git push`, publishing, deploying, sending messages. Give it its own step so the
  user sees it on its own, and never bury it in a batch
- The next step depends on judgment about the result, not just its value

## Writing the code

Use these building blocks, and only helpers the `REPL` tool describes. Don't invent
others.

```javascript
const files = await $`rg -l "function retryPolicy" src`.text();
const target = files.trim().split("\n")[0];
await edit(target, "maxAttempts: 3", "maxAttempts: 5");
({ edited: target, tests: $`npm test -- retryPolicy`.nothrow() });
```

- `` $`...` `` runs a shell command through the Bash tool. `.text()` gives its
  output as a string. `.nothrow()` keeps a failing command from throwing, so you
  can return its output.
- `edit(path, oldText, newText)` edits a file. The user sees the diff.
- The value of the last expression comes back to you. Return an object with named
  fields.

Rules for every program:

- **Return little.** Filter, count and trim in code. Return paths, counts, failing
  lines and short excerpts, not whole files or full logs.
- **Check before you act on a computed value.** If a search should find exactly one
  file, check the count and throw a clear error when it doesn't, before the edit.
- **Run independent work together.** Start independent commands concurrently, for
  example with `Promise.all`.
- **Fail loudly.** Let unexpected errors throw. Use `.nothrow()` only where a failure
  is a result you want to read, such as a test run, and return its exit status or
  failing output.
- **Keep edits safe to re-run.** Match text that exists only before the change.
- **One purpose per program.** Investigate in one, change in the next. Small
  programs are easier for the user to read and for permission checks to judge.
- **Quote paths.** Paths from search output can contain spaces or shell characters.

## Time limits

- Code you wait on stops after ten minutes.
- In auto and `bypassPermissions` mode, long code can run in the background for up
  to two hours while the conversation continues. In other modes, split long builds
  and test runs into steps that each finish within ten minutes.
- When background code finishes, read its result before you report the task done.

## Permissions and safety

- Every call inside your code goes through the user's `deny` and `ask` rules and
  `PreToolUse` hooks under its usual name, such as `Bash` or `Edit`. A denial is an
  answer. Don't rewrite a denied command into a form the rule doesn't match.
- In auto mode the classifier reviews the program, not each call, and doesn't see
  values computed while it runs. Don't pass search output or downloaded text into a
  destructive command in the same program. Return it, look at it, then act.
- The sandbox limits shell commands only. Your JavaScript and its file, web and MCP
  calls run outside it. Ask to run a command outside the sandbox only when it
  failed because of the sandbox, and say why.
- If the user answers **No** at a prompt, the code stops, and changes from calls that
  already finished stay. Report what finished and what didn't.
- If a call is refused with `the hook did not finish` or `the hook failed`, report
  the exact error. Don't retry around the hook.

## Outside Code Mode's reach

- Claude in Chrome, computer use and Remote Control don't exist in Code Mode. When a
  task needs a browser or the desktop, say so and tell the user to start a session
  with `CLAUDE_CODE_EXPERIMENTAL_CODE_MODE=0 claude`.
- The session resumes only with Code Mode on. Mention it when the user plans to pick
  the work up somewhere else.

## Reporting

- The user sees your tool calls, such as diffs, but not your code. Say in plain
  words what the code did: which files changed, which commands ran, what failed.
- When reading the code would help the user, tell them to press `Ctrl+O` and expand
  the **Code** row.
- After code stops early, check `/tasks` for subagents, shell commands and MCP calls
  it started that are still running, and say which ones.

## On this Mac

<!-- Applies on the maintainer's Mac, recognizable by the "This Mac" user
instructions. Only Code Mode deltas belong here; the user file's rules are not
repeated. -->

When your user instructions describe "This Mac", where agents own the setup and the
human observes, these apply too. The user file's rules stand as written; this
section adds only what Code Mode changes about them. On other machines, skip it.

- **No checkpoint but yours.** The default permission mode here is
  `bypassPermissions`: programs run start to finish without a prompt, and long ones
  can go to the background. The separate step for destructive and outward-facing
  actions is the only pause the human gets to see.
- **Checks in one program.** The check commands the user file lists don't depend
  on each other. Run them concurrently in one program and return only the ones that
  fail, with their output.
- **Pinned toolchains.** `$` runs through the same shell setup as the Bash tool, so
  mise's shims give the versions the repository pins. A "not trusted" failure ends
  the program: handle it as the user file says, then run the program again.
- **Sandbox and `.git`.** The Bash sandbox refuses writes to `.git/config` and
  `.git/hooks`, so `git init` and `git config` fail with "Operation not permitted".
  Run them as their own step outside the sandbox, and say why.
- **Credentials.** What a program returns lands in the transcript. Return a
  credential's name or whether it is set, never its value. Changes to credentials go
  through the `vitals-keys` skill.
- **The switch.** Code Mode is on through `CLAUDE_CODE_EXPERIMENTAL_CODE_MODE` in the
  `env` block of `~/.claude/settings.json`. chezmoi doesn't manage that file.

## This project

<!-- Replace this section when you create a repository from the template. Keep only
facts Claude can't read from the code: build and test commands, conventions that
differ from tool defaults, and pitfalls. -->

- Build: `<command>`
- Test: `<command>`; a single test: `<command>`
- Before you report a change done, run the tests that cover it and quote the result.
