# wForge

**wForge** is an explicit, Rust-first GPU framework built on **wgpu**, designed for **compute and rendering as equals**.

It is focused on:

- GPU-driven programs (simulation, rendering, tooling)
- Explicit control over resources, pipelines, and passes
- Fast prototyping that can scale into production
- WASM-first frontend support, with native frontends planned

wForge reduces **boilerplate**, not **power**.

---

## What wForge Is

- A low-level but structured GPU framework
- A foundation for tech art, simulation, and rendering tools
- A place to build multi-pass GPU workflows (compute → compute → render)
- Inspired by real-world GPU software and tools (e.g. RenderDoc-style thinking)

---

## What wForge Is _Not_

- A game engine
- A scene graph or ECS
- A material system
- A frame graph (in core)

Those are meant to be built **on top**, not inside.

---

## Core Ideas

- **Explicit command recording**
  Frames are ordered timelines of GPU work.

- **Compute and render are equals**
  No special-casing, no hidden systems.

- **Rust owns the GPU**
  Frontends are thin adapters (WASM, native, headless).

- **Small core, scalable design**
  Minimal now, extensible later without breaking the API.

---

## Documentation

- 📐 **[`docs/architecture.md`](docs/architecture.md)**
  The single source of truth for wForge’s design and structure.

This document defines:

- the programming model
- pass & frame semantics
- frontend integration
- non-goals and growth paths

If it’s not compatible with `architecture.md`, it doesn’t belong in core.

---

## Status

🚧 **Early development**

The initial milestone focuses on:

- WASM frontend
- Explicit Context / Frame API
- Multiple compute + render passes
- Particle system & ray tracing examples

---

## Philosophy (One Sentence)

> **wForge is for people who want full control over GPU work without fighting boilerplate—or being lied to by abstractions.**
