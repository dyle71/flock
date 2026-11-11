# 0005. One Rust crate with feature and platform modules; Cargo and Bun workspaces side by side

- Status: Proposed
- Date: 2026-11-11

Decided in
[Repository layout, toolchains, stack rules and port series](https://github.com/dyle71/flock/issues/14).

## Context

flock is one repository holding a Rust backend, a Vite + TanStack SPA, an Electron shell, and a
TypeScript interface shared by the SPA and the shell (the desktop bridge of
[0004](0004-a-public-stable-rest-api-code-first-openapi-in-server-mode.md)). The code-first API
pipeline needs a place that writes `docs/api/openapi.json` and the TypeScript types from the
Rust code.

klicker.ing, the model project, separates domain slices from technical capabilities: a domain
slice may import a platform module, never the reverse, enforced by a guard test (klicker.ing
ADR 0003 and its `module-boundaries.md` rule).

A Rust backend can carry such boundaries in two ways: as modules inside one crate, or as separate
crates in a Cargo workspace (for example a store, a sync engine and an API crate), where the
compiler enforces the dependency direction. The maintainer weighed both on 2026-11-11.

## Decision

- The repository root holds a **Cargo workspace** and a **Bun workspace** side by side:
  `server/` is the flock binary (crate `flock`), `xtask/` generates the OpenAPI document and the
  TypeScript types and runs chores, `web/` is the SPA, `desktop/` the Electron shell, and
  `packages/bridge/` the TypeScript interface of `window.flockDesktop`. The Bun workspace covers
  `web`, `desktop` and `packages/*`.
- The backend is **one crate**. Its boundaries are modules: `src/feature/` for the domain slices
  and `src/platform/` for the technical capabilities, in klicker.ing's direction — a feature module
  may import a platform module, a platform module never imports a feature module. A guard test
  fails the unit tier when it does.
- The crate is split into several only when compile time demands it; the module boundaries are
  drawn so that a platform module can then be cut out as a crate of its own.
- The authoritative version is `[workspace.package] version` in the root `Cargo.toml`; every
  `package.json` mirrors it, and a guard checks that they agree.

## Consequences

- The dependency direction rests on a guard test rather than on the compiler; a violating import
  compiles and fails the tests.
- One crate keeps the build and the test setup simple; incremental compile time is the cost to
  watch, and the trigger to split.
- The rule `.claude/rules/module-boundaries.md` carries over from klicker.ing with Rust paths.
- SPA, shell and bridge share one Bun workspace; the bridge interface is a package both import,
  not a copy.
- A version bump touches `Cargo.toml` and every `package.json`; the `version-bump` skill is
  adapted to do both.
