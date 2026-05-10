---
title: "The CAN capture that lied"
date: 2026-05-10T00:00:00+02:00
draft: false
tags: ["ovms", "debugging", "claude-code"]
---

The OVMS module talks to the car over two buses. CAN2 handles powertrain — BMS, gear selector, VIN. CAN3 handles the body network — speed, GPS, clima, locks. The J533 gateway sits between them and bridges selected frames. Not all traffic. Specific IDs it's programmed to forward.

Early captures used a script with a bus-labeling bug. Powertrain frames from CAN2 were written to the capture file with bus tag `3` — the body network tag. In the logs, frames like `0x131` (SoC), `0x187` (gear selector), and `0x191` (BMS current) appeared to be arriving on CAN3.

Drew the wrong conclusion: J533 was bridging all CAN2 traffic onto CAN3. Should have caught it. Let Claude run instead. The commit added `IncomingFrameCan3(p_frame)` inside `IncomingFrameCan2` — every powertrain frame forwarded to the KCAN decoder, with a comment explaining the architecture I'd just invented. That went into upstream master as PR #1369.

During OBC AC charging, the car floods CAN2 at a high frame rate. With the forwarding in place and no hardware filter on the mcp2515, every frame hit the KCAN decoder ISR. Events task starved. Task Watchdog Timer fired. Module reset. On restart it loaded the last-persisted SoC from NVS — stale. Charging continued. Loop repeated. SoC stuck at a stale value.

The fix removed the forwarding and added a hardware filter on the mcp2515. That stopped the reset loop. It also broke SoC — the filter blocks `0x131` from reaching `IncomingFrameCan2`, and without the forwarding there's no fallback. SoC stuck at 28.5% across driving and charging. A regression from the fix.

The frames were never on CAN3. J533 does not bridge them — except `0x131`. It does forward that one, and that's now the workaround: decode it in `IncomingFrameCan3` as a fallback until the mcp2515 RXB0 delivery issue is understood. One wrong assumption, two bugs.

---

What would have stopped it: treat "J533 bridges all traffic" as a hypothesis, not a conclusion. CAN3 is a comfort bus — it doesn't generate a high frame rate. If those frames were really coming from CAN3 via J533, the rates alone kill it. That check was never done.

So we built a Claude Code skill for this. `/troubleshoot` is built on Kepner-Tregoe: state the problem, fill IS/IS NOT, name a hypothesis and what would kill it — before any action. "Take a clean capture with explicit bus filter to verify bus assignment" would have been a required step. The labeling bug surfaces there. The forwarding never ships.

One faulty data source, no adversarial check, conclusion shipped to upstream. The skill makes the adversarial check mandatory — not optional review after the fix, required first.
