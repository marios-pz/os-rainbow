# Project Rainbow

An open ecosystem and firmware platform in Rust for building the gadgets of Rainbow Six Siege in real life, starting with the recon drones. Not military-grade, not a 40k budget: affordable prototypes, built to a standard that leaves room to upgrade later.

## Vision

Each Siege operator's device becomes a real, buildable gadget that shares one common platform. The first target is the ground recon drone: two drive sides, a live camera feed to a phone, and a spring jump. Later gadgets may include a flying drone and a cooperative "jammer" that only ever affects our own devices. The platform is written once; each gadget builds on top of it.

This is a learning project built seriously. The developer is new to Rust and to hardware but wants the full picture, not a simplified one. So: explain everything that matters, leave nothing important unsaid, and keep the language as plain as the topic allows. Depth and clarity are not in conflict here, prefer a clear explanation of a hard thing over hiding it. When a concept is genuinely new, define it the first time it appears, then use it normally. Never talk down to the developer; assume someone sharp who is simply early in the material.

## How to work in this repo

- Favour small, reviewable changes with a clear reason for each.
- When introducing anything non-obvious, a tricky constraint, an unsafe block, a latency trade-off, explain the "why" in a comment or the commit message, not just the "what".
- Correct a wrong decision rather than building on it. Past code is not sacred.
- Measure claims about latency or timing against real values where possible, instead of asserting them.
- Don't over-engineer ahead of need. Fill a gadget or crate in only when it is actually being worked on.

## Terminology (be precise)

- This is a **platform / framework**, not an operating system. It is shared Rust code that each gadget's firmware depends on. Call it that.
- "Rainbow Six" and operator names are Ubisoft trademarks. Fine for a personal/learning project; avoid leaning on them in any public package name.
- A **microcontroller** runs the firmware directly, with no OS. A single-board computer runs Linux and is a different kind of target with different trade-offs.

## Repository layout

```
project-rainbow/
├── platform/
│   ├── protocol/   # wire formats: control packet, video packet, encode/decode
│   ├── video/      # framing a frame into packets and reassembling it, newest-frame-wins
│   └── radio/      # the direct link, sending/receiving datagrams
├── gadgets/
│   ├── drone-core/    # drone logic, no board: steering mix, failsafe, jump, sequence handling
│   ├── recon-drone/   # first gadget, the ground drone (see its own PLAN.md)
│   ├── yokai-drone/   # placeholder, flying drone, not started
│   └── mute-jammer/   # placeholder, cooperative jammer, not started
└── app/            # mobile app
```

The root is a Cargo workspace (a container of crates), not a crate itself. The `platform/*` crates are libraries; the gadgets and the app are the things that use them.

The hard rule that makes this ecosystem work: **hardware-independent logic (everything in `platform/*` and the pure logic of a gadget) must never pull in chip-specific or std-only dependencies.** It stays portable, so a new board only needs a thin new glue layer with its pin map and drivers, while the logic and the wire format are reused unchanged. Board-specific code lives only in the glue layer for that board.

## Per-gadget plans

Each gadget keeps its own `PLAN.md` describing its architecture and milestones in software terms. Hardware choices (boards, drivers, wiring) are intentionally kept out of the plan files so a plan stays valid even if the hardware changes. When working on a gadget, read and keep its `PLAN.md` current. The recon drone's plan is the live one; the other gadgets are placeholders.

## Architecture, shared shape across gadgets

Three layers, from shared to specific:

1. **Platform (shared):** protocol, video, radio. Pure logic and transport, reusable by every gadget, testable on a normal computer.
2. **Gadget logic (per gadget, still hardware-independent):** the decision-making for that device, for example a drone's steering mix, failsafe timer, jump state machine, and sequence handling. Pure functions and state machines with unit tests, no board involved.
3. **Board glue (per gadget, the only hardware-aware part):** maps gadget logic to the real board's inputs and outputs and owns the main loop timing. Kept as thin as possible so a different board means rewriting only this.

## Data flow (shared design)

- **Control, phone to gadget:** small fixed-size datagrams at a steady rate. A guard byte rejects stray packets, a sequence number orders them and lets the gadget drop duplicates or out-of-order packets, and the payload carries the intent plus a flags byte for actions like jump.
- **Video, gadget to phone:** each frame is compressed, split into chunks that fit a single datagram, and sent to whoever last sent a valid control packet. The receiver reassembles the newest frame and discards anything older or incomplete. Never buffer; buffering is what adds lag.
- **Two video profiles:** a driving profile tuned for lowest latency, and a parked observation profile tuned for clarity, selected by a command from the app.

## Link and latency principles

- The phone joins the gadget's own wireless access point, with no router in between. Wireless power-save is turned off; this is the single biggest latency win.
- Latency beats quality. Prefer the newest data over complete data.
- Latency is a measured number from real runs, not an assumption. Record it and track it as code and profiles change.

## Build, run, test

- `platform/*` and the pure logic of a gadget are plain host crates: no board dependencies, so `cargo test` runs them on the computer. Keep them that way and unit-test the wire format and the gadget logic here.
- Board glue targets the chip and is flashed to the board, not run on the host.
- Build only what you need with `cargo build -p <crate>`; `cargo build` builds the whole workspace. Being a workspace member does not force a crate to build every time.

## Conventions

- Crate and directory names: lowercase-with-hyphens; in code, hyphens become underscores.
- Do not name a crate `core`; that name is taken by Rust's own core library. Use a descriptive name instead.
- Keep `platform/*` and gadget logic `no_std`-friendly where practical, for portability to any future microcontroller.
- Treat the packet byte layout in `protocol` as the single source of truth for both firmware and app. Never redefine it in two places.
- Comment the reasoning behind hardware-facing and unsafe code; every `unsafe` block gets a one-line justification of why it is sound.

## Safety-critical invariants (never regress these)

- **Failsafe:** if no valid control packet arrives within the timeout, outputs go to a safe stop. Top priority, must survive refactors.
- **Action cooldowns:** a one-shot action like the jump cannot re-fire before its cooldown elapses or its cycle completes.
- **No output without hardware protection:** never drive an output in a way that only works if a protection part is missing. The firmware assumes that protection is present because it depends on it.
- **Staged bring-up:** when testing new wiring, power the board alone first, then add one subsystem at a time. Never energise a freshly wired full circuit in one go.

## Development

- Follow Rust best practices: clear ownership, small focused functions, `Result` over panics in library code, no needless `clone`, meaningful names, and the API guidelines spirit (predictable, hard to misuse).
- Keep code formatted and linted at all times. Run `cargo fmt` before every commit and keep `cargo clippy` clean (treat warnings as things to fix, not ignore). Both `cargo fmt --check` and `cargo clippy` should pass locally and in CI.
- Do not use em dashes anywhere, in code, comments, documentation, or commit messages. Use a comma, a colon, or two sentences instead.
- Write comments and documentation in a humanized, simple voice. Explain the "why" in plain language a tired person can follow, keep sentences short, define a term the first time it appears, and skip anything that just restates the code.
