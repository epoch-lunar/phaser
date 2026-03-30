# TODO

## What this is

`phaser` is a pure Rust library crate. It is the time-scale math kernel for the Epoch ecosystem. It has no HTTP server, no binary, no runtime, and no Cloudflare-specific dependencies. It compiles to native (flight hardware, ground stations) and to WASM (Cloudflare Worker, browser).

**It is not an application.** Nothing in this repo serves requests or renders anything. Other repos import it.

## What it contains

The functions being extracted from `epochlunar.com/backend/src/lib.rs`:

- `true_anomaly(utc_ms: f64) -> f64` — Kepler's equation solver (8 Newton iterations) returning the Moon's true anomaly in radians
- `lunar_drift_us(utc_ms: f64) -> f64` — TCL (Lunar Coordinate Time) drift in µs since J2000, with first-order eccentricity correction

Plus thin wrappers or re-exports around `hifitime` for the standard time scales consumed by callers:
- TAI, TT, TDB from UTC/Unix input
- GPS week + time-of-week
- TCG, TCB offsets from TT

The exact boundary between "phaser owns this" and "callers use hifitime directly" is an open question — resolve it as the first real consumers (the Cloudflare Worker and dash) make their needs clear.

## Cargo.toml shape

```toml
[lib]
crate-type = ["cdylib", "rlib"]
# rlib: used by epochlunar.com Worker and dash (Rust imports)
# cdylib: future WASM/npm package or PyO3 Python bindings

[dependencies]
hifitime = "4"
# no `worker`, no `js-sys`, no `wasm-bindgen` here
```

## Commands

```bash
cargo test          # run all unit tests
cargo build         # verify it compiles to native
cargo build --target wasm32-unknown-unknown  # verify WASM target
```

## Constants (from the existing implementation)

```rust
const RATE_M: f64   = 56.0199;             // lunar secular rate, µs/day
const J2000_MS: f64 = 946_728_000_000.0;   // Unix ms at J2000.0
const PERIGEE_MS: f64 = 1_705_142_100_000.0; // ref perigee 2024-01-13T10:35Z
const T_SID_S: f64  = 27.321661 * 86_400.0; // sidereal month, seconds
const E_MOON: f64   = 0.0549;              // lunar orbital eccentricity
```

These are not magic numbers — they have physical meaning. Don't change them without a citation.

## How other repos use this

**epochlunar.com Worker** — git dependency in production, path override locally:
```toml
# epochlunar.com/backend/Cargo.toml
phaser = { git = "https://github.com/epoch-lunar/phaser" }
```
```toml
# epochlunar.com/.cargo/config.toml (local dev only)
[patch."https://github.com/epoch-lunar/phaser"]
phaser = { path = "../../phaser" }
```

**dash** — same pattern:
```toml
# dash/Cargo.toml
phaser = { git = "https://github.com/epoch-lunar/phaser" }
```

## Python bindings

A `python/` directory with PyO3 bindings (`phaser-py`) is planned but not started. It would expose the same functions to ground station operators and Jupyter notebooks. Keep the core library `no_std`-friendly where possible so the Python bindings don't constrain embedded targets.

## Open questions

- How much of `hifitime` should phaser re-export vs. let callers depend on directly?
- Does `python/` live here or in a separate `phaser-py` repo?
