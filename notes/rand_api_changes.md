# rand crate API changes — textbook vs current version

Reference: `lib/src/crypto.rs` line 16

---

## The change

The textbook was written against `rand` 0.8. This project uses `rand` 0.10.1.

```rust
// textbook (rand 0.8)
SigningKey::random(&mut rand::thread_rng())

// fixed (rand 0.10.1)
SigningKey::random(&mut rand::rng())
```

---

## Why it changed

`thread_rng()` was renamed to `rng()` in rand 0.9. The functions are identical — it was
a pure rename for clarity.

**Source:** `rand-0.10.1/CHANGELOG.md`, line 127:
```
- Rename fn `rand::thread_rng()` to `rand::rng()` and remove from the prelude (#1506)
```

---

## What `rand::rng()` actually does

Source: `~/.cargo/registry/src/index.crates.io-.../rand-0.10.1/src/rngs/thread.rs`, line 201:

```rust
pub fn rng() -> ThreadRng {
    let rng = THREAD_RNG_KEY.with(|t| t.clone());
    ThreadRng { rng }
}
```

Returns a `ThreadRng` — a handle to a thread-local cryptographically secure random number
generator. "Thread-local" means each thread gets its own RNG instance, so no locking or
contention between threads.

`&mut rand::rng()` passes a mutable reference to that handle into `SigningKey::random()`,
which uses it to generate the random bytes for the private key.

---

## Other renames in rand 0.9

The same release renamed several other functions. If you hit similar errors from the
textbook code:

| Old (rand 0.8) | New (rand 0.9+) | Source |
|---|---|---|
| `rand::thread_rng()` | `rand::rng()` | CHANGELOG line 127 |
| `rng.gen()` | `rng.random()` | CHANGELOG line 131 |
| `rng.gen_range(...)` | `rng.random_range(...)` | CHANGELOG line 132 |
| `rng.gen_bool(...)` | `rng.random_bool(...)` | CHANGELOG line 132 |

---

## How to verify a rename is safe

When the textbook uses an API that no longer exists, check the crate's `CHANGELOG.md`
in the local cargo cache:

```
~/.cargo/registry/src/index.crates.io-.../rand-0.10.1/CHANGELOG.md
```

A "Rename" entry means behaviour is identical — safe to update.
A "Remove" or "Breaking change" entry warrants more investigation.
