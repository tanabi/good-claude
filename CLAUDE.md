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
   the terminal width, it is the wrong change. There is exactly one escape
   sequence in the program, `ESC[?2004l` in `disable_bracketed_paste`, and it
   is not drawing: it sets an input mode, it exists to undo a mode another
   program turned on, and it is written only when stderr is a terminal. Adding
   a second one needs the same standard of justification.
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
10. **A turn is not over while a delegated agent is still running.** `run_turn`
    reads past the first `ResultMessage` and returns only when a result arrives
    with nothing in flight, because a background agent outlives the turn that
    started it and its completion wakes the model for a follow-up turn on the
    same connection. Returning early puts the message prompt up over the middle
    of that: the agent's output is not read off the connection until the user
    sends something unrelated, which then gets answered together with it. Track
    only `DEFERRED_TASK_TYPES`; waiting on a backgrounded shell would park the
    session forever, since one may never reach a terminal status.
11. **Nothing else may write to the terminal.** The SDK gives the claude
    subprocess this process's own stderr unless a `stderr` callback is
    registered, so without one the subprocess writes to the user's terminal
    directly -- control sequences included -- on top of a half-typed message,
    and `screen_lock` cannot stop it because it is a different process.
    `cli_stderr` keeps it on a pipe and strips control characters from what it
    prints. Do not drop that callback to "simplify" the options.
12. **Pasting must survive libedit.** Python's `readline` is libedit on macOS
    and on any Homebrew/linuxbrew python3, GNU readline on most Linux distros.
    libedit does not understand bracketed-paste markers and mangles them into
    the text, so the terminal's paste mode is turned off before every read.
    Test paste changes on a libedit build specifically; a GNU-readline box will
    not show the bug.
13. **Subagents run in the foreground.** `foreground_input` rewrites agent
    calls through the hook's `updatedInput`, so it holds whatever the model
    intended. Backgrounding buys parallel conversations, which a single prompt
    on a single terminal cannot use. The rewrite happens before the
    announcement, the approval prompt, and the session allow list, so all three
    describe the call that will actually run. Note the tool backgrounds agents
    when the field is *absent*, so anything that is not an explicit `false` has
    to be rewritten. Never extend this to `Bash(run_in_background=True)`: that
    is a dev server or a `tail -f`, and forcing it to finish first hangs the
    turn forever.
14. **Exactly one thread reads stdin.** `input_loop` is the only place in the
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
Reads of stdin belong to the reader thread (rule 14): the `PreToolUse` hook
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
- Paste needs a libedit python3 and a pty whose parent tracks mode 2004: turn
  bracketed paste on from the child, paste with the markers only while the mode
  reads as on, and check the collected message equals what was sent. On a GNU
  readline build the bug cannot reproduce, and pasting short lines does not
  reproduce it either -- the markers are the whole story.
- To see whether anything is writing to the terminal behind the client's back,
  run it under a pty and grep the captured bytes for escape sequences. The only
  one that should appear is `ESC[?2004l`.
- The background-agent wait is driven the same way. Ask for one backgrounded
  `Agent` call, feed `y` lines to approve it and whatever it runs, and end with
  `/discard` then `/quit` so leftover answers do not become a second message.
  The turn is correct when a `[waiting for N background agent...]` line appears
  after the first response and the agent's findings print before the prompt
  returns. `run_turn` can also be driven without the CLI: import the script and
  hand it a fake client whose `receive_messages` yields a scripted list of
  `TaskStartedMessage` / `ResultMessage` / `TaskNotificationMessage` frames.
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

MCP servers are not configured by this client and are untested.

Text pasted while a turn is running is echoed twice: once by the tty driver as
it arrives, because nothing is reading stdin then, and again by readline when
the prompt returns and reads those bytes. Verified on a pty on both readline
backends, so it is not the libedit bug and the bracketed-paste fix does not
touch it. The message text is correct; only the screen doubles. The fix is an
ECHO toggle around every read via `termios`, which was declined: a crash with
ECHO off leaves the user typing blind until they run `stty sane`. Revisit only
with a restore path that survives a crash and a signal.

A subagent's tool calls do surface their own approval prompts, through the same
`PreToolUse` hook, and are answered as they happen (verified against a
backgrounded `Agent` call).

The wait on background agents has one hole, which is the SDK's, not this
client's: an agent that reaches a terminal state *before* the result of the turn
that started it leaves nothing in flight at that result, so the turn ends and
the prompt comes back with a continuation still on its way. Nothing in the task
bookkeeping can tell that apart from no work at all; closing it needs a
run-boundary signal from the CLI. The "press Enter to answer" path is the
fallback for exactly this case and must stay.
