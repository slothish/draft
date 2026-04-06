---
title: "How I use Claude Code on OVMS"
date: 2026-04-06
draft: false
tags: ["ovms", "claude-code", "workflow"]
---

The OVMS development manual has a section on AI. It's worth reading in full, but the short version:

> "Vibe" submissions will be rejected and closed without comment.

And:

> Submitting unvalidated "AI" output is a waste of our time and an insult to our dedication to the Open Source idea of sharing knowledge and helping each other to grow.

The maintainer isn't wrong. The PRs he's seen from people pointing an LLM at the repo and submitting whatever comes out are, by his account, consistently bad — wrong function calls, invented APIs, code that compiles but misunderstands how the framework actually works. OVMS is a custom embedded environment running on ESP32. It has its own event system, its own metric types, its own CAN poller, its own naming conventions. Nothing in an LLM's training data maps cleanly onto it.

The failure mode isn't that the generated code is obviously wrong. It's that the code looks plausible — professional, even — until you understand the project well enough to see what's wrong with it. That's actually worse than code that visibly fails.

So the question isn't whether to use Claude Code. It's how to use it in a way that the output is actually correct.

## Ground it first

Before writing a line of code, I had Claude Code read the project. Not a summary — the actual source. The development manual. The PR where initial e-Golf support was added, and the maintainer's review comments on it. The reference vehicle implementations the maintainer points to as examples of how things should be done. The base class headers. The existing e-Golf code.

Then I wrote a CLAUDE.md in the fork. Not a list of generic instructions — a document that encodes what you'd need to know to contribute to this specific project without embarrassing yourself. The key constraints that aren't obvious from the code alone. The maintainer's review rules, extracted from his actual PR feedback. The naming conventions. The architecture of the CAN bus setup in the e-Golf specifically. The fact that climate control uses a proprietary protocol called BAP that the standard OVMS poller can't handle, requiring direct CAN frame construction rather than the built-in polling mechanism.

This is the work that most people skip. They open the repo, write a prompt, and submit what comes back. The CLAUDE.md is what separates "Claude Code has read the docs" from "Claude Code understands the project well enough to work in it."

## Close the loop with real hardware

Context alone isn't enough. Claude Code will still make mistakes — especially in a custom framework this far from anything in its training data. The difference is catching them before they become a PR.

The validation loop is: write code → compile → run unit tests → OTA flash to the OVMS module → verify against the car. The repo has a pre-push hook that blocks pushes on feature branches unless tests pass. The build script runs tests first and aborts on failure. Nothing gets submitted to the module that hasn't compiled clean.

For the car side, OVMS can log CAN traffic directly. There's a capture script that SSHs into the module, grabs a frame dump while the car is in a specific state, and saves it with a description as a `.crtd` file and a companion markdown. The same capture can then be replayed against new firmware builds without the car needing to be in that state again. When something doesn't decode right, the raw frames are there.

OVMS shell output — metrics, log messages, command responses — goes into markdown files that Claude Code reads back. That's the feedback loop. Not "does this look right" — "does this produce the correct metric value when I replay the capture."

## What this gets you

Claude Code operating this way doesn't generate plausible-looking code. It generates code that compiles, passes tests, and decodes the right value from the right CAN frame. It catches its own mistakes when they fail the build. It references the framework correctly because it's read the framework.

The next post covers two concrete cases where the loop caught something — one where a test failure made the mistake obvious, and one where the diagnosis required hardware logs from the car itself.

The maintainer's concern is valid — as a description of how most people use these tools. It's not a description of what the tools are capable of.
