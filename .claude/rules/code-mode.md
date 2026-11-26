# Code Mode

<!--
Maintainers: the template's Code Mode rules. They load in every session in a
repository made from the template, and in every project on machines that copy this
file into ~/.claude/rules/. Rules only: facts and trade-offs live in README.md,
setup in CONTRIBUTING.md. Keep it under 200 lines (ccc refuses to merge over 500).
HTML comments like this one are stripped before Claude sees the file.
-->

These rules apply when you have a tool named `REPL`. That tool runs your code, and
it exists only when Code Mode is on. Without it, skip this file and work with direct
tool calls.

In Code Mode, instead of asking for one tool call, reading the result and deciding
the next, you write a short JavaScript program that makes the calls, and Claude Code
runs it in the session. This is how you work whenever Code Mode is on. The facts and
trade-offs behind these rules: https://github.com/ANcpLua/claude-code-mode-template

## When to write code, when to make one call

Write code when a task needs several related tool calls:

- The same change across many files
- A search whose results need filtering before you read them
- Independent commands that can run at the same time
- A chain where one result feeds the next call: find the file, edit it, test it

Make a single, separate call when:

- One read or one command answers the question
- The action is destructive or reaches outside the machine: deleting files,
  discarding uncommitted changes, `git push`, publishing, deploying, sending
  messages. Give it its own step so the user sees it on its own, and never bury it
  in a batch
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
- **Read a cut output from its file.** When a long output comes back cut, with a
  path to the uncut text, search that file or read the lines you need. Don't run
  the code again to see the rest.
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
- **Pass large data by file.** Write a long list or a file's bytes to a file and
  pass its path. A very large typed array in a tool's input is refused.
- **Return names, not secrets.** What a program returns lands in the transcript.
  Return a credential's name or whether it is set, never its value.

## Time limits

- Code you wait on stops after ten minutes.
- Long code can run in the background for up to two hours while the conversation
  continues, in a session that started in auto or `bypassPermissions` mode and is
  still in it. Otherwise split long builds and test runs into steps that each
  finish within ten minutes.
- When code waits although the mode is auto or `bypassPermissions`, split the work
  the same way and tell the user that a new session started in that mode runs code
  in the background.
- When background code finishes, read its result before you report the task done.
- A slow command can move to the background and leave your program with empty
  output. When you are told it did, wait for the command's result. Don't report the
  empty output as what the command printed.

## Permissions and safety

- Every call inside your code goes through the user's `deny` and `ask` rules and
  `PreToolUse` hooks under its usual name, such as `Bash` or `Edit`. A denial is an
  answer. Don't rewrite a denied command into a form the rule doesn't match.
- In `bypassPermissions` mode nothing pauses between calls, and in auto mode the
  classifier reviews the program, not each call, without seeing values computed
  while it runs. Don't pass search output or downloaded text into a destructive
  command in the same program. Return it, look at it, then act.
- The sandbox limits shell commands only. Your JavaScript and its file, web and MCP
  calls run outside it. Ask to run a command outside the sandbox only when it
  failed because of the sandbox, and say why.
- If the user answers **No** without a comment, or presses `Esc`, at a prompt, the
  code stops, and changes from calls that already finished stay. Report what
  finished and what didn't. A **No** with a comment refuses only that call, and the
  code can go on.
- If a call is refused with `the hook did not finish` or `the hook failed`, report
  the exact error. Don't retry around the hook.

## Outside Code Mode's reach

- Claude in Chrome, computer use and Remote Control don't exist in Code Mode. When a
  task needs a browser or the desktop, say so and tell the user to start a session
  with `CLAUDE_CODE_EXPERIMENTAL_CODE_MODE=0 claude`.
- The session resumes only with Code Mode on. Mention it when the user plans to pick
  the work up somewhere else.
- After a resume, the variables of your earlier programs are gone. Compute a value
  again before you use it.

## Reporting

- The user sees your tool calls, such as diffs, but not your code. Say in plain
  words what the code did: which files changed, which commands ran, what failed.
- When reading the code would help the user, tell them to press `Ctrl+O` and expand
  the **Code** row.
- After code stops early, check `/tasks` for subagents, shell commands and MCP calls
  it started that are still running, and say which ones.
