# wForge — Architecture

## 0. What wForge Is (and Is Not)

### What wForge Is

**wForge** is an explicit, Rust-first GPU framework built on **wgpu**, designed for:

- GPU **compute and rendering** as first-class workloads
- Explicit, ordered **command recording**
- Fast prototyping that **scales into production**
- Tech art, simulation, rendering, and GPU tooling
- Multiple frontends (WASM first, native later)

wForge prioritizes:

- **Control over convenience**
- **Clarity over abstraction**
- **Honesty over magic**

---

### What wForge Is Not

wForge is **not**:

- A game engine
- A scene graph
- An ECS
- A material system
- A frame graph (in core)
- A high-level rendering abstraction

Those may exist **on top of wForge**, never inside its core.

---

## 1. Core Design Principles (Non-Negotiable)

These principles are binding.
If a feature violates them, it does not belong in core.

---

### 1.1 Explicit Over Implicit

- Users explicitly create resources
- Users explicitly create pipelines
- Users explicitly schedule passes
- Users explicitly control ordering

wForge removes **boilerplate**, not **decisions**.

---

### 1.2 Compute and Render Are Equals

Compute is not a “feature” of rendering.

- Compute passes
- Render passes
- Copy / blit passes (future)

All are **just GPU work**.

---

### 1.3 Frame = Ordered Command Timeline

A frame is **not**:

- “the render loop”
- “the draw call”
- “the scene”

A frame is:

> **A linear, ordered recording of GPU commands**

This model is inspired directly by GPU tooling such as RenderDoc.

---

### 1.4 Rust Owns the GPU

- All GPU state lives in Rust
- Frontends are adapters, not owners
- WASM is a platform target, not an architectural driver

---

### 1.5 Small Now, Grows Cleanly

v1 must be:

- understandable end-to-end
- minimal but powerful
- fast to iterate on

The architecture must allow:

- advanced tooling
- multi-pass workflows
- future optimization layers

**Without breaking the API.**

---

## 2. High-Level Architecture

```
┌──────────────────────────────────┐
│ Frontends                        │
│  - WASM + JS                     │
│  - Native desktop (future)       │
│  - CLI / headless (future)       │
└──────────────▲───────────────────┘
               │ Frontend Adapter API
┌──────────────┴───────────────────┐
│ Application Layer (User Code)    │
│  - App trait                     │
│  - User-defined passes           │
│  - CPU logic                     │
└──────────────▲───────────────────┘
               │
┌──────────────┴───────────────────┐
│ Runtime Core                     │
│  - wgpu device & queue           │
│  - Resource registries           │
│  - Pipeline cache                │
│  - Frame & pass recording        │
└──────────────▲───────────────────┘
               │
┌──────────────┴───────────────────┐
│ Platform Layer                   │
│  - Surface / swapchain           │
│  - Window / canvas integration   │
│  - WASM / native differences     │
└──────────────────────────────────┘
```

Dependencies flow **downward only**.

---

## 3. Workspace & Crate Layout

wForge is a Cargo workspace.

```
wforge/
├─ crates/
│  ├─ wforge-core
│  ├─ wforge-runtime
│  ├─ wforge-app
│  ├─ wforge-frontend
│  ├─ wforge-wasm
│  ├─ wforge-egui (optional)
│
├─ examples/
│  ├─ particles
│  ├─ raytracing-weekend
│
├─ docs/
│  └─ architecture.md
```

---

## 4. Crate Responsibilities

### 4.1 `wforge-core` (Pure Logic)

**No wgpu. No platform code.**

Contains:

- Math types
- Cameras
- Scene / simulation data
- CPU-side algorithms
- Shared data layouts mirrored in shaders

Goals:

- Fully testable
- Reusable outside GPU contexts
- Keeps GPU code honest

---

### 4.2 `wforge-runtime` (GPU Engine Core)

**The heart of wForge.**

Responsibilities:

- wgpu `Instance`, `Adapter`, `Device`, `Queue`
- Resource creation & lifetime management
- Pipeline creation & caching
- Command encoder creation
- Command submission

This crate:

- Knows nothing about frontends
- Knows nothing about user application logic

---

### 4.3 `wforge-app` (User-Facing API)

Defines the **programming model** users interact with.

Contains:

- `App` trait
- `Context`
- `Frame`
- Pass recording API

This is the primary crate users depend on.

---

### 4.4 `wforge-frontend` (Frontend Contract)

Defines how frontends integrate with wForge.

Contains:

- Frame loop interface
- Event forwarding traits
- Surface abstraction hooks

Allows:

- WASM frontend
- Native desktop frontend
- Headless / offline frontend

---

### 4.5 `wforge-wasm` (WASM Adapter)

Responsibilities:

- Canvas creation
- Event forwarding
- Browser frame loop integration
- Surface presentation

Contains **no GPU logic decisions**.

---

### 4.6 `wforge-egui` (Optional Plugin)

- egui integration
- Input forwarding
- Rendering pass helpers

Never required by core.

---

## 5. The User Programming Model

### 5.1 `App` Trait

The user entry point.

```rust
pub trait App {
    fn init(ctx: &mut Context) -> Self;
    fn update(&mut self, ctx: &mut Context, dt: f32);
    fn frame(&mut self, ctx: &mut Context, frame: &mut Frame);
}
```

Semantics:

- `init`: allocate GPU resources, pipelines
- `update`: CPU-side logic only
- `frame`: record GPU work

This mirrors real engines and GPU tools.

---

### 5.2 `Context`

Long-lived GPU state and registries.

```rust
pub struct Context {
    device: wgpu::Device,
    queue: wgpu::Queue,

    buffers: BufferRegistry,
    textures: TextureRegistry,
    pipelines: PipelineRegistry,
}
```

Properties:

- Centralized ownership
- Explicit access
- Safe defaults with escape hatches

Raw access is always allowed:

```rust
ctx.device()
ctx.queue()
```

---

## 6. Resources

### 6.1 Explicit Creation

Resources are explicitly created and returned as handles.

```rust
let particles = ctx.buffers.create(BufferDesc {
    size: N * size_of::<Particle>(),
    usage: STORAGE | VERTEX | COPY_DST,
});
```

wForge does **not**:

- Resize automatically
- Guess usage
- Track logical read/write hazards

wgpu handles GPU synchronization.

---

### 6.2 Resource Identity

Resources:

- Have stable identities
- Persist across frames
- Are reusable without restriction

Future extensions may include:

- debug labels
- inspection hooks
- capture/replay support

---

## 7. Pipelines

### 7.1 Pipeline Creation

Pipelines are immutable and cached.

```rust
let pipeline = ctx.pipelines.compute(ComputePipelineDesc {
    shader: "shaders/particles.wgsl",
    entry: "update",
    bind_layouts: &[&layout],
});
```

Design goals:

- WGSL is user-authored
- Bind layouts are explicit
- Pipelines are long-lived

---

## 8. Frame & Pass Model (Core of wForge)

### 8.1 Frame Philosophy

A `Frame` represents:

> **A linear, ordered recording of GPU commands**

Inspired by:

- RenderDoc
- Nsight
- Low-level engine internals

Frames are:

- deterministic
- replayable (conceptually)
- inspectable

wForge does not reinterpret or reorder work.

---

### 8.2 Frame Structure

```rust
pub struct Frame<'a> {
    encoder: wgpu::CommandEncoder,
    ctx: &'a Context,
}
```

Rules:

- All GPU work must occur inside a frame
- Frames are short-lived
- Submission happens once per frame

---

### 8.3 Pass Recording

Frames consist of an **ordered list of passes**.

Users may record:

- Any number of compute passes
- Any number of render passes
- In any order

```rust
frame.pass(|pass| {
    pass.compute()
        .pipeline(&simulate)
        .bind(0, &particles)
        .dispatch(groups);
});

frame.pass(|pass| {
    pass.compute()
        .pipeline(&sort)
        .bind(0, &particles)
        .dispatch(groups);
});

frame.pass(|pass| {
    pass.render()
        .pipeline(&draw)
        .draw(count);
});
```

Execution order **exactly matches recording order**.

---

### 8.4 Pass Semantics

Passes are:

- Mechanical scopes
- Command groupings
- Debuggable units

They are **not**:

- Logical systems
- Automatically optimized
- Reordered or merged

---

### 8.5 No Fixed Output Model

wForge does not assume:

- a swapchain
- a final render pass
- a screen output

Frames may:

- Render offscreen
- Run compute-only workloads
- Perform offline processing

---

## 9. Multi-Pass Workflows (First-Class)

The architecture explicitly supports:

- Compute → compute → render
- GPU sorting pipelines
- GPU-driven rendering
- Ray tracing pipelines
- Simulation chains

Examples:

- Particle simulation
- Prefix sum + compaction
- Ray generation + accumulation + tonemapping

Each stage is a **separate pass**.

---

## 10. CPU–GPU Interaction

### 10.1 CPU (`update`)

- Input handling
- Parameter updates
- Buffer uploads
- High-level decisions

No command encoding.

---

### 10.2 GPU (`frame`)

- Compute passes
- Render passes
- Copy / readback (future)

---

### 10.3 Readback & Feedback

Architecture allows:

- Async buffer mapping
- CPU feedback loops
- GPU-driven logic across frames

No forced async model is imposed.

---

## 11. Frontend Architecture

### 11.1 Frontend Role

Frontends are **adapters**, not controllers.

They:

- Create surfaces
- Forward events
- Drive the frame loop
- Present results

They do **not**:

- Create pipelines
- Manage resources
- Decide GPU work

---

### 11.2 Frontend Integration Flow

```
Frontend
  ↓
Creates Context
  ↓
Runs App::init
  ↓
Loop:
  - collect events
  - call App::update
  - create Frame
  - call App::frame
  - submit frame
```

---

### 11.3 WASM Frontend (`wforge-wasm`)

Responsibilities:

- Canvas setup
- Browser event forwarding
- RequestAnimationFrame loop
- Surface presentation

All GPU logic remains in Rust.

---

### 11.4 Native Frontend (Future)

Will:

- Use the same frontend contract
- Share App and Runtime code
- Differ only in platform glue

---

## 12. Debuggability & Tooling (Architectural Hooks)

The architecture is designed to support:

- Pass labeling
- Resource labeling
- Capture/replay tooling
- GPU debugging

Even if not implemented initially, the **structure supports it**.

---

## 13. Non-Goals (Explicit)

wForge core will not include:

- Frame graphs
- Automatic resource aliasing
- ECS
- Scene management
- Asset pipelines
- Material systems

These are layered **on top**, not inside.

---

## 14. Growth Path (Additive Only)

Future extensions may include:

- Optional frame graph layer
- Shader hot reloading
- Async compute helpers
- GPU profiling hooks
- RenderDoc integration helpers

None of these should require core API redesign.

---

## 15. v1 Milestone (Concrete & Achievable)

wForge v1 must support:

- WASM frontend
- Explicit Context & Frame
- Multiple compute passes
- Render passes
- Explicit resource & pipeline creation
- egui parameter tweaking
- Particle system example

If this is clean and pleasant to use, the foundation is correct.

---

## 16. Final Guiding Question

Before adding anything to core, ask:

> **Does this reduce boilerplate without removing control or hiding ordering?**

If not, it does not belong in wForge core.
