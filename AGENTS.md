# AGENTS.md — @netasibas/map-sdk

## Current state

This repo contains only `ARCHITECTURE.md` — a vision/roadmap document. **No code, build config, or tests exist yet.** Everything in ARCHITECTURE.md is planned, not implemented, unless explicitly marked otherwise.

## When implementation begins

### Monorepo structure
- pnpm workspace root with `packages/core` (TypeScript SDK), `packages/wasm-core` (Rust), `packages/playground` (demo app)
- Root `Cargo.toml` is a Rust workspace; `pnpm-workspace.yaml` is the JS workspace

### Build tooling
- TypeScript: Vite library mode, ESM-only output
- Rust: `wasm-pack` with `wasm32-unknown-unknown` target
- Dev server: Vite serves `packages/playground` for live map testing

### Critical architectural constraints
- **MapLibre owns the render loop** — never call `requestAnimationFrame`; use `map.triggerRepaint()` to request frames
- **Single GL context** — use `map.painter.context.gl` from `onAdd`; no second canvas
- **WASM memory views are ephemeral** — never hold a view across another WASM call or an `await`; use immediately and release
- **WASM cannot call GL** — all WebGL2 draw calls must go through JavaScript
- **State management**: `@preact/signals` for consumer-facing state, plain mutable for hot-path internals, event emitter for notifications

### Key reference
- `ARCHITECTURE.md` — full architecture vision, data flows, error handling, and implementation phases

## Agents

| Agent | Model | Mode | Use for |
|-------|-------|------|---------|
| `orchestrator` | qwen3.7-max | primary (default) | Multi-step tasks, coordination, decomposition |
| `build` | qwen3.7-max | primary | Complex implementation, heavy code changes |
| `plan` | qwen3.7-max | primary | Architectural decisions, system design, trade-offs |
| `explore` | qwen3-vl-32b-instruct | subagent | Codebase exploration, file discovery, pattern search |
| `visual` | qwen3-vl-32b-instruct | subagent | Screenshot review, map rendering debugging, visual QA |
| `quick` | qwen3-vl-32b-instruct | subagent | Simple edits, refactors, documentation, Q&A |
