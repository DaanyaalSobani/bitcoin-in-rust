# Custom Serde — the `with` pattern

Reference: `lib/src/crypto.rs`, lines 10–33

---

## The orphan rule — why we can't just write `impl Serialize for SigningKey`

You might wonder: why not just write this?

```rust
impl Serialize for SigningKey<Secp256k1> { ... }
```

The **orphan rule** blocks it. You must own at least one of the two sides:

| Scenario | Allowed? |
|---|---|
| `impl MyTrait for MyType` | ✓ own both |
| `impl MyTrait for ForeignType` | ✓ own the trait |
| `impl ForeignTrait for MyType` | ✓ own the type |
| `impl ForeignTrait for ForeignType` | ✗ own neither — blocked |

`Serialize` is from `serde` (foreign) and `SigningKey` is from `ecdsa` (foreign).
You own neither — orphan rule violation.

Compare to why `sha256` *can* write:
```rust
impl Sha256Digest for &[u8] { ... }
impl Sha256Digest for &Vec<u8> { ... }
```
The `sha256` crate owns `Sha256Digest` — so it can implement it for any type, including
foreign ones like `&[u8]` and `&Vec<u8>`. That's the "own the trait" case.

`#[serde(with = "signkey_serde")]` sidesteps the orphan rule entirely — you never implement
`Serialize for SigningKey` at all. Serde only needs `Serialize for PrivateKey` (which you
own), and when it reaches the `SigningKey` field it calls your `signkey_serde::serialize()`
function directly instead of looking for a trait impl on `SigningKey`.

```
Serialize for PrivateKey       ← you own PrivateKey ✓
  └─ field: SigningKey
       └─ call signkey_serde::serialize()  ← your function, no trait impl needed
```

---

## Are you stuck if you need a foreign trait on a foreign type?

Not quite — the standard workaround is the **newtype pattern**: wrap the foreign type in
your own struct, then implement the foreign trait on your wrapper. You own the wrapper, so
the orphan rule is satisfied.

```rust
// can't do this — own neither
impl Serialize for SigningKey<Secp256k1> { ... }

// can do this — you own MySigningKey
struct MySigningKey(SigningKey<Secp256k1>);

impl Serialize for MySigningKey {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: Serializer {
        serializer.serialize_bytes(&self.0.to_bytes())
    }
}
```

This is actually exactly what `PrivateKey` is — a newtype wrapper around `SigningKey`.
The `#[serde(with = "...")]` approach is just a more ergonomic way to achieve the same
thing without having to impl the full trait manually on the wrapper.

**Your options when you need a foreign trait on a foreign type:**

| Option | When to use |
|---|---|
| Newtype wrapper + `impl Trait for Wrapper` | General case — works for any trait |
| `#[serde(with = "...")]` | Serde-specific — cleaner when the foreign field is inside your own type |
| Enable a feature flag on the foreign crate | When the crate offers it (e.g. `serde` feature) |
| You're actually stuck | If none of the above apply — this is rare but possible |

---

## The problem

```rust
#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct PrivateKey(#[serde(with = "signkey_serde")] SigningKey<Secp256k1>);
```

`PrivateKey` wraps `SigningKey<Secp256k1>` from the `ecdsa` crate.
`#[derive(Serialize, Deserialize)]` works by calling the same traits on every field —
but `SigningKey` doesn't implement serde's `Serialize`/`Deserialize`.

This is the orphan rule again:
- you don't own `SigningKey`
- serde doesn't own `SigningKey`
- neither can add `Serialize` to it

There is no feature flag to enable this time. The escape hatch is a **custom serde module**.

---

## The attribute — line by line

```rust
#[serde(with = "signkey_serde")]
```

This is a **field-level serde attribute** — it is **not a Rust language feature**.
`#[serde(...)]` attributes are only meaningful when serde's derive macro is present to read
and act on them. The derive macro runs at compile time, sees the `with` attribute, and
generates different code for that field. Without `#[derive(Serialize, Deserialize)]` above
the struct, `#[serde(with = "...")]` does nothing at all.

Other derive macros like `#[derive(Clone)]` or `#[derive(Debug)]` have no equivalent
escape hatch — they simply require every field to implement the trait. Serde built this
override mechanism itself; it is not something Rust provides automatically to all macros.

It sits on the field inside the struct:

```rust
pub struct PrivateKey(
    #[serde(with = "signkey_serde")]  // ← on the field, not the struct
    SigningKey<Secp256k1>
);
```

It tells serde: "don't try to derive serialize/deserialize for this field automatically.
Instead, use the `serialize` and `deserialize` functions from the module named
`signkey_serde`."

Serde will look for exactly two functions in that module:
- `signkey_serde::serialize`
- `signkey_serde::deserialize`

If either is missing or has the wrong signature, it won't compile.

**The function names are not arbitrary** — they must match the method names defined in
serde's own traits:

```rust
// serde source — ser/mod.rs:234
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: Serializer;
}

// serde source — de/mod.rs:554
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where D: Deserializer<'de>;
}
```

The `with` macro generates code that calls `signkey_serde::serialize` and
`signkey_serde::deserialize` in exactly the shape these trait methods define. Your module
functions are essentially providing a **fake trait implementation** — matching the trait
method signatures — without actually implementing the trait.

If you named them `encode`/`decode` instead, it would fail to compile because serde's
generated code looks for `serialize` and `deserialize` specifically.

Also notice **`use serde::Deserialize` is present but not `use serde::Serialize`**:

```rust
mod signkey_serde {
    use serde::Deserialize;  // ← present
    // no use serde::Serialize
```

`serde::Serializer` is referenced by full path (`S: serde::Serializer`) so no `use` is
needed. But `Vec::<u8>::deserialize(deserializer)` calls the `.deserialize()` method from
the `Deserialize` trait — and in Rust a trait must be in scope to call its methods. So
`use serde::Deserialize` is there purely to bring that method call into scope on line 30.

---

## The module

```rust
mod signkey_serde {
    use serde::Deserialize;
    ...
}
```

A private nested module. It only needs to be visible to the parent module — not `pub`.
`super::` is used inside it to refer back up to the parent module's types
(`super::SigningKey`, `super::Secp256k1`).

---

### `serialize` — converting the key to bytes

```rust
pub fn serialize<S>(
    key: &super::SigningKey<super::Secp256k1>,
    serializer: S,
) -> Result<S::Ok, S::Error>
where
    S: serde::Serializer,
{
    serializer.serialize_bytes(&key.to_bytes())
}
```

**Parameters:**
- `key` — a reference to the field being serialized (`&SigningKey<Secp256k1>`)
- `serializer: S` — the serializer serde passes in (could be JSON, CBOR, binary, etc.)

**`where S: serde::Serializer`** — a trait bound saying S must be any serde serializer.
This makes the function generic over the output format — the same function works for JSON,
CBOR, or any other format.

**`Result<S::Ok, S::Error>`** — associated types on the `Serializer` trait. `S::Ok` is
whatever success type the serializer uses, `S::Error` is its error type. You don't need to
know the concrete types — the trait defines them.

**Body:**
```rust
serializer.serialize_bytes(&key.to_bytes())
```
- `key.to_bytes()` — converts `SigningKey` to its raw byte representation (32 bytes for
  secp256k1 private key)
- `serializer.serialize_bytes(...)` — hands those bytes to the serializer and returns
  `Result<S::Ok, S::Error>`, which is returned implicitly (no semicolon)

---

### `deserialize` — converting bytes back to the key

```rust
pub fn deserialize<'de, D>(
    deserializer: D,
) -> Result<super::SigningKey<super::Secp256k1>, D::Error>
where
    D: serde::Deserializer<'de>,
{
    let bytes: Vec<u8> = Vec::<u8>::deserialize(deserializer)?;
    Ok(super::SigningKey::from_slice(&bytes).unwrap())
}
```

**`'de` lifetime** — serde boilerplate. It ties the deserialized output's lifetime to the
input data. You don't need to think about it — every serde `deserialize` function has it.

**`D: serde::Deserializer<'de>`** — `D` is any serde deserializer, generic over format.

**`Result<SigningKey<Secp256k1>, D::Error>`** — returns either the reconstructed key
or a deserialization error.

**Body line by line:**

```rust
let bytes: Vec<u8> = Vec::<u8>::deserialize(deserializer)?;
```
- Deserializes the raw input as `Vec<u8>` (a byte array)
- `Vec::<u8>::` — turbofish syntax to tell the compiler which type to deserialize into
- `?` — if deserialization failed, return the error immediately from this function.
  Shorthand for the `if let Err(e) { return Err(e) }` pattern seen in `sha256.rs`

```rust
Ok(super::SigningKey::from_slice(&bytes).unwrap())
```
- `SigningKey::from_slice(&bytes)` — reconstructs the key from raw bytes. Returns a
  `Result` because the bytes might be invalid
- `.unwrap()` — panics if the bytes are invalid. Acceptable here because if we serialized
  the key correctly, the bytes should always be valid on the way back
- `Ok(...)` — wraps the result in `Ok` to match the `Result` return type

---

## The `?` operator

```rust
let bytes: Vec<u8> = Vec::<u8>::deserialize(deserializer)?;
```

`?` is shorthand for:
```rust
let bytes: Vec<u8> = match Vec::<u8>::deserialize(deserializer) {
    Ok(val) => val,
    Err(e) => return Err(e),
};
```

If the operation succeeds, `?` unwraps the `Ok` value. If it fails, it immediately returns
the error from the current function. Much cleaner than writing the match every time.

---

## Why not just `.unwrap()` everything?

`.unwrap()` panics on error — acceptable for "this should never happen" cases.
`?` propagates the error to the caller — required when the caller needs to handle or
display the error. Serde's deserialize functions must return `Result`, so `?` is used
to propagate errors up the chain rather than panicking.

---

## Summary

| Part | What it does |
|---|---|
| `#[serde(with = "signkey_serde")]` | tells serde to use custom serialize/deserialize for this field |
| `serialize` | converts `SigningKey` → raw bytes → serialized output |
| `deserialize` | serialized input → raw bytes → `SigningKey` |
| `super::` | accesses parent module types from inside the nested module |
| `'de` | lifetime boilerplate required by serde's deserialize signature |
| `?` | propagates errors up instead of panicking |
| `S: serde::Serializer` | makes the function generic — works with any output format |
