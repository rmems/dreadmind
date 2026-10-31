# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

dreadmind trains an adaptive "pursuit director" for Nemesis in Resident Evil 3 Remake. The goal is a persistent stalker that learns how the player plays, closer to the Alien: Isolation alien. The core is Rust; the game side is Lua running under REFramework.

The design source of truth is the Notion page "dreadmind — Research Guide: How to Build the Nemesis Brain", under "stalker-director — Adaptive Nemesis AI (RE3 Remake)". Check it before making architectural decisions.

## Current state

The repo is pre-code: there is no `Cargo.toml` and no Lua or Rust source yet. Once the crate exists, use the usual Cargo commands: `cargo build`, `cargo test`, `cargo test <name>` for a single test, `cargo clippy`, and `cargo fmt`. Add any project-specific commands here once they exist.

## Architecture (planned)

It uses a two-brain director pattern from Alien: Isolation and the Left 4 Dead AI Director. **The learned model does not drive Nemesis's movement.** The game already handles pathing, attacks and animation. The brain answers three questions every few seconds:

1. **Where** Nemesis heads: a target room/zone, discrete over N zones.
2. **How aggressive** he is: mode `{stalk, ambush, chase, retreat}` plus an aggression scalar in 0–1.
3. **When** to back off so tension resets: L4D cycle Build Up → Sustain Peak (3–5 s) → Peak Fade → Relax (30–45 s).

Data flow:

- **Lua (REFramework)** reads player state about every 0.5 s: zone, position, facing, health, ammo, healing items, time since the last encounter or damage, events (gunshot, door transition, pickup, save room), and Nemesis state. It applies the director's action and keeps Nemesis from despawning across rooms.
- **Bridge**: start with a local socket between Lua and the Rust process because it's easier to debug. Move to a DLL later.
- **Rust director**: observation vector, then model, then action. Slow player-habit features are rolling-window stats: favourite rooms, time per room, save-room retreats, shoot-vs-run, backtracking.
- **Reward**: positive for a "scare" (damage dealt, player forced out of a room, resource used). Negative when Nemesis is trivially dodged, stuck or absent too long, or when intensity stays maxed too long.
- **Guard**: a deterministic layer clamps aggression and enforces a minimum relax time over any learned output. The rule-based director is the fallback if the learned brain misbehaves.

Model options, in build priority:

- **Baseline (required):** a rule-based director with an L4D state machine and an Alien-style menace gauge, plus a tiny non-spiking GRU policy. Every learned model is compared against these.
- **A: Liquid State Machine (first learned model):** a fixed random LIF reservoir of 200–500 neurons, about 80/20 excitatory/inhibitory and about 10% connectivity. It has an exponential spike trace readout (τ ≈ 20–50 ms) and trains only a linear readout: ridge regression offline, RLS/FORCE or reward-modulated Hebbian online. Tune the reservoir to the edge of chaos before touching the readout. Feed the slow habit features straight to the readout, because the liquid's memory is only a few hundred ms.
- **B: ANN+SNN hybrid:** spiking actor with an ANN critic (PopSAN), or ANN→SNN conversion.
- **C: pure SNN with R-STDP:** a research comparison only, not the first thing to ship.

Use dependencies, not forks, from the rmems neuromorphic stack. Fork only if core changes are needed:

| Stage | Crate |
| --- | --- |
| Spike encoding (population coding) | axon-encoder |
| Rolling habit features | kinetic-signals |
| Liquid connectivity and neuron dynamics | synaptic-wiring, neuromod |
| Reward | limbic-critic |
| Readout learning / R-STDP | plasticity-lab |
| Hybrid ANN stage (Option B) | hybrid-fusion, cortex-tensor |
| Guard rails | vahtisiru, thalamic-relay |
| Playthrough logs and perf cost | gaming-telemetry |
| Weight save/load | engram-parser |
| NIR export / FPGA stretch goal | nir-rs, silicon-bridge |

## Build order

1. Lua prototype: read player room/health/ammo and make Nemesis follow across two rooms. Use `sdk.get_managed_singleton(sdk.game_namespace("PlayerManager"))` and `get_manualPlayer`, with refdocs.praydog.com as the reference.
2. Rust socket bridge plus the rule-based director. This ships as v0.1.
3. Telemetry logging (JSONL/CSV) plus a map-graph simulator: rooms as nodes, doors as edges, and scripted player presets (runner, fighter, hoarder) for offline RL and director unit tests. The real game is too slow and non-deterministic to be the only environment.
4. Option A offline, compared against the baseline.
5. Online readout learning in-game behind the guard.
6. Options B and C as comparison experiments.

Evaluation covers four things. The adaptation test checks whether encounters shift toward a repeatedly used hiding spot. The tension curve plot should show peaks and valleys. Compute cost is measured as ms per decision tick. Ablations cover liquid vs none, online learning on vs off, and population vs rate coding.

## CI

`.github/workflows/claude.yml` answers `@claude` mentions as `claude[bot]`, pinned to `claude-fable-5-1`. It is deliberately not allowed to run `cargo` or install a toolchain: build scripts and tests would execute PR code with the subscription token in scope. Keep it that way.
