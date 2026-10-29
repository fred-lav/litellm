# Rust and Axum

## Existing pattern

- `crates/gateway` composes endpoint routers and owns server lifecycle
- Endpoint crates inject state with `.with_state(...)`, such as `crates/gateway-inference` using `Arc<Gateway>`
- Follow the owning crate's `AGENTS.md` before adding a database consumer

## Database state scaffold

- This is a proposed wiring shape, not an existing gateway database API
- The runtime supplies an initialized pool

```rust
use axum::Router;
use sqlx::PgPool;

#[derive(Clone)]
struct DatabaseState {
    pool: PgPool,
}

fn with_database(routes: Router<DatabaseState>, pool: PgPool) -> Router {
    routes.with_state(DatabaseState { pool })
}
```

- Handlers extract `State<DatabaseState>` and call typed `litellm_db` operations with the pool or transaction
- Extend existing endpoint state or use `FromRef` when other dependencies are already present
- Cloning `PgPool` shares its handle, it does not create another pool
- Keep SQL in `crates/db`, and response conversion and HTTP error mapping in the endpoint crate
- Reuse the same query APIs as Python's bridge, without creating a second SQL implementation

## Lifetime and verification

- Apply the [connection ownership rules](../SKILL.md#connection-ownership), finish active requests and database jobs before closing the pool
- Preserve authentication, optional-database behavior and transaction boundaries for the specific route
- Verify HTTP behavior through the owning router and query behavior against real PostgreSQL

## Framework references

- [Axum state and substates](https://docs.rs/axum/latest/axum/extract/struct.State.html)
- [SQLx pool ownership](https://docs.rs/sqlx/latest/sqlx/struct.Pool.html)
