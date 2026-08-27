# good-claude

A plain terminal client for Claude Code, for people who want the text and
nothing else. It draws no UI. It spawns the `claude` binary already on the
user's PATH and talks to it through the Claude Agent SDK, so it reuses the
existing Claude Code login, configuration, and billing.

Read `README.md` for the user-facing description. This file is the working
agreement for changing the code.

## The design principle

**good-claude exists to be copied and pasted into, and copied and pasted out
of.** That is the first goal and it outranks everything else in this file. The
stock client is bad at it, which is the reason this one exists. A change that
makes paste less reliable is wrong even if it improves something else.

The second goal is no terminal jank: no stray newlines, no odd characters, no
escape sequences on screen, nothing decorating the text. The plainest thing
that does the job.

When a trade-off comes up, resolve it in that order. Paste fidelity, then
plainness, then everything else. The rules below are what those two goals have
turned into so far.

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
3. **No TUI.** No curses, no alternate screen buffer, no progress spinners, no
   ANSI color, no redrawing. If a change needs to know the terminal width, it
   is the wrong change. Two exceptions exist, both on the input path and both
   there to serve paste fidelity, and neither draws anything: `ESC[?2004l` in
   `disable_bracketed_paste`, which turns off a terminal input mode another
   program turned on, and the `\b \b` the reader writes to rub out an erased
   character, which is what the terminal's own echo did before the reader took
   echo over. A third needs the same standard of justification.
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
12. **No line editor on the input path.** Bytes are read raw and assembled
    into lines by `consume_input`. This is not a preference: on macOS, libedit
    rewrites a large paste into repeated fragments of itself at the correct
    total length, and GNU readline corrupts it differently. Measured on the
    affected machine with the same 2207-character text -- a plain read was
    byte-exact, both editors were not. Canonical mode is out for the same
    reason in a different form: `MAX_CANON` is 1024 on macOS and truncates a
    long pasted line without saying so. Do not reintroduce `input()`,
    `readline`, or canonical reads on a terminal. Anything added to
    `consume_input` interprets bytes that might be pasted, so the bar is: would
    this corrupt a document that happens to contain that byte? Tab is the
    worked example -- it is indentation, not "complete".
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
- The reader can be driven without the SDK: import the script with `importlib`,
  call `enter_raw_mode`, and drive `read_line` through a pty. No line editor is
  involved any more, so the old macOS-libedit-on-a-thread caveat is gone.
- Paste changes need the whole matrix, and it is cheap to run: a multi-kilobyte
  document compared by md5 against its source, the same document arriving while
  a turn is running, a single line longer than 4096, backspace, Ctrl-U, Ctrl-W,
  arrow keys, UTF-8, backspace over a multi-byte character, and tab-indented
  text. Every one of those has been a real bug or a near miss.
- The corruption this reader exists to prevent does not reproduce on Linux, on
  either readline backend, at any terminal speed. Do not conclude from a clean
  Linux run that a paste change is safe; the failing case was macOS, and the
  discriminating test was a plain read versus a line editor on that machine.
  The raw reader was confirmed working on that macOS box over ssh, against the
  same document that had been failing, so the approach is settled and only
  regressions are in question. What is *not* settled is any future change to
  the input path: it inherits a fix that cannot be regression-tested on Linux,
  so it needs a paste on a real macOS terminal before it is believed.
- To see whether anything is writing to the terminal behind the client's back,
  run it under a pty and grep the captured bytes for escape sequences. The only
  one that should appear is `ESC[?2004l`.
- The background-agent wait is driven the same way. Ask for one backgrounded
  `Agent` call, feed `y` lines to approve it and whatever it runs, and end with
  `/discard` then `/quit` so leftover answers do not become a second message.
  The turn is correct when a `[waiting for N agent...]` line appears
  after the first response and the agent's findings print before the prompt
  returns. `run_turn` can also be driven without the CLI: import the script and
  hand it a fake client whose `receive_messages` yields a scripted list of
  `TaskStartedMessage` / `ResultMessage` / `TaskNotificationMessage` frames.
- Ctrl-C during a turn and Ctrl-D sending need a real terminal and can only be
  checked interactively.
- Anything that leaves the process without running `restore_terminal` leaves
  the terminal without echo. If a test run ends with a dead-looking terminal,
  `stty sane` fixes it.
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

The terminal is in raw mode for the length of the session, restored by a
`finally` in `main` and by an `atexit` handler. Neither runs on `SIGKILL`, and
that leaves the terminal without echo until `stty sane`. This was accepted
knowingly: holding the mode for the session is what keeps text pasted during a
turn out of the canonical queue, where `MAX_CANON` would truncate it, and it is
also what ended the double echo of type-ahead. Adding `SIGTERM` and `SIGHUP`
handlers that restore and re-raise would narrow the window further and has not
been done.

There is no history and no arrow-key editing, and that is a deliberate
consequence of rule 12 rather than an oversight. Adding either back means
adding key interpretation to the paste path, which is where the bug lived.
Anything on that path has to be safe for a byte that arrives inside a pasted
document.

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
