---
name: database-access
description: Organize database repositories and proxy dependency wiring, manage connection ownership, or port Prisma operations to Rust and SQLx without expanding migration scope
metadata:
  internal: true
---

# Database access

## Adding new database access

- Put Python database access behind an operation in `litellm/repositories`
- Give each repository its own FastAPI provider, so a route builds only the repositories it uses
- A provider borrows existing handles and performs no database I/O during construction
- Domain protocols are optional substitution boundaries, not required for every repository
- Give callers typed inputs and outputs, not Prisma clients, models, or query dictionaries
- Keep query execution inside the repository, with one implementation per operation and no backend switch or fallback
- Prisma is fine for the implementation, adding a query does not require a Rust port
- Native Rust callers use `litellm-rust/crates/db` directly

## Framework wiring

- Python routes receive a per-repository dependency such as `UserBannerRepositoryDep` and call its operations, such as `banners.get_banner()`
- Use named FastAPI providers with `Annotated[..., Depends(...)]`, ordinary helpers and jobs receive explicit arguments
- Preserve each route's no-database behavior, authorization precedence and background-task scheduling
- Read [Python and FastAPI](references/python-fastapi.md) for provider examples, lifespan state and route verification
- Read [Rust and Axum](references/rust-axum.md) when adding database access to Rust handlers

## Connection ownership

- Keep connection setup and shutdown separate from repositories, which own no pools
- Reuse existing handles, never connect per request or disconnect from a repository operation
- Give each pool one owner for setup and shutdown, borrowed handles never close it
- Finish requests, jobs and queued database work before closing connections
- Clean up resources opened before a startup failure, without a second disconnect fallback
- Add a backend to the existing lifecycle owner and pass its handle only to repositories that need it
- Preserve cancellation, timeout, error and transaction semantics across the native bridge

## Changing ownership or reloading resources

- Inventory initialization, reload, tests, background jobs and shutdown together
- Move every writer when changing authoritative storage, copying a replaceable global into state is insufficient
- A frozen repository does not make its clients immutable
- Decide whether each request captures one resource generation or reads it again during processing
- Build and validate replacements before publishing them, preserving the old generation on failure
- Serialize reloads and coordinate publication with shutdown
- Drain streams and background users before retiring old resources, retaining every generation grows resource use
- Define multi-worker coordination separately from an in-process swap
- Choose mechanisms such as `ArcSwap` only after defining publication, capture and retirement

## Asked to move existing access behind a repository

- Identify the calls being moved and their transaction boundary
- Move the existing Prisma implementation into the relevant repository
- Replace those direct database calls with repository operations
- Preserve return values, errors, and transaction behavior, with focused regression tests
- Keep this an extraction, not a backend migration or unrelated cleanup

## An equivalent Rust or SQLx query already exists

- Leave it and the current backend selection unchanged unless integration is explicitly requested
- Its existence alone does not justify switching a Prisma caller to Rust
- Repository extraction can keep using Prisma even when the Rust query is available

## Asked to port a Prisma query to Rust or SQLx

- Identify the existing inputs, outputs, errors, and transaction boundary
- Check `litellm-rust/crates/db` for an equivalent query and reuse it if its behavior matches
- Otherwise follow `litellm-rust/crates/db/AGENTS.md`
  - Put static SQL in `queries/` and compile-check it with SQLx
  - Keep typed inputs and reusable row projections in `src/`
  - Refresh SQLx query metadata after SQL changes
- Keep Python conversion in the Python bridge, not in `litellm-rust/crates/db`
- Expose typed operations through PyO3, never an API accepting SQL text, table names or column names from Python
- Map dynamic filters and sort keys to closed Rust enums selecting static SQL, and bind every value
- Switch all callers of the operation and delete its Prisma implementation in the same change
- Move each transaction as a whole, including reads, writes and locks
- Verify behavior against real PostgreSQL before switching callers
  - Check relevant null handling, conflicts, ordering, errors, and transaction semantics
- Repository extraction can happen in the same PR, without converting unrelated routes to dependency injection
- Read [incremental migration](references/python-fastapi.md#incremental-migration) for the caller, repository and wiring changes

## Migration context and boundaries

- Long-term direction is to centralize Python database access in repositories, then replace their Prisma implementations incrementally with Rust APIs
- Mixed Prisma and Rust implementations are expected during the transition, unrelated existing calls can stay as they are
- Prisma and SQLx keep separate client pools to PgBouncer, which pools connections to PostgreSQL
  - Size both layers together and preserve reader/writer routing and pooling-mode compatibility
- PgBouncer does not share transactions between Prisma and SQLx, keep each atomic unit of work on one backend and transaction
- Never retry a failed Rust write through Prisma when it may already have committed
- Prisma owns schema migrations today, eventual SQLx migration ownership and Prisma removal are separate planned steps, not part of a query port
