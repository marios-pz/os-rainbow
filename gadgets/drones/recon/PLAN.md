# Recon drone, plan

The software plan for the recon drone gadget. This is about architecture and milestones, not parts. Anything physical (boards, drivers, motors, wiring) is decided and documented elsewhere, and is deliberately kept out of this file so the plan stays true even if the hardware changes.

## Goal

A remotely driven recon drone, controlled from a phone over a direct wireless link, streaming a live camera feed back with the lowest practical latency. It can drive (two independent sides), send video, and perform a one-shot "jump" action with a cooldown. The control and video logic is shared platform code; only the parts that talk to the board are specific to this gadget.

## Principles

- Keep all hardware-independent logic in the shared platform crates. This gadget's own crate holds only the glue: wiring the platform logic to whatever board runs it, plus the main loop.
- The wire format is defined once in the protocol crate and is the single source of truth for both this gadget and the app. Never redefine it here.
- Latency beats quality. Prefer the newest data over complete data: drop stale frames and out-of-order packets rather than buffering them.
- Safety first. The failsafe and the jump cooldown are invariants, not features, and must survive every refactor.
- Build in small milestones that each leave something testable. Do not start a milestone before the one it depends on works.

## Architecture

Three layers, from shared to specific:

1. Platform (shared, reused by every gadget)
   - protocol: the control packet and the video packet layouts, plus encode/decode. Pure logic, testable on a normal computer.
   - video: framing a camera image into packets and reassembling it, newest-frame-wins policy. Transport-agnostic.
   - radio: setting up the direct link and sending/receiving datagrams with power saving off.

2. Gadget logic (the drone-core crate, still hardware-independent where possible)
   - Steering mix: turn a single drive-plus-steer intent into a value for each side.
   - Failsafe timer: if no fresh valid control packet arrives within the timeout, command a full stop.
   - Jump state machine: accept a jump request only when ready, run one jump cycle, then enforce the cooldown.
   - Sequence handling: ignore duplicate or out-of-order control packets.

3. Board glue (this crate, the only hardware-aware part)
   - Map the gadget logic's outputs to the board's actual outputs, and the board's inputs back into the logic.
   - Own the main loop timing.
   - Keep this layer as thin as possible so a different board means rewriting only this part.

## Data flow

- Control, phone to drone: small fixed-size datagrams at a steady rate. A guard byte rejects stray packets, a sequence number orders them, the payload carries a single drive-plus-steer intent (mixed into per-side values on the drone by the steering mix) and a flags byte for actions like jump.
- Video, drone to phone: each frame is compressed, split into chunks that fit a single datagram, and sent to whoever last sent a valid control packet. The receiver reassembles the newest frame and discards anything older or incomplete.
- Two video profiles: a driving profile tuned for lowest latency, and a parked observation profile tuned for clarity, chosen by a command from the app.

## Milestones

Each milestone is done when it is demonstrable and, where it is pure logic, covered by tests.

1. Protocol: define and test the control and video packet layouts in the protocol crate. Round-trip encode/decode tests pass on a normal computer.
2. Gadget logic: implement steering mix, failsafe, jump state machine, and sequence handling as pure functions/state machines with unit tests. No board involved yet.
3. Minimal output: bring the logic up on a board and prove one real output responds to a hard-coded input. Smallest possible "it lives" step.
4. Drive: wire both sides through the steering mix so a local hard-coded command drives them correctly, including the failsafe stopping them on silence.
5. Link: stand up the direct wireless link and drive the sides live from control packets, with the failsafe cutting in when packets stop.
6. Jump: add the jump action end to end, proving the cooldown cannot be bypassed.
7. Video: stream the camera to a simple receiver with the newest-frame-wins policy, then measure real latency and tune.
8. Profiles: add the driving vs parked video profiles switchable from the app side.
9. App integration: a real phone client sends control and renders video, closing the loop.

## Testing strategy

- Pure logic (protocol, steering, failsafe, jump, sequence) is unit-tested on a normal computer and must not depend on the board.
- Board glue is validated by staged bring-up: one output, then one subsystem, then the whole loop, never all at once.
- Latency is a measured number from real runs, not an assumption. Record it and track it as profiles and code change.

## Open questions

- Exact control rate and failsafe timeout: pick initial values, then tune against measured feel.
- Video chunk size and frame-drop policy details: start simple, refine after the first latency measurement.
- How the app selects a video profile: define the command in the protocol crate once decided.
