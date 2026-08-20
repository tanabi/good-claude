# good-claude

I hate Claude Code's TUI client. I have very few good things to say about it. The worst part is, it enforces a two-space 'gutter column' that breaks copying and hard new-lines which means if you copy/paste out of the terminal and into pretty much anything else, you will have to hand-massage it to work. This is a property of the TUI client and not something claude can control no matter how much you cajole it to do so.

Further, the multi-line input sometimes acts curiously. Sometimes things glitch out and become misaligned or otherwise look weird. Someone spent way too much time making TUI look fancy without any particular thought to if it is day-to-day usable. And I'm not the only person who hates it. There are dozens of requests to fix these problems in Claude's issue repo but no fix in sight after months.

So this is a plain terminal client for Claude Code, for people who want the text and nothing else. `good-claude` draws no UI at all. It prints the model's text to stdout as whole message segments, with no added line breaks, and lets the terminal do the wrapping and scrolling it already knows how to do. It bypasses the "typewriter" streaming effect as well, and just gives you the straight finished block of text.

Everything that is not model text -- the banner, tool announcements, approval prompts, errors -- goes to stderr instead, so `good-claude > transcript.txt` captures only what the model actually said.

It is a thin client, not a reimplementation. It spawns the `claude` binary you already have and talks to it through the Claude Agent SDK, so it uses your existing Claude Code login and your existing configuration, and it bills the same way your normal `claude` sessions do. There is no separate API key and no separate API bill, unless you have an `ANTHROPIC_API_KEY` set in your environment, which the CLI would prefer over your subscription login.

## Requirements

Python 3.10 or newer, and a working `claude` CLI on your PATH that you have already logged in to (run `claude` once and complete the login if you have not).

## Install

Clone the repository and install the SDK into a virtualenv:

```
git clone <this repo> good-claude
cd good-claude
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

Optionally put it on your PATH with a symlink:

```
ln -s "$PWD/good-claude" ~/.local/bin/good-claude
```

## Use

Activate the virtualenv, then run it:

```
source .venv/bin/activate
good-claude
```

It is an ordinary Python script with an ordinary `#!/usr/bin/env python3` shebang, so it runs under whichever interpreter is first on your PATH. With the virtualenv active that is the virtualenv's Python, which is the one that has the SDK. Without it, you get a `ModuleNotFoundError` for `claude_agent_sdk`, which means you forgot to activate.

Run it from the directory you want to work in. Your working directory matters exactly as much as it does in the normal client: project `CLAUDE.md` files, project skills, and project settings are loaded from it.

Typing works like a text box, not like a chat prompt. Enter only ever starts a new line, so a stray Enter cannot fire off a half-written message, and pasted multi-line text arrives as one prompt rather than as a dozen accidental ones. When the message is ready, press Ctrl-D on an empty line to send it. Ctrl-D with nothing typed does nothing at all.

Four commands are recognized, each on a line of its own:

- `/send` sends the message, the same as Ctrl-D. It is also what makes the client scriptable, since piped input has no way to deliver an EOF mid-stream.
- `/discard` throws away the message you are composing.
- `/quit` exits. It refuses while you have unsent text, so an in-progress message is never lost to a stray quit; `/discard` it first.
- `/exit` is an alias for `/quit`.

While a turn is running, `Waiting for response...` is printed to stderr, and Ctrl-C interrupts the turn and returns you to the prompt. On exit, the session id is printed with the command to pick that conversation back up: `good-claude --resume <id>`.

Session ids are also appended to `~/.good-claude/session-history.txt` as they are created, so a conversation is still recoverable when the terminal, the ssh connection, or the whole machine dies before you ever see that exit line.

Skill-backed slash commands work, because the client loads your user, project, and local settings sources. Anything in `~/.claude/skills`, a project's `.claude/skills`, or an installed plugin can be invoked the usual way, for example `/my-skill`. The TUI's own built-in commands, such as `/config`, `/model`, and `/resume`, are implemented by the terminal client rather than by a skill, so they do not exist here.

## Tool approval

Every tool call is presented for approval before it runs:

```
Tool request: Bash
  git status --short
allow? [y] once  [a] always this exact call  [t] always any Bash  [n] no  [v] view input:
```

`y` allows this one call. `a` allows this exact call for the rest of the session, and means the call and not the tool: approving `git status --short` approves `git status --short`, and the next Bash command is a new question. `t` is the blanket answer, allowing every call to that tool for the rest of the session, which for `Bash` means every shell command; it says so when you pick it. `n` denies the call and then offers to attach a reason that is passed back to the model, and `v` prints the tool's full input as JSON when the one-line summary is not enough to decide.

Two calls count as the same call when their inputs match, ignoring the `description` field, which the model rewords freely without changing what runs.

A call that would change a file shows the change itself, as a diff, so you are never asked "may I edit this file" without being told how:

```
Tool request: Edit
  /tmp/notes/sample.txt
   alpha
  -beta
  +BETA
   gamma
allow? [y] once  [a] always this exact call  [t] always any Edit  [n] no  [v] view input:
```

`Edit` diffs the text being replaced against its replacement, and says so when the edit replaces every occurrence rather than one. `Write` diffs the file on disk against the new contents, or prints the whole thing as an addition when the file does not exist yet. `NotebookEdit` prints the new cell source and says which cell it lands in and whether it is being inserted, replaced, or deleted. Tools that do not write files -- `Bash`, `Read`, `Grep` and the rest -- are unchanged; the one-line summary is all they get.

The preview stops at 60 lines so that a large rewrite cannot scroll the question off the screen. When it is cut short it says how much was left out, and `v` still shows the entire input. A call that runs unprompted, because you approved it with `a` or approved its tool with `t`, prints its one-line announcement only, without the diff; the tool result that follows reports what changed.

Note that `a` is of little use on the writing tools. Two edits are the same call only when they change the same text in the same file the same way, which is rare, so `Edit` and `Write` will keep asking. That is the intended direction: a diff you have already read is the thing you approved, and the next one deserves its own look. `t` is there when you want to stop being asked at all.

Once a tool runs, its output is printed under a `[tool result]` header, or `[tool error]` when the call failed. Both go to stderr along with everything else that is not model text. Results are printed in full and are not truncated, so reading a large file prints the whole file.

This is enforced with a `PreToolUse` hook rather than the SDK's `can_use_tool` callback. That distinction matters: `can_use_tool` only fires when the CLI's permission rules already evaluate to "ask", so any tool covered by an allow rule in your settings would run without ever being announced. The hook sees every call regardless of permission rules. Approval prompts are serialized, so parallel tool calls queue up rather than fighting over your terminal. A prompt also owns the terminal while it waits: anything the turn wants to print, including the previous tool's result, is held until you have answered, so nothing is dumped on top of the question you are being asked.

## Environment

- `GOOD_CLAUDE_MODEL` picks a model, for example `haiku` or `sonnet`. Defaults to whatever your CLI is configured to use.
- `GOOD_CLAUDE_THINKING=1` prints thinking text along with the response.

## Limits

MCP servers are not configured by this client and have not been tested with it. Skills that fan out to subagents have not been tested either; whether a subagent's tool calls surface their own approval prompts is unverified.

Ctrl-C during a turn and terminal paste behavior both need a real terminal to exercise, so neither is covered by the piped-input testing this was built with.
