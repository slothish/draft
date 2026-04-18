---
title: "Your code is also a context budget"
date: 2026-04-18T12:00:00+02:00
draft: false
tags: ["ovms", "claude-code", "workflow"]
---

There's a lot of writing about how to prompt an LLM. Very little about what happens on the other side — the code the model reads every time it opens a file.

When Claude Code works in a repository, it reads source files. It reads CLAUDE.md on every turn. It reads whatever documentation you've pointed it at. Every comment in every file it touches gets fed into context. Most of that happens invisibly, in the background, before you've typed a single word.

The implication is obvious once you see it: verbosity in your source code and docs isn't just a style question. It's a per-session cost you pay indefinitely.

## What I changed

Two separate passes in the OVMS fork.

The first was source comments. The `.cpp` file had accumulated explanatory blocks — multi-line rationale comments that described why a design decision was made, RE history for how a frame was decoded, context about which ECU owned what. Some of it was useful when it was written. Some had gone stale. All of it was getting read on every file access.

A few examples of the before:

```cpp
// When the bus goes idle, clear transient state that cannot self-correct via CAN.
// Do NOT clear metrics here — they are owned by their respective frame decoders and
// will stale-expire naturally. Clearing charge_inprogress would falsely show "not
// charging" during CCS DC (KCAN silent while charging). Clearing ms_v_env_hvac would
// undo the optimistic update before 0x05EA confirms.
if (just_went_idle) {
    // Clear OCU node presence so no heartbeats are queued on the sleeping bus.
    // Any heartbeat in-flight when the bus dies will be retried by the TWAI hardware
    // until the next wake; clearing m_ocu_active prevents additional frames from
    // piling into the TX queue and driving TEC up before the next CommandWakeup.
    m_ocu_active = false;
```

And after:

```cpp
// On idle: clear OCU node presence so no further heartbeats queue on a dead bus.
// Do NOT clear metrics — they are owned by their decoders and stale-expire naturally.
// (Clearing charge_inprogress here would falsely show "not charging" during CCS DC,
// which keeps KCAN silent while actively charging.)
if (just_went_idle) {
    m_ocu_active = false;
```

Same information, tighter. The reasoning that's non-obvious stays. The redundant outer context goes.

The second pass was documentation. CLAUDE.md is loaded on every turn — it's the first thing in context before any tool call runs. Mine had grown to include the full build procedure: package installs, toolchain setup script, OTA flash command, SSH address. All accurate, none of it needed unless you're about to build the firmware. It went into `docs/dev-guide.md`, which gets read on demand, not constantly.

The architecture section had similar problems — paragraphs of prose explaining the component system, the CAN bus setup, the naming conventions. Some of it was replaced with bullet tables. The rest was true but redundant; the code itself encodes that information.

PROJECT.md got the same treatment. A separate TODO-metrics.md was folded in and the combined document tightened.

## The underlying distinction

There are two kinds of content in source comments and project docs.

The first kind is load-bearing context: the non-obvious constraint, the invariant a reader would violate by refactoring, the specific reason a check that looks redundant actually isn't. This belongs in the code, close to the thing it describes. It earns its tokens.

The second kind is narrative: the history of how a decision was reached, the RE evidence that confirmed a byte offset, the maintainer feedback that ruled out an approach. This belongs in commit messages. It was useful at decision time. It doesn't need to be re-read on every subsequent file access.

Most over-commented code is over-commented because engineers write comments while they're reasoning — they leave the reasoning in place instead of extracting just the conclusion. That's reasonable when you're the only reader. It's a recurring cost when a model is reading the file on every turn.

## What this isn't

This isn't an argument for sparse code or minimal documentation. It's an argument for documentation in the right place. The dev guide still exists, it's just not in CLAUDE.md. The RE history for a frame decode is still in the git log for that commit. The information isn't gone — it's been moved to a location that gets read when it's relevant instead of unconditionally.

If anything, the discipline makes the remaining comments better. When every comment has to justify its tokens, you find yourself being precise about what's actually non-obvious and what the reader — human or model — could figure out from the code itself.
