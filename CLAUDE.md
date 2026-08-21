# good-claude

A plain terminal client for Claude Code, for people who want the text and
nothing else. It draws no UI. It spawns the `claude` binary already on the
user's PATH and talks to it through the Claude Agent SDK, so it reuses the
existing Claude Code login, configuration, and billing.

Read `README.md` for the user-facing description. This file is the working
agreement for changing the code.

## Layout

The whole client is one executable Python script, `good-claude`, with a
module-level docstring that mirrors the README. There is no package, no
`setup.py`, and no test suite. Supporting files:

- `requirements.txt` -- one dependency, `claude-agent-sdk`.
- `README.md` -- user-facing documentation.
- `.venv/` -- created by the user, gitignored.

Keep it a single file unless there is a real reason to split it. If a split
becomes necessary, the entry point stays an executable script with a
`#!/usr/bin/env python3` shebang, because that is how it lands on a PATH.

## The rules that make this program what it is

These are not preferences. Violating one of them defeats the point of the
project.

1. **stdout is model text and nothing else.** Everything else -- the banner,
   tool announcements, approval prompts, change previews, tool results, errors
   -- goes to stderr through `meta()`. The test is that
   `good-claude > transcript.txt` captures only what the model said. Never call
   `print()`; use `emit()` / `emit_block()` for model text and `meta()` for
   everything else.
2. **Never add line breaks to model text.** No wrapping, no hard-wrapping to
   terminal width, no reflowing, no indentation, no gutter. The terminal wraps.
   This is the single behavior the project exists to provide.
3. **No TUI.** No curses, no alternate screen buffer, no cursor positioning, no
   progress spinners, no ANSI color, no redrawing. If a change needs to know
   the terminal width, it is the wrong change.
4. **Print whole segments, not tokens.** Model text is emitted a finished
   message segment at a time. Do not reintroduce a typewriter effect.
5. **Stay a thin client.** Do not reimplement anything the CLI already does.
   Session state, permission rules, settings, skills, and plugins belong to the
   CLI. This program's job is I/O.
6. **Enter is never send.** Enter starts a new line, always. A message is sent
   by Ctrl-D on an empty line or by `/send` on its own line. This is what makes
   pasting multi-line text safe.
7. **Never lose typed text.** `/quit` refuses while a message is in progress,
   and Ctrl-D on an empty buffer does nothing. Any new path that could discard
   composed input needs an explicit confirmation.
8. **Every tool call is announced before it runs.** Approval goes through the
   `PreToolUse` hook, not the SDK's `can_use_tool` callback: `can_use_tool`
   only fires when the CLI's permission rules already evaluate to "ask", so any
   tool covered by an allow rule in settings would run unannounced. Do not
   switch to `can_use_tool`.
9. **Tool output is not truncated.** Reading a large file prints the whole
   file. The user asked for the text.
10. **Exactly one thread reads stdin.** `input_loop` is the only place in the
    program that may call `input()`. Two things want typed lines -- the message
    being composed and an approval prompt -- and a tool started in the
    background raises its prompt while the message prompt is already blocked in
    `input()`. Two threads inside `input()` are serialized by readline's own
    lock, so each line goes to whichever one won the race: the answer vanishes
    into the half-typed message and the prompt re-asks forever. Anything that
    needs a line asks `input_loop` for it.

## Concurrency

`screen_lock` is held by whoever owns the terminal: an approval prompt while it
waits for an answer, or the turn loop while it prints. Parallel tool calls fire
hooks concurrently, and one tool's result arrives while the next tool's prompt
is up, so any new code that writes to the terminal from inside the message loop
or a hook must take that lock.

Blocking calls that are not reads of stdin -- file reads, for instance -- run
under `asyncio.to_thread` so the event loop keeps servicing the SDK connection.
Reads of stdin belong to the reader thread (rule 10): the `PreToolUse` hook
hands its question to that thread through `queue_approval` and waits on a
future, and the thread prints and asks. Because the thread is not the event
loop, it takes `screen_lock` through `hold_screen` / `release_screen`, which
schedule the acquire and the release on the loop; `screen_lock` is an
`asyncio.Lock` and must not be turned into a `threading.Lock`, or a prompt
waiting on a human would freeze the SDK connection.

The reader is a daemon thread. It can be sitting in `input()` when the process
ends, and a reader waiting on a human must not hold up the exit.

## Style

- Python 3.10 is the floor. `from __future__ import annotations` is at the top;
  `X | None` annotations are fine.
- PEP8. Type annotations on every function. A docstring on every function and
  on the module.
- Standard library plus `claude_agent_sdk`. Adding a third dependency needs a
  reason; there is no build step to hide one behind.
- Comments explain why, not what. The existing comments about the hook choice,
  the screen lock, and `/send` are the model: each one records a decision that
  is not obvious from the code.
- ASCII only, in code, comments, output, and documentation.

## Testing

There is no test suite. The client is exercised by hand:

- Piped input covers the scriptable path, since piped stdin cannot deliver an
  EOF mid-stream: `printf '%s\n' 'say hi' '/send' '/quit' | ./good-claude`.
  Extra lines after `/send` answer approval prompts, so a tool call can be
  driven the same way.
- The input machinery can be driven without the SDK: import the script with
  `importlib`, start `input_loop` on a thread, and type at it through a pty.
  Give it a pty for stdin and a pipe for stdout. macOS links readline against
  libedit, which returns an empty line for everything when it reads a pty slave
  from a thread, and `input()` only reaches readline when stdout is a terminal
  too.
- Ctrl-C during a turn, Ctrl-D sending, terminal paste, and readline editing
  need a real terminal and can only be checked interactively.
- Check the stdout/stderr split on any change that prints:
  `./good-claude > out.txt 2> err.txt` and confirm `out.txt` holds only model
  text.

When a change touches behavior the README describes, update the README, the
module docstring, and this file together.

## Known gaps

Ctrl-C does not interrupt a turn. The signal is delivered while the event loop
is idle, so it propagates out of `asyncio.run` rather than into the
`except KeyboardInterrupt` in `main`, and the process exits 130 with any typed
text lost. Fixing it means `loop.add_signal_handler` plus running the turn as a
cancellable task; the handlers in `main` are dead code until then.

MCP servers are not configured by this client and are untested. Skills that fan
out to subagents are untested; whether a subagent's tool calls surface their own
approval prompts is unverified.
