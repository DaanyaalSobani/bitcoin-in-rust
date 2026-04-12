# SHA-256 Hashing in rsbtc

Reference: `lib/src/sha256.rs`

## What `Hash::hash` does

Takes any serializable value and returns a `Hash` wrapping its SHA-256 digest.

```
any struct T (implements Serialize)
  → CBOR bytes  (via ciborium)
  → SHA-256 hex string  (via sha256::digest)
  → decode hex → [u8; 32]
  → wrap in Hash(U256)
```

---

## Step by step

### 1. Serialize to bytes

```rust
let mut serialized: Vec<u8> = vec![];
if let Err(e) = ciborium::into_writer(data, &mut serialized) {
    panic!("Failed to serialize data: {:?}. This should not happen", e);
}
```

`ciborium::into_writer` serializes `data` into CBOR format (compact binary, like binary JSON)
and writes the bytes into `serialized` through a `&mut` reference.

**Key ownership points:**
- `serialized` lives on the stack and is never moved
- `&mut serialized` (a mutable reference) is what gets passed — the function writes through it
- `data` is passed as `&T` (shared reference) so it is still available after the call
- `into_writer` actually takes `writer: W` by value, but `W = &mut Vec<u8>` — moving a
  `&mut` reference is fine, the underlying Vec stays put

**`if let Err(e) = ...`** — pattern match on the Result. Only enters the block on failure.
The happy path (`Ok`) carries no value so there is no else branch needed.

**`\` in the string literal** — line continuation, joins the two lines into one string
with no newline.

### 2. Hash the bytes

```rust
let hash = digest(&serialized);
```

`sha256::digest` has the signature:
```rust
pub fn digest<D: Sha256Digest>(input: D) -> String
```

`&Vec<u8>` satisfies `Sha256Digest` because the `sha256` crate implements the trait for it:
```rust
impl Sha256Digest for &Vec<u8> { ... }
```

You didn't write this — the crate author did. Rust looks up the impl at compile time and it
just works. Returns a hex string like `"deadbeef1234..."`.

**Why this works for composites of primitives:**

In most OOP languages you can only add methods to types you own. In Rust, any crate can
implement its own trait for any type — including composites like `&Vec<u8>` — as long as it
owns either the trait or the type (the orphan rule).

The `sha256` crate owns `Sha256Digest`, so it can implement it for whatever it likes.
You can see all the impls in the crate source at:

`~/.cargo/registry/src/index.crates.io-.../sha256-1.6.0/src/lib.rs`, lines 148–194

```rust
impl Sha256Digest for &[u8] {           // raw byte slice
    fn digest(self) -> String { __digest__(self) }
}
impl Sha256Digest for &Vec<u8> {        // reference to Vec — what we pass
    fn digest(self) -> String { __digest__(self) }
}
impl Sha256Digest for Vec<u8> {         // owned Vec
    fn digest(self) -> String { __digest__(&self) }
}
impl Sha256Digest for String {
    fn digest(self) -> String { __digest__(self.as_bytes()) }
}
impl Sha256Digest for &str {
    fn digest(self) -> String { __digest__(self.as_bytes()) }
}
impl Sha256Digest for char {
    fn digest(self) -> String { __digest__(self.encode_utf8(&mut [0; 4]).as_bytes()) }
}
impl Sha256Digest for &String {
    fn digest(self) -> String { __digest__(self.as_bytes()) }
}
```

Every impl funnels into `__digest__` which does the actual SHA-256 computation. The impls
just handle converting each type to `&[u8]` first. This is unlike Python duck typing —
the compiler verifies at compile time that your type satisfies the trait, no runtime surprise.


### 3. Convert hex string → `[u8; 32]`

```rust
let hash_bytes = hex::decode(hash).unwrap();
let hash_array: [u8; 32] = hash_bytes.as_slice().try_into().unwrap();
```

- `hex::decode` — turns the hex string back into raw bytes (`Vec<u8>`)
- `.as_slice()` — gets a `&[u8]` view of the Vec (no allocation)
- `.try_into()` — attempts to convert the slice into a fixed-size `[u8; 32]`. Returns
  `Err` if the length isn't exactly 32 (SHA-256 always produces 32 bytes so this is safe)
- `.unwrap()` — panics on error; acceptable here since SHA-256 output is always 32 bytes

### 4. Wrap in Hash

```rust
Hash(U256::from(hash_array))
```

Constructs the tuple struct. No `return` keyword — last expression is the return value.

---

## Display impl

```rust
impl fmt::Display for Hash {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:x}", self.0)
    }
}
```

- `fmt::Result` is `Result<(), fmt::Error>` — success or formatting error
- `write!(f, ...)` returns `fmt::Result` itself; since it's the last expression it is
  returned automatically (no semicolon)
- `{:x}` — lowercase hex format specifier
- `self.0` — the inner `U256` field of the tuple struct
- Once implemented, `println!("{}", some_hash)` prints the hash as a hex string

---

## Key concepts illustrated

| Concept | Where |
|---|---|
| `&mut` reference as out-parameter | `&mut serialized` in `into_writer` |
| Trait impl by an external crate | `Sha256Digest for &Vec<u8>` |
| `if let Err(e)` error handling | ciborium call |
| Tuple struct field access `.0` | `self.0`, `Hash(U256::from(...))` |
| Implicit return (no semicolon) | last line of `hash()` and `fmt()` |
| `{:x}` hex formatting | `Display` impl |
